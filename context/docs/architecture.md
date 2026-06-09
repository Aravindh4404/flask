# Flask Architecture

## Component Map

| Module | What it does |
|---|---|
| `app.py` | `Flask` class — WSGI entry point, request dispatch, error handling, app setup |
| `sansio/app.py` | `App` base — URL map, config, blueprints, error handler registry (no I/O) |
| `sansio/scaffold.py` | `Scaffold` — shared decorator registration (`route`, `before_request`, etc.) for both Flask and Blueprint |
| `sansio/blueprints.py` | `SansioBlueprint` + `BlueprintSetupState` — blueprint registration logic |
| `blueprints.py` | `Blueprint` — extends SansioBlueprint with CLI support |
| `ctx.py` | `AppContext` — context object holding request, session, `g`; pushed/popped per request |
| `globals.py` | `current_app`, `request`, `session`, `g` — `LocalProxy` objects backed by `ContextVar` |
| `wrappers.py` | `Request` / `Response` — Werkzeug subclasses with `url_rule`, `view_args`, JSON helpers |
| `sessions.py` | `SecureCookieSessionInterface` — HMAC-signed cookie sessions; pluggable via `SessionInterface` |
| `config.py` | `Config` dict + `ConfigAttribute` descriptor — loads from objects, files, env vars |
| `helpers.py` | `url_for`, `redirect`, `abort`, `flash`, `send_file`, `stream_with_context` |
| `templating.py` | `Environment` (Jinja2 subclass) + `DispatchingJinjaLoader` — searches app then blueprints |
| `signals.py` | Blinker signals: `request_started`, `request_finished`, `got_request_exception`, etc. |
| `cli.py` | `FlaskGroup`, `ScriptInfo`, `find_best_app` — the `flask` CLI and app discovery |
| `testing.py` | `FlaskClient`, `FlaskCliRunner` — test client with context preservation |
| `json/provider.py` | `JSONProvider` base + `DefaultJSONProvider` — pluggable JSON serialisation |
| `json/tag.py` | `TaggedJSONSerializer` — lossless round-trip for session values (dates, tuples, etc.) |
| `views.py` | `View` — class-based view with `as_view()` and `dispatch_request()` |
| `logging.py` | `create_logger()` — per-app logger with WSGI error stream handler |
| `debughelpers.py` | Debug-mode helpers for routing redirects and template load tracing |
| `typing.py` | `ResponseReturnValue` and other type aliases used across the codebase |
| `__init__.py` | Public API re-exports — everything a user imports from `flask` |
| `__main__.py` | Calls `cli.main()` so `python -m flask` works |

---

## Module Dependency Graph

```
                         ┌─────────────────┐
                         │   __init__.py   │  (re-exports public API)
                         └────────┬────────┘
                                  │ imports all of
                    ┌─────────────▼──────────────┐
                    │           app.py            │
                    │         (Flask class)       │
                    └──┬──────────┬──────────┬───┘
                       │          │          │
              ┌────────▼──┐  ┌────▼────┐  ┌─▼──────────┐
              │sansio/app │  │  ctx.py │  │ wrappers.py│
              │  (App)    │  │(Context)│  │(Req / Resp)│
              └────┬──────┘  └────┬────┘  └────────────┘
                   │              │
         ┌─────────▼──────┐  ┌───▼──────────────┐
         │sansio/scaffold │  │   globals.py      │
         │  (Scaffold)    │  │(LocalProxy / CVars│
         └────────────────┘  └──────────────────┘
                   ▲
         ┌─────────┴──────────┐
         │  blueprints.py     │
         │  (Blueprint)       │
         │  sansio/blueprints │
         └────────────────────┘

    Shared dependencies (imported by most modules):
    ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐
    │signals.py│  │helpers.py│  │sessions.py │  │config.py │
    └──────────┘  └──────────┘  └────────────┘  └──────────┘

    Leaf modules (no internal flask imports):
    ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────┐
    │typing.py │  │logging.py│  │json/tag.py   │  │views.py  │
    └──────────┘  └──────────┘  └──────────────┘  └──────────┘
```

