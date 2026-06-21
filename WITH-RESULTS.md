# Flask Codebase — 40 Questions Answered

Generated: 2026-06-16  
Source: `C:\Users\aravi\Desktop\flask` (Flask 3.x)  
Context layers used: GBrain (`gstack-code-flask-7d20071a`), SocratiCode, direct file reads.

---

## SECTION A — CODE SYMBOL & PRECISION (Q1–Q13)

---

### Q1. Where is `wsgi_app` defined? Walk through every operation it performs in order.

**Definition:** `src/flask/app.py:1566`, method of class `Flask`.

**Operations in order:**

1. **Create request context** (line 1592): `ctx = self.request_context(environ)` — builds an `AppContext` from the WSGI environ.
2. **Initialize error tracker** (line 1593): `error: BaseException | None = None`.
3. **Push the context** (line 1596): `ctx.push()` — registers the context as active in the `_cv_app` ContextVar, fires the `appcontext_pushed` signal, lazily opens the session, and matches the URL.
4. **Dispatch the request** (line 1597): `response = self.full_dispatch_request(ctx)` — runs before-request hooks, the view function, and after-request hooks to produce a `Response`.
5. **Catch any `Exception`** (lines 1598–1600): If any exception is raised, records it in `error` and calls `self.handle_exception(ctx, e)` to produce a 500 response.
6. **Catch bare `BaseException`** (lines 1601–1603): Captures into `error` and re-raises (for `KeyboardInterrupt`, `SystemExit`, etc.).
7. **Call the response as WSGI** (line 1604): `return response(environ, start_response)` — sends headers and body to the WSGI server.
8. **`finally` block — debugger hook** (lines 1606–1607): If `werkzeug.debug.preserve_context` is in `environ`, invokes that callback with `ctx` (supports the interactive debugger and test client context preservation).
9. **`finally` block — suppress error conditionally** (lines 1609–1614): If `should_ignore_error` is set and returns `True` for the error, resets `error` to `None`.
10. **`finally` block — pop the context** (line 1616): `ctx.pop(error)` — runs teardown functions and resets the ContextVar.

---

### Q2. Where is `full_dispatch_request` defined? List every function call inside it in order.

**Definition:** `src/flask/app.py:992`, method of class `Flask`.

**Function calls in order:**

1. `warnings.warn(...)` (line 1002) — deprecation warning emitted only if `should_ignore_error` is set and `_got_first_request` is False.
2. `self._got_first_request = True` (line 1010) — state mutation marking first request served.
3. `request_started.send(self, _async_wrapper=self.ensure_sync)` (line 1013) — fires the `request_started` Blinker signal.
4. `self.preprocess_request(ctx)` (line 1014) — runs URL value preprocessors and `before_request` handlers; if any returns a value, that becomes `rv` and dispatch is skipped.
5. `self.dispatch_request(ctx)` (line 1016) — called only if `preprocess_request` returned `None`; matches the URL rule and calls the view function.
6. `self.handle_user_exception(ctx, e)` (line 1018) — called in the `except Exception` branch; handles HTTP exceptions and registered error handlers.
7. `self.finalize_request(ctx, rv)` (line 1019) — converts `rv` to a `Response`, runs `after_request` hooks, saves the session, and sends `request_finished` signal.

---

### Q3. Where is `has_request_context` defined? What exactly does it check and what does it return?

**Definition:** `src/flask/ctx.py:209`, module-level function.

**What it checks:** Single expression:
```python
return (ctx := _cv_app.get(None)) is not None and ctx.has_request
```
It performs two checks via a walrus-operator expression:
1. `_cv_app.get(None)` — retrieves the current value of the `_cv_app` ContextVar (`src/flask/globals.py:40`). Returns `None` if no app context is active.
2. `ctx.has_request` — a `bool` property on `AppContext` (`ctx.py:351–353`) that returns `True` if `self._request is not None`, i.e., the context was created with request data.

**Returns:** `True` if and only if both: (a) an `AppContext` is currently pushed on the current thread/task, **and** (b) that context carries a request object. Returns `False` in all other cases (no context at all, or a pure app context from `app.app_context()`).

---

### Q4. Where is `AppContext` defined? What does pushing it do to the application state?

**Definition:** `src/flask/ctx.py:260`, class `AppContext`.

**What `push()` does (`ctx.py:416–444`) in order:**

1. **Increment push counter** (line 428): `self._push_count += 1`. If already pushed, returns immediately (no-op for nested pushes).
2. **Set the ContextVar** (line 433): `self._cv_token = _cv_app.set(self)` — installs this `AppContext` as the active context for the current Python ContextVar context (thread/async-task scoped). All accesses to `current_app`, `g`, `request`, `session` via `LocalProxy` now resolve through this context.
3. **Fire `appcontext_pushed` signal** (line 434): `appcontext_pushed.send(self.app, _async_wrapper=self.app.ensure_sync)`.
4. **Lazily open the session** (lines 436–439): If this is a request context (`self._request is not None`), calls `self._get_session()`, which calls `session_interface.open_session(app, request)`. If that returns `None`, falls back to `make_null_session`.
5. **Match the request URL** (lines 441–444): If a `url_adapter` exists, calls `self.match_request()` which invokes `url_adapter.match(return_rule=True)` and stores `url_rule` and `view_args` on the request object.

**`AppContext.__init__` (`ctx.py:300`) sets up:**
- `self.app` — reference to the Flask app
- `self.g` — a fresh `_AppCtxGlobals()` instance (the `g` object)
- `self.url_adapter` — a `MapAdapter` bound to the request environ (or app-level if no request)
- `self._request`, `self._session`, `self._flashes` — request/session state, lazily populated

---

### Q5. Where is `LocalProxy` defined? What does it proxy and how does it resolve the underlying object?

**Definition:** `werkzeug/src/werkzeug/local.py:738` — it is **not** in the Flask codebase. It is in the Werkzeug package, imported into Flask at `src/flask/globals.py:6`.

**What it proxies:** Flask uses `LocalProxy` to create five module-level proxy objects in `src/flask/globals.py`:
- `app_ctx` → the active `AppContext`
- `current_app` → `ctx.app` (the Flask app)
- `g` → `ctx.g` (per-request global data)
- `request` → `ctx.request`
- `session` → `ctx.session`

Each proxy is backed by the `_cv_app` ContextVar with an optional attribute name argument:
```python
current_app = LocalProxy(_cv_app, "app", unbound_message=...)
request     = LocalProxy(_cv_app, "request", unbound_message=...)
session     = LocalProxy(_cv_app, "session", unbound_message=...)
g           = LocalProxy(_cv_app, "g", unbound_message=...)
```

**How it resolves the underlying object:**

`LocalProxy.__init__` stores the lookup source in `__wrapped` (in `__slots__`). It creates a `_get_current_object` closure at init time. For a `ContextVar` argument:
```python
def _get_current_object() -> T:
    try:
        obj = local.get()        # ContextVar.get() — raises LookupError if unset
    except LookupError:
        raise RuntimeError(unbound_message) from None
    return get_name(obj)         # getattr(obj, attr_name) if attr_name provided
```
All attribute/item accesses on the proxy (via `__getattr__`, `__getitem__`, operators, etc.) are forwarded through `_ProxyLookup` descriptors that call `_get_current_object()` at access time, making the proxy fully transparent. `__wrapped__` returns the original ContextVar for introspection.

---

### Q6. Where is `open_session` defined? What does it return and what happens if it returns `None`?

**Two definitions:**

1. **Base class stub** — `src/flask/sessions.py:249`, `SessionInterface.open_session()`: Raises `NotImplementedError`. Intended to be overridden.

2. **Concrete implementation** — `src/flask/sessions.py:323`, `SecureCookieSessionInterface.open_session()`:
   - Calls `self.get_signing_serializer(app)` — returns `None` if `app.secret_key` is not set.
   - If serializer is `None`, returns `None` immediately (secret key missing).
   - If no cookie named `SESSION_COOKIE_NAME` in the request: returns `self.session_class()` — an empty `SecureCookieSession`.
   - If a cookie exists: calls `s.loads(val, max_age=max_age)` to verify the HMAC signature and deserialise. Returns `self.session_class(data)` on success, or `self.session_class()` (empty) on `BadSignature`.

**What happens if it returns `None`:** In `AppContext._get_session()` (`ctx.py:381–393`):
```python
self._session = si.open_session(self.app, self.request)
if self._session is None:
    self._session = si.make_null_session(self.app)
```
A null session (`NullSession`) is created. Accessing it raises a `RuntimeError` with a helpful message indicating the secret key is not set, rather than silently losing data.

---

### Q7. Where is `save_session` defined? What arguments does it take and what does it write?

**Two definitions:**

1. **Base class stub** — `src/flask/sessions.py:263`, `SessionInterface.save_session(self, app, session, response)`: Raises `NotImplementedError`.

2. **Concrete implementation** — `src/flask/sessions.py:337`, `SecureCookieSessionInterface.save_session(self, app: Flask, session: SessionMixin, response: Response) -> None`.

**Arguments:** `app` (the Flask application), `session` (the `SessionMixin` instance), `response` (the `Response` object to write the cookie into).

**Logic flow:**

1. Reads cookie parameters from the app config: `name`, `domain`, `path`, `secure`, `partitioned`, `samesite`, `httponly`.
2. If `session.accessed` is `True`, adds `Vary: Cookie` to the response headers.
3. **Empty + modified session** (lines 354–367): If the session is empty and modified, calls `response.delete_cookie(name, ...)` to remove the cookie, and adds `Vary: Cookie`.
4. **Empty + unmodified session**: returns without writing anything.
5. **`should_set_cookie` check** (line 369): If `SESSION_REFRESH_EACH_REQUEST` is False and the session is not modified, returns without writing.
6. **Write the cookie** (lines 372–385): Calls `get_signing_serializer(app).dumps(dict(session))` to serialise and HMAC-sign the session data, then calls `response.set_cookie(name, val, expires=..., httponly=..., domain=..., path=..., secure=..., partitioned=..., samesite=...)` which writes the `Set-Cookie` header on the response. Adds `Vary: Cookie`.