**Key rule:** `sansio/` modules have no I/O and no WSGI knowledge — they hold pure logic. `app.py` and `ctx.py` add the WSGI and context machinery on top.

---

## Data Flow: Entry Point to Output

### 1. App creation

```python
app = Flask(__name__)
```

```
Flask.__init__
  └─ App.__init__  (sansio/app.py)
       ├─ creates url_map (Werkzeug Map)
       ├─ creates Config (config.py)
       ├─ initialises error_handler_spec, before/after_request_funcs dicts
       ├─ sets request_class = Request, response_class = Response
       ├─ sets session_interface = SecureCookieSessionInterface
       └─ registers /static route if static_folder exists
```

### 2. Request lifecycle (WSGI → response)

```
WSGI server
    │  environ dict + start_response callable
    ▼
Flask.__call__(environ, start_response)
    └─ delegates to Flask.wsgi_app()

Flask.wsgi_app(environ, start_response)
    ├─ [A] create AppContext  ──────────────────────────────────────────┐
    │       AppContext.from_environ(app, environ)                        │
    │         ├─ Request(environ)  →  wrappers.py                        │
    │         ├─ _AppCtxGlobals()  →  g object                           │
    │         └─ url_adapter = url_map.bind_to_environ(environ)          │
    │                                                                    │
    ├─ [B] ctx.push()                                                    │
    │       ├─ _cv_app.set(ctx)  (ContextVar — thread-safe)              │
    │       ├─ appcontext_pushed signal                                  │
    │       ├─ session_interface.open_session(app, request)              │
    │       └─ url_adapter.match(return_rule=True)                       │
    │             ├─ sets request.url_rule, request.view_args            │
    │             └─ on failure: request.routing_exception = HTTPError   │
    │                                                                    │
    ├─ [C] Flask.full_dispatch_request()                                 │
    │       ├─ request_started.send(app)                                 │
    │       ├─ preprocess_request()                                      │
    │       │     runs before_request_funcs[None] + [blueprint_name]     │
    │       │     first non-None return short-circuits to finalize        │
    │       ├─ dispatch_request()                                        │
    │       │     ├─ if routing_exception → raise it (404/405/…)         │
    │       │     ├─ if OPTIONS → auto OPTIONS response                  │
    │       │     └─ view_functions[endpoint](**view_args)               │
    │       │           view returns str / dict / tuple / Response        │
    │       └─ finalize_request(rv)                                      │
    │             ├─ make_response(rv)                                   │
    │             │     str → Response(rv)                               │
    │             │     dict → jsonify(rv)                               │
    │             │     tuple → parse (body, status, headers)            │
    │             │     Response → pass through                          │
    │             ├─ process_response()                                  │
    │             │     runs after_request_funcs[blueprint] + [None]     │
    │             │     each function receives and must return response   │
    │             ├─ session_interface.save_session(app, session, resp)  │
    │             └─ request_finished.send(app, response=response)       │
    │                                                                    │
    ├─ [D] return response(environ, start_response)                      │
    │       Werkzeug Response.__call__ writes status, headers, body      │
    │                                                                    │
    └─ [E] ctx.pop()  (finally block)                          ◄─────────┘
            ├─ teardown_request_funcs[blueprint] + [None]
            ├─ request.close()  (temp file cleanup)
            ├─ teardown_appcontext handlers
            ├─ _cv_app.reset()
            └─ appcontext_popped signal
```

### 3. Exception path

```
Exception during dispatch
    └─ handle_user_exception(e)
         ├─ if HTTPException and not trap_http_exception
         │     └─ handle_http_exception(e)
         │           _find_error_handler by code, then by MRO
         ├─ else: _find_error_handler by exception MRO
         │         searches blueprint handlers first, then app-level
         ├─ if handler found → call handler(e) → finalize_request
         └─ else → handle_exception(e)
               ├─ got_request_exception.send(app, exception=e)
               ├─ if PROPAGATE_EXCEPTIONS or DEBUG → reraise
               └─ return 500 InternalServerError response
```