---

### Q8. Where is `handle_exception` defined? What is the full flow inside it?

**Definition:** `src/flask/app.py:897`, method of class `Flask`.

**Full flow in order:**

1. **Capture exc_info** (line 925): `exc_info = sys.exc_info()` — captures the active exception's traceback.
2. **Send `got_request_exception` signal** (line 926): `got_request_exception.send(self, _async_wrapper=self.ensure_sync, exception=e)` — always fired regardless of propagation.
3. **Read `PROPAGATE_EXCEPTIONS`** (line 927): `propagate = self.config["PROPAGATE_EXCEPTIONS"]`.
4. **Auto-set propagate** (lines 929–930): If `propagate is None`, set it to `self.testing or self.debug`.
5. **Propagate branch** (lines 932–938): If `propagate` is True — if the exception is the currently active one: bare `raise` to preserve traceback; otherwise `raise e`.
6. **Log the exception** (line 940): `self.log_exception(ctx, exc_info)` — logs at `ERROR` level via `app.logger`.
7. **Wrap as `InternalServerError`** (line 942): `server_error = InternalServerError(original_exception=e)` — always 500, with the original exception accessible as `e.original_exception`.
8. **Find an error handler** (line 943): `handler = self._find_error_handler(server_error, ctx.request.blueprints)` — searches blueprint and app handlers for `InternalServerError`/500.
9. **Call the handler if found** (lines 945–946): `server_error = self.ensure_sync(handler)(server_error)`.
10. **Finalize** (line 948): `return self.finalize_request(ctx, server_error, from_error_handler=True)` — runs `make_response`, then `process_response`. The `from_error_handler=True` flag causes exceptions in finalization to be logged and swallowed rather than re-raised.

---

### Q9. Where is `ensure_sync` defined? What problem does it solve and what Python feature does it use?

**Definition:** `src/flask/app.py:1065`, method of class `Flask`.

**The problem it solves:** Flask is a synchronous WSGI application. When a user defines an `async def` view function or hook, calling it directly returns a coroutine object — the coroutine would never be awaited. `ensure_sync` bridges the gap: it detects `async def` functions and wraps them so they can be called synchronously from WSGI code.

**Python feature used:** `inspect.iscoroutinefunction(func)` — checks whether `func` was declared with `async def`. If it was, `ensure_sync` delegates to `self.async_to_sync(func)` (`app.py:1079–1100`), which by default imports `asgiref.sync.async_to_sync` to run the coroutine in an event loop and block until it completes. Plain `def` functions are returned unchanged.

```python
def ensure_sync(self, func):
    if iscoroutinefunction(func):
        return self.async_to_sync(func)
    return func
```

`ensure_sync` is called before every view function, `before_request` handler, `after_request` handler, template context processor, error handler, and signal sender throughout the dispatch pipeline.

---

### Q10. Where is `make_response` defined? What types can it accept and how does it handle each?

**Definition:** `src/flask/app.py:1224`, method of class `Flask`.  
Also: `src/flask/helpers.py:151`, standalone helper function (calls `current_app.make_response()`).

**Input types and handling:**

| Input type | Handling |
|---|---|
| `tuple` (3-element) | Unpacked as `(body, status, headers)` |
| `tuple` (2-element, 2nd is `Headers`/`dict`/`list`/`tuple`) | Unpacked as `(body, headers)` |
| `tuple` (2-element, 2nd is `int`/`str`) | Unpacked as `(body, status)` |
| `tuple` (other size) | Raises `TypeError` |
| `None` | Raises `TypeError` ("did not return a valid response") |
| `str` / `bytes` / `bytearray` | `self.response_class(rv, status=status, headers=headers)` — creates Response with string/bytes body |
| `cabc.Iterator` | Same as bytes/str — streaming response |
| `dict` or `list` | `self.json.response(rv)` — JSON-serialised via the app's JSON provider |
| `BaseResponse` (Werkzeug Response, not the app's subclass) | `self.response_class.force_type(rv, request.environ)` — coerces to the app's response class |
| callable (WSGI app) | `self.response_class.force_type(rv, request.environ)` — evaluates the WSGI callable |
| `self.response_class` instance | Returned unchanged |
| Anything else | Raises `TypeError` |

After normalisation, if `status` is set, it updates `rv.status` (str) or `rv.status_code` (int). If `headers` is set, `rv.headers.update(headers)` extends the response headers.

---

### Q11. Trace from `Flask.__call__` to where the response is finalized. Every function with file and line.

| Step | Function | File & Line |
|---|---|---|
| 1 | `Flask.__call__(environ, start_response)` | `src/flask/app.py:1618` |
| 2 | `Flask.wsgi_app(environ, start_response)` | `src/flask/app.py:1566` |
| 3 | `Flask.request_context(environ)` → returns `AppContext` | `src/flask/app.py:1501` (called from `wsgi_app:1592`) |
| 4 | `AppContext.push()` | `src/flask/ctx.py:416` |
| 5 | `AppContext._get_session()` (inside push) | `src/flask/ctx.py:381` |
| 6 | `AppContext.match_request()` (inside push) | `src/flask/ctx.py:405` |
| 7 | `Flask.full_dispatch_request(ctx)` | `src/flask/app.py:992` |
| 8 | `request_started.send(...)` | `src/flask/app.py:1013` |
| 9 | `Flask.preprocess_request(ctx)` | `src/flask/app.py:1366` |
| 10 | `Flask.dispatch_request(ctx)` | `src/flask/app.py:966` |
| 11 | `Flask.ensure_sync(view_func)(**view_args)` | `src/flask/app.py:990` |
| 12 | `Flask.finalize_request(ctx, rv)` | `src/flask/app.py:1021` |
| 13 | `Flask.make_response(rv)` | `src/flask/app.py:1224` |
| 14 | `Flask.process_response(ctx, response)` | `src/flask/app.py:1394` |
| 15 | `SessionInterface.save_session(app, session, response)` | `src/flask/sessions.py:337` (called inside `process_response:1416`) |
| 16 | `request_finished.send(...)` | `src/flask/app.py:1042` |
| 17 | `response(environ, start_response)` | `src/flask/app.py:1604` (back in `wsgi_app`) |
| 18 | `AppContext.pop(error)` | `src/flask/ctx.py:446` (in `finally`) |

If an exception occurs at step 10/11, `handle_user_exception` or `handle_exception` is called instead, both of which ultimately call `finalize_request(..., from_error_handler=True)`.

---

### Q12. What is the exact chain of functions called when a session is saved at the end of a request?

1. **`Flask.finalize_request(ctx, rv)`** — `src/flask/app.py:1021`: Calls `make_response(rv)` then calls `process_response`.
2. **`Flask.process_response(ctx, response)`** — `src/flask/app.py:1394`: Runs `after_request` handlers first, then on lines 1415–1416:
   ```python
   if not self.session_interface.is_null_session(ctx._get_session()):
       self.session_interface.save_session(self, ctx._get_session(), response)
   ```
   `is_null_session()` checks if the session is a `NullSession`. If not null, proceeds.
3. **`ctx._get_session()`** — `src/flask/ctx.py:381`: Returns the already-opened `SecureCookieSession`.
4. **`SecureCookieSessionInterface.save_session(app, session, response)`** — `src/flask/sessions.py:337`: Reads all cookie attributes from config.
5. **`self.get_signing_serializer(app)`** — `src/flask/sessions.py:303`: Returns a `URLSafeTimedSerializer` configured with `app.secret_key`, `salt="cookie-session"`, HMAC key derivation, SHA-1 digest.
6. **`serializer.dumps(dict(session))`** — Serialises the session dict using `TaggedJSONSerializer` then signs with HMAC-SHA1, producing a URL-safe base64 string.
7. **`response.set_cookie(name, val, expires=..., httponly=..., ...)`** — Werkzeug `Response.set_cookie()`. Writes the `Set-Cookie` header into `response.headers`.
8. **`response.vary.add("Cookie")`** — Appends `Vary: Cookie` to the response headers.
9. Back in `wsgi_app`, **`response(environ, start_response)`** — Werkzeug `Response.__call__` calls `start_response(status, headers)` with all headers (including `Set-Cookie`) and returns the body iterable, sending the cookie to the browser.

---

### Q13. Which methods of `Flask` are inherited from `App` vs added directly?

#### Methods defined directly in `Flask` (`src/flask/app.py:109–1625`)

| Method | Line | Notes |
|---|---|---|
| `__init_subclass__` | 254 | Compatibility shim for old ctx-less signatures |
| `__init__` | 310 | Overrides `App.__init__`; adds CLI, static route |
| `get_send_file_max_age` | 365 | Reads `SEND_FILE_MAX_AGE_DEFAULT` config |
| `send_static_file` | 392 | Serves static files via `send_from_directory` |
| `open_resource` | 414 | Opens files relative to `root_path` |
| `open_instance_resource` | 447 | Opens files relative to `instance_path` |
| `create_jinja_environment` | 469 | Overrides `App` abstract method; builds full Jinja env |
| `create_url_adapter` | 509 | Overrides `App` stub; binds URL map to environ |
| `raise_routing_exception` | 562 | Overrides `App` method |
| `update_template_context` | 590 | Injects request/session/g into template context |
| `make_shell_context` | 620 | Returns `{"app": self, "g": g}` + shell processors |
| `run` | 632 | Starts dev server via `werkzeug.serving.run_simple` |
| `test_client` | ~795 | Creates a `FlaskClient` |
| `test_cli_runner` | 813 | Creates a `FlaskCliRunner` |
| `handle_http_exception` | 830 | Finds registered error handler or returns exc |
| `handle_user_exception` | 865 | Routes to HTTP/user error handler |
| `handle_exception` | 897 | 500 handler with propagation logic |
| `log_exception` | 950 | Logs exception at ERROR level |
| `dispatch_request` | 966 | Matches URL rule, calls view function |
| `full_dispatch_request` | 992 | Pre/post processing + dispatch |
| `finalize_request` | 1021 | Converts rv → Response, runs process_response |
| `make_default_options_response` | 1053 | Builds auto OPTIONS response |
| `ensure_sync` | 1065 | Wraps async views for WSGI |
| `async_to_sync` | 1079 | Uses asgiref to run coroutines |
| `url_for` | 1102 | Generates URLs |
| `make_response` | 1224 | Converts view return value → Response |
| `preprocess_request` | 1366 | Runs url_value_preprocessors + before_request hooks |
| `process_response` | 1394 | Runs after_request hooks + save_session |
| `do_teardown_request` | 1420 | Calls teardown_request callbacks |
| `do_teardown_appcontext` | ~1460 | Calls teardown_appcontext callbacks |
| `app_context` | ~1490 | Returns `AppContext(self)` with no request |
| `request_context` | ~1510 | Returns `AppContext.from_environ(self, environ)` |
| `wsgi_app` | 1566 | The actual WSGI entry point |
| `__call__` | 1618 | Delegates to `wsgi_app` |

#### Methods inherited from `App` (`src/flask/sansio/app.py:59–1013`) — not overridden in `Flask`

| Method | Location in `App` |
|---|---|
| `_check_setup_finished` | `sansio/app.py:410` |
| `name` (cached_property) | `sansio/app.py:422` |
| `logger` (cached_property) | `sansio/app.py:439` |
| `jinja_env` (cached_property) | `sansio/app.py:466` |
| `make_config` | `sansio/app.py:479` |
| `make_aborter` | `sansio/app.py:495` |
| `auto_find_instance_path` | `sansio/app.py:507` |
| `create_global_jinja_loader` | `sansio/app.py:520` |
| `select_jinja_autoescape` | `sansio/app.py:533` |
| `debug` (property + setter) | `sansio/app.py:549` |
| `register_blueprint` | `sansio/app.py:569` |
| `iter_blueprints` | `sansio/app.py:597` |
| `add_url_rule` | `sansio/app.py:604` |
| `template_filter` / `add_template_filter` | `sansio/app.py:663` |
| `template_test` / `add_template_test` | `sansio/app.py:719` |
| `template_global` / `add_template_global` | `sansio/app.py:778` |
| `teardown_appcontext` | `sansio/app.py:826` |
| `shell_context_processor` | `sansio/app.py:857` |
| `_find_error_handler` | `sansio/app.py:868` |
| `trap_http_exception` | `sansio/app.py:893` |
| `redirect` | `sansio/app.py:939` |
| `inject_url_defaults` | `sansio/app.py:960` |
| `handle_url_build_error` | `sansio/app.py:981` |
| All `Scaffold` methods (`route`, `before_request`, `after_request`, `errorhandler`, `teardown_request`, etc.) | `sansio/scaffold.py` |

---

## SECTION B — CONCEPTUAL / ARCHITECTURAL (Q14–Q25)

---

### Q14. How does the Flask request lifecycle work? Every function in the chain with file.

#### Phase 1: WSGI Entry

**`Flask.__call__`** (`src/flask/app.py:1618`) — The WSGI server calls the application object. Its only job is to delegate to `wsgi_app`.

#### Phase 2: Context Setup and Teardown Shell

**`Flask.wsgi_app`** (`src/flask/app.py:1566`) — Wraps the entire request in a try/finally guaranteeing `ctx.pop()` is always called.

1. **`Flask.request_context`** (`src/flask/app.py:1501`) — Calls `AppContext.from_environ(self, environ)`, constructing a new `AppContext`.
2. **`AppContext.from_environ`** (`src/flask/ctx.py:340`) — Creates the `Request` via `app.request_class(environ)`, attaches `app.json`, returns `AppContext(app, request=request)`.
3. **`AppContext.__init__`** (`src/flask/ctx.py:300`) — Initialises `.g`, `.url_adapter`, `._request`, `._session=None`, `._cv_token=None`. Calls `app.create_url_adapter(request)`.
4. **`Flask.create_url_adapter`** (`src/flask/app.py:509`) — Calls `url_map.bind_to_environ(request.environ, ...)` to create a `werkzeug.routing.MapAdapter`.
5. **`ctx.push()`** (`src/flask/ctx.py:416`) — Calls `_cv_app.set(self)`, fires `appcontext_pushed` signal, then calls `self._get_session()` and `self.match_request()`.
6. **`AppContext._get_session`** (`src/flask/ctx.py:381`) — Calls `app.session_interface.open_session(app, request)`.
7. **`AppContext.match_request`** (`src/flask/ctx.py:405`) — Calls `url_adapter.match(return_rule=True)`, storing matched `Rule` and `view_args` on the request object.

#### Phase 3: Dispatch

8. **`Flask.full_dispatch_request`** (`src/flask/app.py:992`) — Fires `request_started` signal, then calls `preprocess_request`, then `dispatch_request`. On any exception, calls `handle_user_exception`.
9. **`Flask.preprocess_request`** (`src/flask/app.py:1363`) — Iterates `url_value_preprocessors` then `before_request_funcs`.
10. **`Flask.dispatch_request`** (`src/flask/app.py:966`) — Looks up `self.view_functions[endpoint]`, wraps it in `ensure_sync`, calls it with `**view_args`.
11. **`Flask.ensure_sync`** (`src/flask/app.py:1065`) — Returns `async_to_sync(func)` for coroutine functions (via `asgiref`), or returns sync functions unchanged.

#### Phase 4: Response Finalization

12. **`Flask.finalize_request`** (`src/flask/app.py:1021`) — Calls `make_response(rv)`, then `process_response(ctx, response)`, then fires `request_finished` signal.
13. **`Flask.make_response`** (`src/flask/app.py:1224`) — Normalises the view return value into a `Response` instance.
14. **`Flask.process_response`** (`src/flask/app.py:1394`) — Runs `after_request_funcs` (blueprint order reversed, then app-level), then calls `session_interface.save_session(app, session, response)`.

#### Phase 5: Teardown (always runs in `finally`)

15. **`ctx.pop(error)`** (`src/flask/ctx.py:446`) — Calls `app.do_teardown_request(ctx, exc)`, `request.close()`, `app.do_teardown_appcontext(ctx, exc)`, then `_cv_app.reset(token)`.
16. **`Flask.do_teardown_request`** (`src/flask/app.py:1420`) — Runs all `teardown_request_funcs`, fires `request_tearing_down` signal.
17. **`Flask.do_teardown_appcontext`** (`src/flask/app.py:1453`) — Runs all `teardown_appcontext_funcs`, fires `appcontext_tearing_down` signal.

#### Exception sub-path

- **`Flask.handle_user_exception`** (`src/flask/app.py:865`) — Called from within `full_dispatch_request`. For `HTTPException` delegates to `handle_http_exception`. For others, searches `_find_error_handler`; re-raises if no handler.
- **`Flask.handle_http_exception`** (`src/flask/app.py:830`) — Skips `RoutingException`; calls `_find_error_handler(e, blueprints)`.
- **`Flask.handle_exception`** (`src/flask/app.py:897`) — Last resort; fires `got_request_exception`, logs, wraps in 500, finds handler, calls `finalize_request(..., from_error_handler=True)`.

---

### Q15. Application context vs request context — lifecycle, push/pop timing, and access outside context.

#### Flask 3.x Unified Model

In Flask 3.x (specifically 3.2+), there is a **single class `AppContext`** (`src/flask/ctx.py:260`) that represents both concepts. The old `RequestContext` is now an alias for `AppContext` and will be removed in Flask 4.0. The distinction is tracked by the boolean property `has_request` (`ctx.py:351`), which is `True` when the context was constructed with a `request` argument.

#### AppContext (Application Context)

Pushed for every request **and** every CLI command. Provides:
- `current_app` — the Flask application instance
- `g` — per-context scratch space (`_AppCtxGlobals`, `ctx.py:30`)

**Pushed:** `ctx.push()` (`ctx.py:416`) is called in `wsgi_app` before dispatch (`app.py:1596`), or when you enter `with app.app_context()`.

**Popped:** `ctx.pop()` (`ctx.py:446`) is called in the `finally` block of `wsgi_app` (`app.py:1616`), or on `with` block exit.

Inside `push()`:
```python
self._cv_token = _cv_app.set(self)    # ctx.py:433
appcontext_pushed.send(self.app)      # ctx.py:434
```
Inside `pop()`:
```python
app.do_teardown_request(ctx, exc)     # ctx.py:490
app.do_teardown_appcontext(ctx, exc)  # ctx.py:496
_cv_app.reset(self._cv_token)         # ctx.py:498
```

#### Request Context

The same `AppContext` object, constructed via `AppContext.from_environ(app, environ)` (`ctx.py:340`), so `._request` is not `None` and `has_request` returns `True`. Created by `Flask.request_context(environ)` (`app.py:1501`) and `Flask.test_request_context(...)` (`app.py:1517`).

When `has_request` is True, the context additionally provides `request`, `session`, and URL matching.

#### Access Outside a Context

`request` is defined as:
```python
# src/flask/globals.py:57-59
request: RequestProxy = LocalProxy(
    _cv_app, "request", unbound_message=_no_req_msg
)
```
If `_cv_app.get()` returns nothing (no context pushed), `LocalProxy` raises:
```
RuntimeError: Working outside of request context.
Attempted to use functionality that expected an active HTTP request.
```
`current_app` and `g` raise a `RuntimeError` with: `Working outside of application context.`  
Error messages are defined in `src/flask/globals.py:33–56`.

---

### Q16. How does Flask use `LocalProxy` and `ContextVar` for thread-safe globals?

#### The Single Source of Truth

```python
# src/flask/globals.py:40
_cv_app: ContextVar[AppContext] = ContextVar("flask.app_ctx")
```

This is a `contextvars.ContextVar` from Python's stdlib. Each OS thread — and each `asyncio` Task — has an independent slot. Two concurrent requests in two threads each see a completely different `AppContext` value for the same `_cv_app` variable.

#### Proxy Definitions

```python
# src/flask/globals.py:41-62
app_ctx     = LocalProxy(_cv_app, unbound_message=_no_app_msg)
current_app = LocalProxy(_cv_app, "app", unbound_message=_no_app_msg)
g           = LocalProxy(_cv_app, "g", unbound_message=_no_app_msg)
request     = LocalProxy(_cv_app, "request", unbound_message=_no_req_msg)
session     = LocalProxy(_cv_app, "session", unbound_message=_no_req_msg)
```

When passed a `ContextVar` as its first argument, `LocalProxy` resolves on every attribute access by:
1. Calling `_cv_app.get()` to retrieve the current thread's `AppContext`
2. Optionally accessing an attribute on the result (e.g., `"app"`, `"g"`, `"request"`, `"session"`)
3. Forwarding the subsequent attribute access or method call to that resolved object

#### Resolution is Deferred

```python
from flask import request    # no resolution — import is cheap
request.method               # resolution happens HERE: _cv_app.get().request.method
```

#### Push and Pop via ContextVar Tokens

`ctx.push()` (`ctx.py:433`): `self._cv_token = _cv_app.set(self)`  
`ctx.pop()` (`ctx.py:498`): `_cv_app.reset(self._cv_token)`

The token returned by `ContextVar.set()` records the previous value. `reset(token)` restores exactly that prior value, making contexts safely nestable (useful in tests and CLI).

#### Thread Safety

Each OS thread has its own `contextvars.Context` copy. PEP 567 guarantees isolation without locks. For async code, each `asyncio.Task` inherits a copy of the context at creation time, so a task pushed via `copy_current_request_context` (`ctx.py:154`) runs in an isolated copy of the current `AppContext`.

#### `_AppCtxGlobals` — `g`

`g` is an instance of `_AppCtxGlobals` (`ctx.py:30`), a plain namespace object storing arbitrary attributes in `__dict__`. It is created fresh for every `AppContext.__init__` call (`ctx.py:312`), living for exactly one request lifetime.

---

### Q17. How does blueprint registration work internally?

#### Triggering Registration

`app.register_blueprint(blueprint, **options)` is defined at `src/flask/sansio/app.py:570`. Its sole action: `blueprint.register(self, options)` (`sansio/app.py:595`).

#### `Blueprint.register` (`src/flask/sansio/blueprints.py:273`) — Steps in order:

1. **Name collision check** (bp.py:306–314) — Builds the fully-qualified name (`name_prefix.self_name`). If that name is already in `app.blueprints` pointing to a different object, raises `ValueError`.
2. **Register in `app.blueprints`** (bp.py:319) — `app.blueprints[name] = self`. Sets `self._got_registered_once = True`, which makes any future `@blueprint.route()` decorator raise `AssertionError`.
3. **Create `BlueprintSetupState`** (bp.py:321) — Calls `self.make_setup_state(app, options, first_bp_registration)`. `BlueprintSetupState.__init__` (`bp.py:41`) resolves final `url_prefix`, `subdomain`, `name_prefix`, and `url_defaults`.
4. **Static route** (bp.py:323–328) — If the blueprint has a `static_folder`, immediately calls `state.add_url_rule(...)` for the static file route.
5. **`_merge_blueprint_funcs`** (bp.py:379–410) — Copies all of the blueprint's hook registries into the app's registries (renaming None keys to `<bp_name>`): `error_handler_spec`, `view_functions`, `before_request_funcs`, `after_request_funcs`, `teardown_request_funcs`, `url_default_functions`, `url_value_preprocessors`, `template_context_processors`.
6. **Run deferred functions** (bp.py:334–335) — Iterates `self.deferred_functions` and calls each with `state`. Every `@blueprint.route(...)` decorator appended a lambda to `deferred_functions` via `Blueprint.record` (`bp.py:224`). Those lambdas now call `state.add_url_rule(...)`.
7. **CLI registration** (bp.py:337–347) — Merges the blueprint's Click command group to `app.cli` based on `cli_group` setting.
8. **Recurse into nested blueprints** (bp.py:349–377) — For each `(child_bp, child_options)` in `self._blueprints`, resolves combined subdomain/URL prefix and calls `child_bp.register(app, bp_options)` recursively.

#### `app.add_url_rule` (`src/flask/sansio/app.py:604`)

Creates a `werkzeug.routing.Rule(rule, methods=methods, **options)` and calls `self.url_map.add(rule_obj)`. The view function is stored in `self.view_functions[endpoint]`.

---

### Q18. How does Flask's error handling hierarchy work?

#### The Three Entry Points

- **`handle_user_exception(ctx, e)`** (`app.py:865`) — called inside `full_dispatch_request` for any exception escaping the view or `before_request` hooks.
- **`handle_exception(ctx, e)`** (`app.py:897`) — called from `wsgi_app`'s outer except block for exceptions that escape `full_dispatch_request` itself.
- **`handle_http_exception(ctx, e)`** (`app.py:830`) — called by `handle_user_exception` for `HTTPException` instances.

#### Exact Order in `handle_user_exception` (`app.py:865`)

1. **`BadRequestKeyError` special case** (`app.py:882`) — If `debug=True` or `TRAP_BAD_REQUEST_ERRORS=True`, sets `e.show_exception = True`.
2. **`trap_http_exception` check** (`app.py:887`) — If `TRAP_HTTP_EXCEPTIONS=True`, the exception is re-raised.
3. **Delegate `HTTPException` to `handle_http_exception`** (`app.py:888`).
4. **Search for a non-HTTP exception handler** (`app.py:890`) — `_find_error_handler(e, ctx.request.blueprints)`.
5. **If no handler found** — re-raises (`app.py:893`).
6. **If handler found** — wrapped in `ensure_sync` and called (`app.py:895`).

#### Exact Order in `handle_http_exception` (`app.py:830`)

1. If `e.code is None` (proxy exception) — returns `e` unchanged.
2. If it is a `RoutingException` — returns it unchanged.
3. **`_find_error_handler` lookup** (`app.py:860`).
4. If no handler — returns `e` (the HTTPException becomes the response directly).
5. If handler found — calls it.

#### `_find_error_handler` Lookup Order (`sansio/app.py:868`)

Priority (highest to lowest):
1. Blueprint-specific handler for this exact HTTP code (innermost blueprint first)
2. App-level handler for this exact HTTP code
3. Blueprint-specific handler matching a class in the exception's MRO
4. App-level handler matching a class in the exception's MRO
5. `None` — no match

The MRO walk means a handler registered for `HTTPException` catches all HTTP exceptions if no more specific handler exists.

#### Unhandled 500 path in `handle_exception` (`app.py:897`)

1. Fires `got_request_exception` signal.
2. Checks `PROPAGATE_EXCEPTIONS` (auto-True when `TESTING` or `DEBUG`). If True, re-raises.
3. Logs via `log_exception`.
4. Wraps in `InternalServerError(original_exception=e)`.
5. Calls `_find_error_handler(server_error, ...)` for any 500/`InternalServerError` handler.
6. Calls `finalize_request(..., from_error_handler=True)` which suppresses further exceptions.

---

### Q19. How does Flask handle sessions end to end?

#### Inbound: Cookie → Session Object

**Step 1 — `ctx.push()`** (`ctx.py:436–439`): When a request context is pushed, `_get_session()` is called immediately.

**Step 2 — `AppContext._get_session()`** (`ctx.py:381`): Calls `app.session_interface.open_session(app, self.request)`.

**Step 3 — `SecureCookieSessionInterface.open_session`** (`sessions.py:323`):
1. Calls `self.get_signing_serializer(app)` → builds a `URLSafeTimedSerializer` (from `itsdangerous`) using `app.secret_key` plus any `SECRET_KEY_FALLBACKS`, with `salt="cookie-session"` and HMAC/SHA-1.
2. Reads `request.cookies.get(SESSION_COOKIE_NAME)` (default `"session"`).
3. If no cookie: returns an empty `SecureCookieSession()`.
4. If cookie present: calls `s.loads(val, max_age=int(permanent_session_lifetime.total_seconds()))`. Verifies HMAC and timestamp. On `BadSignature`, returns empty session.
5. Session data was serialised by `TaggedJSONSerializer` (`json/tag.py`), which extends JSON with tags for `datetime`, `tuple`, `bytes`. `loads` deserialises it back.
6. Returns `SecureCookieSession(data)` — a `CallbackDict` that sets `.modified = True` on any mutation.

**Step 4 — In the view**: `session` proxy (`globals.py:60`) resolves to `_cv_app.get().session`.

#### Outbound: Session Object → `Set-Cookie` Header

**Step 1 — `Flask.process_response`** (`app.py:1415`): If session is not null, calls `session_interface.save_session(self, ctx._get_session(), response)`.

**Step 2 — `SecureCookieSessionInterface.save_session`** (`sessions.py:337`):
1. Reads all cookie parameters from config.
2. If `session.accessed`, adds `Vary: Cookie` to response headers.
3. If session is empty and modified: calls `response.delete_cookie(name, ...)`.
4. If session is empty and not modified: returns without writing.
5. Calls `should_set_cookie(app, session)`: returns `True` if `session.modified` OR (`session.permanent` AND `SESSION_REFRESH_EACH_REQUEST=True`).
6. Computes expiry via `get_expiration_time` — `now + PERMANENT_SESSION_LIFETIME` if permanent, else `None`.
7. Serialises with `self.get_signing_serializer(app).dumps(dict(session))` — `TaggedJSONSerializer` serialises to JSON, `URLSafeTimedSerializer` signs with HMAC and base64-encodes.
8. Calls `response.set_cookie(name, val, ...)` → adds `Set-Cookie` header.

---

### Q20. How does Flask's signal system work?

#### Library

Flask uses **blinker** (`src/flask/signals.py:3`). All signals are created in a private `blinker.Namespace` called `_signals`.

#### All Defined Signals (`src/flask/signals.py`)

| Signal | Line | Fired at |
|---|---|---|
| `template_rendered` | 8 | After template rendering |
| `before_render_template` | 9 | Before template rendering |
| `request_started` | 10 | Start of `full_dispatch_request` |
| `request_finished` | 11 | End of `finalize_request` |
| `request_tearing_down` | 12 | In `do_teardown_request` |
| `got_request_exception` | 13 | In `handle_exception` |
| `appcontext_tearing_down` | 14 | In `do_teardown_appcontext` |
| `appcontext_pushed` | 15 | In `AppContext.push()` |
| `appcontext_popped` | 16 | After `AppContext.pop()` |
| `message_flashed` | 17 | In `helpers.flash()` |

#### Exact Firing Points

| Signal | File:Line | Extra kwargs |
|---|---|---|
| `appcontext_pushed` | `ctx.py:434` | — |
| `request_started` | `app.py:1013` | — |
| `request_finished` | `app.py:1042` | `response` |
| `got_request_exception` | `app.py:926` | `exception` |
| `request_tearing_down` | `app.py:1449` | `exc` |
| `appcontext_tearing_down` | `app.py:1477` | `exc` |
| `appcontext_popped` | `ctx.py:499` | `exc` |
| `before_render_template` | `templating.py:126` | `template`, `context` |
| `template_rendered` | `templating.py:130` | `template`, `context` |
| `message_flashed` | `helpers.py` | `message`, `category` |

The `_async_wrapper=app.ensure_sync` argument on every `.send()` call means async signal receivers can be automatically wrapped.

---

### Q21. How does Flask's template context work? What variables are auto-available?

#### `create_jinja_environment` (`src/flask/app.py:469`)

When `Flask.jinja_env` is first accessed (lazy `cached_property` at `sansio/app.py:466`), `create_jinja_environment` is called. It injects these **global** Jinja variables (always present in every template):

```python
rv.globals.update(
    url_for=self.url_for,
    get_flashed_messages=get_flashed_messages,
    config=self.config,
    request=request,       # the LocalProxy
    session=session,       # the LocalProxy
    g=g,                   # the LocalProxy
)
```
(`src/flask/app.py:495–505`)

#### `_default_template_ctx_processor` (`src/flask/templating.py:21`)

Registered by `Scaffold.__init__` in `template_context_processors[None]`. For each template render, `update_template_context` calls it and it returns concrete object refs (not proxies) for `g` and (if `has_request`) `request`.

#### `update_template_context` (`src/flask/app.py:590`)

Called from `templating._render` (`templating.py:125`). It:
1. Builds the list of `names` to check: `(None, *reversed(request.blueprints))` — app-level processors first, then blueprint processors from outermost to innermost.
2. For each name with registered context processors, calls each and merges the result.
3. Reapplies `orig_ctx` (caller-supplied variables) last, so explicit render-time variables always win.

#### Complete List of Auto-Available Variables

| Variable | Source | Notes |
|---|---|---|
| `request` | `jinja_env.globals` + `_default_template_ctx_processor` | The current request object |
| `session` | `jinja_env.globals` (proxy only) | The current session dict |
| `g` | `jinja_env.globals` + `_default_template_ctx_processor` | Per-request scratch space |
| `config` | `jinja_env.globals` | The app's `Config` dict |
| `url_for` | `jinja_env.globals` | URL building function |
| `get_flashed_messages` | `jinja_env.globals` | Returns flash messages |

---

### Q22. How does Flask's configuration loading work — `from_object`, `from_envvar`, `from_pyfile`?

#### `Config` Class (`src/flask/config.py:50`)

`Config` is a plain `dict` subclass. It accepts only **uppercase keys** (enforced in `from_object`, `from_mapping`). There is no built-in priority mechanism — each load call simply overwrites existing keys. **The last write wins.**

#### Default Values

Set in `Flask.default_config` (`app.py:206`) — an `ImmutableDict` with defaults for all 26 built-in keys. Passed to `Config.__init__` as `defaults` and loaded first.

#### Loading Methods

**`from_object(obj)`** (`config.py:218`) — Accepts a string import path or object. Calls `dir(obj)` and sets every uppercase attribute:
```python
for key in dir(obj):
    if key.isupper():
        self[key] = getattr(obj, key)
```

**`from_pyfile(filename)`** (`config.py:187`) — Reads a `.cfg`/`.py` file, compiles it into a temporary `types.ModuleType`, execs it, then calls `self.from_object(d)`. Paths relative to `app.root_path`.

**`from_envvar(variable_name)`** (`config.py:102`) — Reads `os.environ[variable_name]` to get a file path, then calls `self.from_pyfile(rv)`. A shortcut for `from_pyfile(os.environ['VAR'])`.

**`from_prefixed_env(prefix='FLASK')`** (`config.py:126`) — Scans `os.environ` for keys starting with `FLASK_`, strips the prefix, attempts `json.loads`, then sets the key. Double-underscore `__` creates nested dict access.

**`from_file(filename, load)`** (`config.py:256`) — Generic: opens a file, calls `load(f)`, passes to `from_mapping`.

#### Priority Order

Entirely determined by **order of calls** — no automatic merge strategy. Typical pattern:
```python
app.config.from_object("yourapp.default_config")  # 1. lowest priority
app.config.from_pyfile("config.cfg", silent=True)  # 2. deployment config
app.config.from_envvar("YOURAPP_SETTINGS", silent=True)  # 3. env-specified override
app.config.from_prefixed_env()  # 4. highest priority: FLASK_* env vars
```

---

### Q23. What config keys control session cookie behaviour?

All defaults defined in `Flask.default_config` (`src/flask/app.py:206`). Accessor methods in `src/flask/sessions.py`.

| Key | Type | Default | What It Controls |
|---|---|---|---|
| `SESSION_COOKIE_NAME` | `str` | `"session"` | The cookie name in `Cookie:` and `Set-Cookie:` headers |
| `SESSION_COOKIE_DOMAIN` | `str \| None` | `None` | The `Domain` attribute; `None` means exact domain only |
| `SESSION_COOKIE_PATH` | `str \| None` | `None` | The `Path` attribute; `None` falls back to `APPLICATION_ROOT` (default `"/"`) |
| `SESSION_COOKIE_HTTPONLY` | `bool` | `True` | Sets `HttpOnly` flag; blocks JavaScript `document.cookie` access |
| `SESSION_COOKIE_SECURE` | `bool` | `False` | Sets `Secure` flag; browser only sends cookie over HTTPS |
| `SESSION_COOKIE_SAMESITE` | `str \| None` | `None` | Sets `SameSite` attribute: `"Strict"`, `"Lax"`, `"None"` |
| `SESSION_COOKIE_PARTITIONED` | `bool` | `False` | Sets `Partitioned` flag (CHIPS) for Chrome's privacy sandbox |
| `SESSION_REFRESH_EACH_REQUEST` | `bool` | `True` | If `True` and session is permanent, resets cookie expiry on every request even if session not modified |
| `PERMANENT_SESSION_LIFETIME` | `timedelta \| int` | `timedelta(days=31)` | Max age of a permanent session cookie; integers treated as seconds |
| `SECRET_KEY` | `str \| bytes \| None` | `None` | Required for signing; if `None`, `open_session` returns `None` → `NullSession` |
| `SECRET_KEY_FALLBACKS` | `list \| None` | `None` | Old keys still valid for signature verification during key rotation |

Accessor methods in `sessions.py`: `get_cookie_name` (171), `get_cookie_domain` (175), `get_cookie_path` (187), `get_cookie_httponly` (195), `get_cookie_secure` (202), `get_cookie_samesite` (208), `get_cookie_partitioned` (215), `get_expiration_time` (223), `should_set_cookie` (233).

---

### Q24. How does Flask's URL routing work internally?

#### Registration Phase

1. **`@app.route("/path")`** calls `Scaffold.route` (`sansio/scaffold.py`) which calls `self.add_url_rule(rule, endpoint, view_func, **options)`.

2. **`App.add_url_rule`** (`sansio/app.py:604`):
   - Infers endpoint name from view function name if not given.
   - Resolves `methods` set (defaults to `{"GET"}`); adds `"OPTIONS"` if `PROVIDE_AUTOMATIC_OPTIONS=True`.
   - Creates `self.url_rule_class(rule, methods=methods, **options)` — a `werkzeug.routing.Rule` object.
   - Calls `self.url_map.add(rule_obj)` to register in the `werkzeug.routing.Map`.
   - Stores `view_func` in `self.view_functions[endpoint]`.

#### The URL Map

`self.url_map` is a `werkzeug.routing.Map` instance created in `App.__init__` (`sansio/app.py:402`). It compiles all `Rule` objects into an internal regex-based matching automaton.

#### Request Time: URL Matching

3. **`AppContext.__init__`** (`ctx.py:326`) calls `app.create_url_adapter(request)`.
4. **`Flask.create_url_adapter`** (`app.py:509`) calls `url_map.bind_to_environ(request.environ, ...)`, returning a `werkzeug.routing.MapAdapter`.
5. **`AppContext.push()`** (`ctx.py:443`) calls `self.match_request()`.
6. **`AppContext.match_request()`** (`ctx.py:405`): `url_adapter.match(return_rule=True)` → `(Rule, view_args_dict)`. On no match: raises `NotFound`; on wrong method: `MethodNotAllowed`. These are stored in `request.routing_exception`.
7. **`Flask.dispatch_request`** (`app.py:966`): Checks `req.routing_exception` (re-raises if set), looks up `self.view_functions[rule.endpoint]`, calls `self.ensure_sync(view_functions[endpoint])(**view_args)`.

#### URL Building (`url_for`)

`Flask.url_for` (`app.py:1102`) / `helpers.url_for` (calls `current_app.url_for`):
1. Gets `url_adapter` from `ctx.url_adapter`.
2. Calls `self.inject_url_defaults(endpoint, values)`.
3. Calls `url_adapter.build(endpoint, values, ...)`.
4. On `BuildError`: `self.handle_url_build_error(error, endpoint, values)`.

---

### Q25. How does Flask's test client work?

#### `FlaskClient` (`src/flask/testing.py:109`)

Inherits from `werkzeug.test.Client`. The Werkzeug `Client` is not a mock — it calls the real WSGI application callable directly in-process, without any network I/O.

#### What `FlaskClient` does NOT mock

Everything runs for real:
- `Flask.wsgi_app` — the full WSGI entry point
- `AppContext.from_environ` and `ctx.push()` — full context setup including URL matching and session loading
- `Flask.full_dispatch_request` — `before_request`, `dispatch_request`, `after_request`
- `Flask.finalize_request` and `process_response`
- `session_interface.open_session` / `save_session` — real session cookie signing via `itsdangerous`
- `ctx.pop()` and teardown functions
- All signals

#### What `FlaskClient` Adds/Overrides

1. **`EnvironBuilder`** (`testing.py:27`) — Subclasses `werkzeug.test.EnvironBuilder`. Reads `app.config["SERVER_NAME"]`, `APPLICATION_ROOT`, and `PREFERRED_URL_SCHEME` to construct base URL. Uses `app.json.dumps` for JSON serialisation.
2. **`environ_base`** (`testing.py:130`) — Injects default WSGI environ values: `REMOTE_ADDR="127.0.0.1"` and `HTTP_USER_AGENT="Werkzeug/x.y.z"`.
3. **`preserve_context`** (`testing.py:127`) — When used as a context manager, sets `preserve_context = True`. Injects `werkzeug.debug.preserve_context` into the environ (`testing.py:189`). In `wsgi_app` at `app.py:1606`, Flask calls this callback instead of popping the context, keeping the `AppContext` alive until the `with` block exits.
4. **`session_transaction()`** (`testing.py:135`) — Context manager that manually opens and saves the session via the real `session_interface`.
5. **`_context_stack`** (`testing.py:129`) — An `ExitStack` that manages preserved contexts across redirects; closed before each new request.
6. **`open()`** (`testing.py:204`) — Overrides Werkzeug's `Client.open` to use `FlaskClient.EnvironBuilder`, copy environ defaults, and handle preserved contexts. Delegates to `super().open()` which calls `app.wsgi_app` directly.

---

## SECTION C — STRUCTURAL ANALYSIS (Q26–Q40)

---

### Q26. Blast radius of changing the `Flask` class in `app.py`

The `Flask` class (`src/flask/app.py:109`) is the top-level WSGI application object. Its blast radius is extremely wide.

**Direct importers of `Flask` (hop 1):**

| File | What it imports / uses |
|---|---|
| `src/flask/__init__.py:2` | `from .app import Flask as Flask` — re-exports `Flask` publicly |
| `src/flask/ctx.py:21` | TYPE_CHECKING `from .app import Flask` — type annotations throughout |
| `src/flask/globals.py:9` | TYPE_CHECKING `from .app import Flask` — `FlaskProxy` type |
| `src/flask/sessions.py:19` | TYPE_CHECKING `from .app import Flask` — method signatures |
| `src/flask/testing.py:24` | TYPE_CHECKING `from .app import Flask` — `FlaskClient`, `FlaskCliRunner` |
| `src/flask/cli.py:34` | TYPE_CHECKING `from .app import Flask` — CLI `ScriptInfo.load_app` |

**Indirect dependents via `__init__.py` (hop 2):** Any application code or library doing `from flask import Flask`, plus all test files: `tests/conftest.py`, `tests/test_basic.py`, `tests/test_appctx.py`, `tests/test_blueprints.py`, `tests/test_testing.py`, and every other test in the suite, plus `examples/` apps.

**What specifically would break:**

1. Changing `wsgi_app` / `__call__` signature — breaks all WSGI servers and middleware wrappers.
2. Changing `full_dispatch_request` / `dispatch_request` — breaks the core request lifecycle; subclasses that override these auto-detected by `__init_subclass__` at `app.py:254`.
3. Changing `handle_exception` / `handle_user_exception` / `handle_http_exception` — breaks error handling.
4. Changing `__init__` attributes — breaks all instantiation of `Flask(...)`.
5. Changing `url_for` — `helpers.url_for` delegates to it at `helpers.py:244`; Jinja template globals set to `self.url_for` at `app.py:496`.
6. Changing `create_jinja_environment` — breaks `templating.py` `Environment` and `DispatchingJinjaLoader`.
7. Changing `app_context()` / `request_context()` — breaks all context-dependent code in `ctx.py`, `helpers.py`, `templating.py`.
8. Changing `session_interface` — breaks `sessions.py` `SecureCookieSessionInterface`.
9. Changing `teardown_appcontext_funcs` — breaks `do_teardown_appcontext` and all registered teardown functions.
10. Changing `response_class` / `request_class` — breaks `wrappers.py` `Request`/`Response`.

**Total files in blast radius: ~15 source files + all test files (30+) + all example apps.**

---

### Q27. Complete call graph from `full_dispatch_request`

`full_dispatch_request` is at `src/flask/app.py:992`.

```
full_dispatch_request(ctx)                          app.py:992
├── request_started.send(self)                      signals.py  [blinker signal]
├── preprocess_request(ctx)                         app.py:1366
│   ├── url_value_preprocessors[name](endpoint, view_args)
│   └── before_request_funcs[name]()
│       ├── ensure_sync(before_func)()              app.py:1065
│       │   └── async_to_sync(func) [if async]      app.py:1079 → asgiref
│       └── (user before_request callbacks)
├── dispatch_request(ctx)                           app.py:966
│   ├── raise_routing_exception(req) [if routing_exception]
│   ├── make_default_options_response(ctx) [if OPTIONS]  app.py:1053
│   │   └── ctx.url_adapter.allowed_methods()       [werkzeug]
│   └── ensure_sync(view_functions[endpoint])(**view_args)  app.py:990
│       └── (user view function)
├── handle_user_exception(ctx, e)   [on Exception]  app.py:865
│   ├── handle_http_exception(ctx, e) [if HTTPException]   app.py:830
│   │   ├── _find_error_handler(e, blueprints)       sansio/app.py
│   │   └── ensure_sync(handler)(e)
│   └── _find_error_handler(e, blueprints)           sansio/app.py
│       └── ensure_sync(handler)(e)
└── finalize_request(ctx, rv)                       app.py:1021
    ├── make_response(rv)                           app.py:1224
    │   ├── response_class(rv, status, headers)     [werkzeug Response]
    │   ├── json.response(rv) [if dict/list]        json/provider.py
    │   └── response_class.force_type(rv, environ)  [werkzeug]
    ├── process_response(ctx, response)             app.py:1394
    │   ├── ensure_sync(_after_request_functions)()
    │   ├── ensure_sync(after_request_funcs[name])()
    │   └── session_interface.save_session(app, session, response)  sessions.py
    │       └── response.set_cookie(...)            [werkzeug]
    └── request_finished.send(self, response=response) [blinker signal]
```

**Exception path in `wsgi_app` (caller of `full_dispatch_request`, `app.py:1592`):**
```
wsgi_app(environ, start_response)
├── request_context(environ) → AppContext.from_environ()
├── ctx.push()
├── full_dispatch_request(ctx)   ← described above
├── handle_exception(ctx, e)     [on unhandled Exception, app.py:897]
│   ├── got_request_exception.send(...)
│   ├── log_exception(ctx, exc_info)     app.py:950
│   │   └── self.logger.error(...)
│   ├── _find_error_handler(server_error, blueprints)
│   └── finalize_request(ctx, server_error, from_error_handler=True)
└── ctx.pop(error)
    ├── do_teardown_request(ctx, exc)   app.py:1420
    │   ├── teardown_request_funcs callbacks
    │   └── request_tearing_down.send(...)
    └── do_teardown_appcontext(ctx, exc)  app.py:1453
        ├── teardown_appcontext_funcs callbacks
        └── appcontext_tearing_down.send(...)
```

---

### Q28. Circular import dependencies between Flask modules

The graph analysis found **53 circular dependency chains**. The unique module-level cycles are:

| # | Cycle | Length |
|---|---|---|
| 1 | `__init__.py` → `app.py` → `__init__.py` | 2 |
| 2 | `__init__.py` → `app.py` → `ctx.py` → `__init__.py` | 3 |
| 3 | `app.py` → `ctx.py` → `globals.py` → `app.py` | 3 (most critical) |
| 4 | `ctx.py` → `globals.py` → `ctx.py` | 2 |
| 5 | `globals.py` → `sessions.py` → `json/tag.py` → `json/__init__.py` → `globals.py` | 4 |
| 6 | `sansio/app.py` → `config.py` → `sansio/app.py` | 2 |
| 7 | `ctx.py` → `globals.py` → `sessions.py` → `json/...` → `json/provider.py` → `sansio/app.py` → `ctx.py` | 7 |
| 8 | `globals.py` → `sessions.py` → `json/...` → `json/provider.py` → `sansio/app.py` → `helpers.py` → `globals.py` | 6 |
| 9 | `helpers.py` → `wrappers.py` → `helpers.py` | 2 |
| 10 | `globals.py` → ... → `sansio/app.py` → `helpers.py` → `wrappers.py` → `debughelpers.py` → `blueprints.py` → `cli.py` → `globals.py` | 9 |

All 53 cycles are variations of these base cycles. They are managed through:
- `if t.TYPE_CHECKING:` guards — imports inside `TYPE_CHECKING` blocks do not execute at runtime (e.g., `globals.py:8–13`, `ctx.py:17–23`).
- Deferred imports inside function bodies (e.g., `from .testing import EnvironBuilder` inside `test_request_context` at `app.py:1555`).
- Python's module caching resolves apparent cycles at import time.

---

### Q29. Files that directly import from `globals.py`

| File | What it imports |
|---|---|
| `src/flask/__init__.py` (lines 10–15) | `current_app`, `g`, `request`, `session` |
| `src/flask/app.py` (lines 34–38) | `_cv_app`, `app_ctx`, `g`, `request`, `session` |
| `src/flask/blueprints.py` (line 8) | `current_app` |
| `src/flask/cli.py` (line 23) | `current_app` |
| `src/flask/ctx.py` (line 12) | `_cv_app` |
| `src/flask/debughelpers.py` (line 9) | `_cv_app` |
| `src/flask/helpers.py` (lines 17–21) | `_cv_app`, `app_ctx`, `current_app`, `request`, `session` |
| `src/flask/json/__init__.py` (line 5) | `current_app` |
| `src/flask/logging.py` (line 6) | `request` |
| `src/flask/sansio/app.py` (line 23) | `_AppCtxGlobals` (TYPE_CHECKING only, from `..ctx`) |
| `src/flask/templating.py` (line 11) | `app_ctx` |
| `src/flask/views.py` (lines 6–7) | `current_app`, `request` |
| `src/flask/wrappers.py` (line 11) | `current_app` |
| `tests/conftest.py` | `app_ctx` |
| `tests/test_appctx.py` | `app_ctx` |
| `tests/test_basic.py` | `app_ctx` |
| `tests/test_session_interface.py` | `app_ctx` |
| `tests/test_testing.py` | `_cv_app` |

---

### Q30. Functions that call `AppContext.push()` — every caller with file and line

`AppContext.push()` is defined at `src/flask/ctx.py:416`.

| Caller | File | Line | Mechanism |
|---|---|---|---|
| `AppContext.__enter__` | `src/flask/ctx.py` | 507 | `self.push()` — called by `with ctx:` blocks |
| `Flask.wsgi_app` | `src/flask/app.py` | 1596 | `ctx.push()` — called at start of every HTTP request |

**Transitively triggered by:**
- `Flask.app_context()` returns an `AppContext`; users call `with app.app_context(): ...` which calls `__enter__` → `push()`.
- `Flask.request_context(environ)` creates an `AppContext`; pushed by the caller.
- `Flask.test_request_context(...)` creates and returns an `AppContext` via `request_context`.
- `stream_with_context` in `helpers.py:133` uses `with ctx:` on an existing context.
- `copy_current_request_context` wrapper in `ctx.py:203` uses `with original.copy() as ctx:`.

---

### Q31. Blast radius of changing `RequestContext` or `AppContext`

In Flask 3.2+, `RequestContext` is an alias for `AppContext` (raises `DeprecationWarning`, `ctx.py:528–540`). Analysis covers `AppContext`.

**Files that directly depend on `AppContext`:**

| File | How | What breaks |
|---|---|---|
| `src/flask/app.py:33` | `from .ctx import AppContext` | All WSGI entry points: `wsgi_app`, `request_context`, `app_context`, `test_request_context`, all `ctx:`-typed method signatures |
| `src/flask/globals.py:11` | TYPE_CHECKING `from .ctx import AppContext` | `_cv_app: ContextVar[AppContext]`, all five proxy objects |
| `src/flask/templating.py:10` | `from .ctx import AppContext` | `_render`, `render_template`, `render_template_string`, `stream_template*` |
| `src/flask/sansio/app.py:23` | `from ..ctx import _AppCtxGlobals` | `App.app_ctx_globals_class`, `do_teardown_appcontext` |
| `src/flask/__init__.py:5-8` | Re-exports `after_this_request`, `copy_current_request_context`, `has_app_context`, `has_request_context` | Public API breakage |

**Indirect dependents (hop 2 — anything using `current_app`, `g`, `request`, `session`):**
`src/flask/helpers.py`, `src/flask/views.py`, `src/flask/wrappers.py`, `src/flask/blueprints.py`, `src/flask/cli.py`, `src/flask/json/__init__.py`, `src/flask/logging.py`.

**Total: ~13 source files directly or 1 hop away; entire public API (via `__init__.py`) at hop 2.**

---

### Q32. What calls `teardown_appcontext`? Full trigger chain.

`teardown_appcontext` is the **decorator** defined at `src/flask/sansio/app.py:827`. It appends functions to `self.teardown_appcontext_funcs` (line 854).

The **execution** happens through `do_teardown_appcontext` (`app.py:1453`), called by `AppContext.pop()` (`ctx.py:496`).

**Complete trigger chain (in order):**

```
1. HTTP request ends OR CLI command ends OR manual `with app.app_context()` exits
   → ctx.__exit__(exc_type, exc_val, tb)         ctx.py:510
   → ctx.pop(exc_val)                            ctx.py:446
       ├── do_teardown_request(ctx, exc)          app.py:1420   [if has_request]
       │   ├── teardown_request_funcs callbacks   (user registered)
       │   └── request_tearing_down.send(...)     signals.py
       └── do_teardown_appcontext(ctx, exc)       app.py:1453
           ├── reversed(teardown_appcontext_funcs)  [LIFO order]
           │   └── ensure_sync(func)(exc)          (user registered via @teardown_appcontext)
           └── appcontext_tearing_down.send(...)   signals.py

2. Via wsgi_app `finally` block (app.py:1616):
   ctx.pop(error)    → same chain as above

3. Via Flask CLI (cli.py):
   AppContext is pushed for each CLI command; popped after → same chain
```

**Registration:** `@app.teardown_appcontext` calls `teardown_appcontext(f)` at `sansio/app.py:827`, appending `f` to `teardown_appcontext_funcs`. Execution always runs in **LIFO** (reversed) order. Signal emitted after all teardown functions: `appcontext_tearing_down` from `signals.py`.

---

### Q33. Files affected if the `Blueprint` class changes — dependency chain depth

`Blueprint` is defined in two places:
- `src/flask/sansio/blueprints.py` (pure sansio `SansioBlueprint`)
- `src/flask/blueprints.py:18` (`Blueprint(SansioBlueprint)` — adds CLI and file-serving)

**Hop 1 — direct importers of `blueprints.py`:**

| File | Import |
|---|---|
| `src/flask/__init__.py:3` | `from .blueprints import Blueprint as Blueprint` |
| `src/flask/debughelpers.py:8` | `from .blueprints import Blueprint` |

**Hop 1 — direct importers of `sansio/blueprints.py`:**

| File | Import |
|---|---|
| `src/flask/blueprints.py:10` | `from .sansio.blueprints import Blueprint as SansioBlueprint` |
| `src/flask/sansio/app.py:41` | TYPE_CHECKING `from .blueprints import Blueprint` |

**Hop 2 — via `__init__.py` or `sansio/app.py`:** Everything doing `from flask import Blueprint` (all user code and test files), `src/flask/app.py` (has `register_blueprint`, `iter_blueprints`), `src/flask/templating.py` (`DispatchingJinjaLoader._iter_loaders` iterates `app.iter_blueprints()`).

**What would break:**
1. `SansioBlueprint` changes — `BlueprintSetupState`, all `register_blueprint` logic, all `deferred_functions` execution.
2. `Blueprint.__init__` (WSGI layer) — `.cli` attribute, `AppGroup`, CLI command registration.
3. `send_static_file` / `get_send_file_max_age` — static file serving for blueprints.
4. `blueprints` attribute on requests — used in `preprocess_request`, `process_response`, `do_teardown_request`, `_find_error_handler`.

**Dependency chain depth: 2 hops** to all application-level code.

---

### Q34. What calls `handle_exception`? Full error propagation chain.

`handle_exception` is defined at `src/flask/app.py:897`. It is the last-resort handler for unhandled non-HTTP exceptions.

**Complete error propagation chain:**

```
[Exception raised in view function / before_request / after_request]
    │
    ▼
full_dispatch_request(ctx)                     app.py:1012
  except Exception as e:
    → handle_user_exception(ctx, e)            app.py:865
        if isinstance(e, HTTPException):
          → handle_http_exception(ctx, e)      app.py:830
              → _find_error_handler(e, ...)    sansio/app.py
              → ensure_sync(handler)(e)        [user error handler, or returns e unchanged]
        else:
          → _find_error_handler(e, ...)        sansio/app.py
          if handler: → ensure_sync(handler)(e) [user error handler]
          else:       → raise  [re-raises to caller]
    → finalize_request(ctx, rv)                [if handle_user_exception returned a value]

    [If handle_user_exception re-raised, propagates up to wsgi_app:]

wsgi_app(environ, start_response)             app.py:1592
  except Exception as e:
    → handle_exception(ctx, e)                app.py:897
        → got_request_exception.send(self, exception=e)   [signal]
        if PROPAGATE_EXCEPTIONS (debug/testing):
            → raise  [propagates to WSGI server / test runner]
        else:
            → log_exception(ctx, exc_info)    app.py:950
                → self.logger.error(...)
            → InternalServerError(original_exception=e)
            → _find_error_handler(server_error, blueprints)
            → ensure_sync(handler)(server_error)  [if 500 handler registered]
            → finalize_request(ctx, server_error, from_error_handler=True)
                → make_response(server_error)
                → process_response(ctx, response)
                  except Exception:  [if from_error_handler=True: log and ignore]
                    → self.logger.exception("Request finalizing failed...")
  except:  [non-Exception BaseException]
    error = sys.exc_info()[1]
    raise  [propagates unconditionally]
  finally:
    ctx.pop(error)  [always runs teardown]
```

**Callers of `handle_exception`:** Only `wsgi_app` at `app.py:1600` calls it directly. It is the last resort handler. `__init_subclass__` at `app.py:263` registers it for compatibility wrapping in subclasses.

---

### Q35. Full dependency chain of `app.py` — direct imports and total blast radius

**Direct runtime imports from `src/flask/app.py` (lines 1–54):**

| Module | Lines |
|---|---|
| `click` | 16 |
| `werkzeug.datastructures` | 17–18 |
| `werkzeug.exceptions` | 19–21 |
| `werkzeug.routing` | 22–26 |
| `werkzeug.serving` | 27 |
| `werkzeug.wrappers` | 28 |
| `werkzeug.wsgi` | 29 |
| `src/flask/cli` | 31 |
| `src/flask/typing` | 32 |
| `src/flask/ctx` | 33 |
| `src/flask/globals` | 34–38 |
| `src/flask/helpers` | 39–43 |
| `src/flask/sansio/app` | 44 |
| `src/flask/sessions` | 45–46 |
| `src/flask/signals` | 47–51 |
| `src/flask/templating` | 52 |
| `src/flask/wrappers` | 53–54 |

**Deferred / conditional imports (inside functions):**
- `src/flask/testing` — imported in `test_client()`, `test_cli_runner()`, `test_request_context()`, `wsgi_app()`
- `src/flask/debughelpers` — imported in `raise_routing_exception()` (`app.py:586`)

**Transitive closure (unique Flask source files in blast radius):**

`app.py` → `sansio/app.py` → `config.py`, `ctx.py`, `helpers.py`, `logging.py`, `templating.py`, `json/provider.py`, `json/__init__.py`, `json/tag.py`  
`app.py` → `ctx.py` → `globals.py` → `sessions.py`  
`app.py` → `helpers.py` → `wrappers.py` → `debughelpers.py` → `blueprints.py` → `sansio/blueprints.py` → `sansio/scaffold.py`  
`app.py` → `signals.py`  
`app.py` → `cli.py`

**Total unique Flask source files in blast radius: approximately 18 Flask-internal modules**, plus all of `werkzeug`, `click`, `jinja2` as external dependencies.

The 18 internal modules: `cli`, `ctx`, `globals`, `helpers`, `sansio/app`, `sansio/scaffold`, `sansio/blueprints`, `sessions`, `signals`, `templating`, `wrappers`, `config`, `logging`, `typing`, `json/__init__`, `json/provider`, `json/tag`, `blueprints`, `debughelpers`.

---

### Q36. Functions that call `url_for` internally within Flask

| Caller | File | Line | Context |
|---|---|---|---|
| `Flask.url_for` (the method) | `src/flask/app.py` | 1102 | Implements URL building; calls `inject_url_defaults`, `url_adapter.build`, `handle_url_build_error` |
| `helpers.url_for` (the function) | `src/flask/helpers.py` | 244 | Calls `current_app.url_for(...)` — the public proxy function exposed at `flask.url_for` |
| Jinja global `url_for` | `src/flask/app.py` | 496 | `rv.globals.update(url_for=self.url_for)` — registered as Jinja2 global so templates can call `{{ url_for(...) }}` |

**Internal call chain for `url_for`:**
```
flask.url_for (helpers.py:244)
  → current_app.url_for(endpoint, ...)    (app.py:1102)
      → self.inject_url_defaults(endpoint, values)   sansio/app.py:960
      → url_adapter.build(endpoint, values, ...)      [werkzeug MapAdapter]
      [on BuildError]:
      → self.handle_url_build_error(error, endpoint, values)   sansio/app.py:983
```

---

### Q37. If `ctx.py` were deleted — everything that would break

**Direct importers of `ctx.py` (hop 1):**

| File | What breaks |
|---|---|
| `src/flask/app.py:33` | `from .ctx import AppContext` — entire request/app context system; `wsgi_app` and all 12+ ctx-parameterised methods fail |
| `src/flask/globals.py:10–11` | TYPE_CHECKING `_AppCtxGlobals`, `AppContext` — `_cv_app` type collapses; `current_app`, `g`, `request`, `session` become unresolvable |
| `src/flask/templating.py:10` | `from .ctx import AppContext` — `_render`, all `render_template*` and `stream_template*` fail |
| `src/flask/__init__.py:5–8` | `after_this_request`, `copy_current_request_context`, `has_app_context`, `has_request_context` — public API gone |
| `src/flask/sansio/app.py:23` | `from ..ctx import _AppCtxGlobals` — `app_ctx_globals_class` and `g` teardown logic fail |

**Indirect dependents (hop 2) — via the above:**
`src/flask/helpers.py`, `src/flask/views.py`, `src/flask/wrappers.py`, `src/flask/blueprints.py`, `src/flask/cli.py`, `src/flask/json/__init__.py`, `src/flask/logging.py`, `src/flask/sessions.py`.

**Hop 3:** All user application code using `flask.request`, `flask.g`, `flask.current_app`, `flask.session`. All Flask test files (`tests/test_appctx.py`, `tests/test_basic.py`, `tests/test_blueprints.py`, `tests/test_testing.py`, etc.)

**Conclusion:** Deleting `ctx.py` would break **every file in the Flask source tree** and all downstream user applications. It is the second-most central file after `app.py`.

---

### Q38. Module dependency graph — 10 most connected files

From `codebase_graph_stats` (106 total files, 300 edges):

| Rank | File | Connections | Notes |
|---|---|---|---|
| 1 | `src/flask/__init__.py` | **128** | Re-exports entire public API; hub connecting all modules |
| 2 | `src/flask/app.py` | **37** | Core `Flask` class; imports 17+ modules |
| 3 | `src/flask/globals.py` | **34** | Imported by 13 source files; defines all five context proxies |
| 4 | `src/flask/helpers.py` | **32** | Imported by 10+ files; exports `url_for`, `flash`, `send_file`, etc. |
| 5 | `src/flask/sansio/app.py` | **25** | Base `App` class; imported by `app.py`, `templating.py`, `sansio/scaffold.py` |
| 6 | `src/flask/cli.py` | **23** | CLI framework; imports `globals`, `app`, `typing`, `click` |
| 7 | `src/flask/signals.py` | **20** | Blinker signals imported by `app.py`, `ctx.py`, `templating.py`, `helpers.py` |
| 8 | `src/flask/ctx.py` | **17** | `AppContext`; imports `globals`, `helpers`, `signals`; imported by `app`, `templating`, `__init__` |
| 9 | `src/flask/templating.py` | **17** | Jinja2 integration; imports `ctx`, `globals`, `helpers`, `signals` |
| 10 | `src/flask/wrappers.py` | **17** | `Request`/`Response`; imported by `app`, `helpers`, `ctx` (TYPE_CHECKING) |

**Core module dependency graph (Mermaid):**
```
flowchart TD
    __init__ --> app
    __init__ --> blueprints
    __init__ --> ctx
    __init__ --> globals
    __init__ --> helpers
    app --> ctx
    app --> globals
    app --> helpers
    app --> sansio/app
    app --> sessions
    app --> signals
    app --> templating
    app --> wrappers
    ctx --> globals
    ctx --> helpers
    ctx --> signals
    globals -. TYPE_CHECKING .-> app
    globals -. TYPE_CHECKING .-> ctx
    helpers --> globals
    helpers --> wrappers
    sansio/app --> ctx
    sansio/app --> config
    sansio/app --> helpers
    sansio/app --> logging
    sansio/app --> templating
    templating --> ctx
    templating --> globals
    blueprints --> globals
    blueprints --> helpers
    blueprints --> sansio/blueprints
```

---

### Q39. 2-hop call graph from `wsgi_app` — unique functions reached

`wsgi_app` is at `src/flask/app.py:1566`.

**Hop 1 — direct calls from `wsgi_app`:**

| Function | File:Line |
|---|---|
| `self.request_context(environ)` | `app.py:1501` |
| `ctx.push()` | `ctx.py:416` |
| `self.full_dispatch_request(ctx)` | `app.py:992` |
| `self.handle_exception(ctx, e)` | `app.py:897` |
| `ctx.pop(error)` | `ctx.py:446` |
| `environ["werkzeug.debug.preserve_context"](ctx)` | (werkzeug debugger, conditional) |

**Hop 2 — what each hop-1 function calls:**

`request_context(environ)` → `AppContext.from_environ(self, environ)` → `AppContext.__init__` → `app.create_url_adapter(request)` → `app.request_class(environ)` (i.e., `Request.__init__`)

`ctx.push()` → `_cv_app.set(self)`, `appcontext_pushed.send(...)`, `self._get_session()`, `self.match_request()`

`full_dispatch_request(ctx)` → `request_started.send(...)`, `self.preprocess_request(ctx)`, `self.dispatch_request(ctx)`, `handle_user_exception(ctx, e)`, `self.finalize_request(ctx, rv)`

`handle_exception(ctx, e)` → `got_request_exception.send(...)`, `log_exception(ctx, exc_info)`, `_find_error_handler(...)`, `finalize_request(..., from_error_handler=True)`

`ctx.pop(error)` → `do_teardown_request(ctx, exc)`, `request.close()`, `do_teardown_appcontext(ctx, exc)`, `_cv_app.reset(token)`, `appcontext_popped.send(...)`

**Total unique functions reached at 2 hops:**

- Hop 1: **6 functions**
- Hop 2 adds: `AppContext.from_environ`, `AppContext.__init__`, `create_url_adapter`, `Request.__init__`, `_cv_app.set`, `appcontext_pushed.send`, `_get_session`, `match_request`, `request_started.send`, `preprocess_request`, `dispatch_request`, `handle_user_exception`, `finalize_request`, `got_request_exception.send`, `log_exception`, `_find_error_handler`, `do_teardown_request`, `request.close`, `do_teardown_appcontext`, `_cv_app.reset`, `appcontext_popped.send`

**Total unique functions at 2 hops: approximately 27 distinct functions.**

---

### Q40. Every function in Flask that catches `Exception` broadly

| Function | File | Line | What it does with the caught exception |
|---|---|---|---|
| `full_dispatch_request` | `src/flask/app.py` | 1017 | `except Exception as e:` → delegates to `self.handle_user_exception(ctx, e)`, which either invokes a registered error handler or re-raises |
| `finalize_request` | `src/flask/app.py` | 1045 | `except Exception:` → if `from_error_handler=True`: logs via `self.logger.exception(...)` and silently swallows; if `from_error_handler=False`: re-raises |
| `wsgi_app` | `src/flask/app.py` | 1598 | `except Exception as e:` → stores as `error`, delegates to `self.handle_exception(ctx, e)` which logs it and attempts a 500 response |
| `Config.from_prefixed_env` | `src/flask/config.py` | 163 | `except Exception:` → silently passes — keeps the env var value as a raw string if JSON parsing fails |
| `FlaskGroup.main` (CLI) | `src/flask/cli.py` | 650 | `except Exception:` → calls `click.secho(traceback.format_exc(), fg="red", err=True)` — prints full traceback to stderr but does not re-raise |
| `run_simple` caller in CLI | `src/flask/cli.py` | 956 | `except Exception as e:` → if running from reloader: prints traceback then stores and re-raises after delay; otherwise shows error and exits |

**Summary of behaviors:**

| Function | Behavior on catch |
|---|---|
| `full_dispatch_request` | Routes to user-registered handlers; re-raises if no handler found |
| `finalize_request` | Suppresses if already in error handler path; re-raises otherwise |
| `wsgi_app` | Always produces a 500 response; never propagates to WSGI server (unless `PROPAGATE_EXCEPTIONS=True`) |
| `Config.from_prefixed_env` | Silently ignores — treats value as string |
| `FlaskGroup.main` | Logs to stderr, continues — error display only |
| CLI `run_simple` caller | Context-dependent: re-raise in reloader, exit in normal mode |