---

## Blueprint System

Blueprints delay all registration as `deferred_functions` and replay them on `app.register_blueprint()`.

```
Blueprint('api', __name__, url_prefix='/api')
    │
    │  @bp.route('/users')           → stored in deferred_functions
    │  @bp.before_request(fn)        → stored in deferred_functions
    │  @bp.errorhandler(404)(fn)     → stored in deferred_functions
    │
app.register_blueprint(bp)
    └─ BlueprintSetupState(bp, app, options)
         └─ for fn in bp.deferred_functions: fn(state)
               ├─ add_url_rule('/api/users', endpoint='api.get_users', view_func=…)
               │     → app.view_functions['api.get_users'] = fn
               │     → app.url_map.add(Rule('/api/users', endpoint='api.get_users'))
               ├─ app.before_request_funcs['api'].append(fn)
               └─ app.error_handler_spec['api'][404] = fn

Request to /api/users:
    url_map.match() → endpoint='api.get_users'
    preprocess_request runs:
        before_request_funcs[None]   (app-level)
        before_request_funcs['api']  (blueprint-level)
    view_functions['api.get_users'](**view_args)
    process_response runs:
        after_request_funcs['api']   (blueprint-level)
        after_request_funcs[None]    (app-level)
```

---

## Context System

The context system is what makes `request`, `current_app`, `g`, and `session` available as module-level globals without passing them as arguments.

```
ContextVar[AppContext]  (_cv_app in globals.py)
    │
    ├─ current_app  =  LocalProxy(lambda: _cv_app.get().app)
    ├─ request      =  LocalProxy(lambda: _cv_app.get().request)
    ├─ session      =  LocalProxy(lambda: _cv_app.get().session)
    └─ g            =  LocalProxy(lambda: _cv_app.get().g)

One AppContext per request (or manually with `app.app_context()`).
Each thread / async task gets its own ContextVar slot — no sharing.
```

---

## Extension Points

| Interface | How to plug in | Where registered |
|---|---|---|
| Custom session backend | Subclass `SessionInterface`, assign to `app.session_interface` | `sessions.py` |
| Custom JSON serialiser | Subclass `JSONProvider`, assign to `app.json` | `json/provider.py` |
| Request hooks | `@app.before_request`, `@app.after_request`, `@app.teardown_request` | `sansio/scaffold.py` |
| App context teardown | `@app.teardown_appcontext` | `sansio/scaffold.py` |
| Error handlers | `@app.errorhandler(code_or_exc)` | `sansio/scaffold.py` |
| Template context | `@app.context_processor` | `sansio/scaffold.py` |
| Template filters/globals | `@app.template_filter`, `@app.template_global` | `sansio/scaffold.py` |
| Blinker signals | `signal.connect(fn, app)` | `signals.py` |
| CLI commands | `@app.cli.command()` or `@bp.cli.command()` | `cli.py` |
| Class-based views | `View.as_view()` passed to `add_url_rule` | `views.py` |

---

## What Depends on What (module-level)

```
app.py
  ├── sansio/app.py
  │     └── sansio/scaffold.py
  ├── ctx.py
  │     ├── globals.py
  │     └── signals.py
  ├── wrappers.py          (Werkzeug Request/Response + flask additions)
  ├── sessions.py
  ├── helpers.py
  │     └── globals.py
  ├── templating.py
  │     └── globals.py
  ├── signals.py           (Blinker)
  ├── logging.py
  └── cli.py
        └── globals.py

blueprints.py
  └── sansio/blueprints.py
        └── sansio/scaffold.py

globals.py                 (no internal flask imports — only Werkzeug LocalProxy + ContextVar)
signals.py                 (no internal flask imports — only Blinker)
typing.py                  (no internal flask imports — stdlib only)
json/tag.py                (no internal flask imports)
```

Werkzeug is the only significant external dependency woven throughout: `Map`/`Rule` for routing, `Request`/`Response` base classes, `LocalProxy`, `HTTPException` hierarchy, and the built-in development server.
