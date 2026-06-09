# Flask Internals: Execution Paths

Key user-facing operations traced from entry point to final output, with exact file and function names.

---

## 1. Handling an HTTP Request (WSGI dispatch)

The central operation — called by any WSGI server for every request.

1. **WSGI server** calls `Flask.__call__(environ, start_response)`
   — `src/flask/app.py:1618`
2. `Flask.__call__` delegates to `Flask.wsgi_app(environ, start_response)`
   — `src/flask/app.py:1566`
3. `Flask.wsgi_app` calls `Flask.request_context(environ)`
   — `src/flask/app.py:1501`
4. `Flask.request_context` calls `AppContext.from_environ(app, environ)`
   — `src/flask/ctx.py:340`
5. `AppContext.from_environ` calls `app.request_class(environ)` to build the `Request` object, then `AppContext.__init__`
   — `src/flask/ctx.py:346–348`
6. `AppContext.__init__` calls `app.create_url_adapter(request)`
   — `src/flask/app.py:509`, called from `src/flask/ctx.py:326`
7. `Flask.create_url_adapter` calls `url_map.bind_to_environ(environ)` (Werkzeug) to create a `MapAdapter`
   — `src/flask/app.py:548`
8. Back in `Flask.wsgi_app`: `ctx.push()` is called
   — `src/flask/ctx.py:416`
9. `AppContext.push` stores context in `_cv_app` (a `ContextVar`), fires `appcontext_pushed` signal, then calls `ctx._get_session()` and `ctx.match_request()`
   — `src/flask/ctx.py:433–444`
10. `AppContext._get_session` calls `session_interface.open_session(app, request)`
    — `src/flask/ctx.py:387–388`
11. `SecureCookieSessionInterface.open_session` reads the cookie, calls `URLSafeTimedSerializer.loads` (itsdangerous) to verify and decode it
    — `src/flask/sessions.py:323–335`
12. `AppContext.match_request` calls `url_adapter.match(return_rule=True)` (Werkzeug routing), stores `url_rule` and `view_args` on the request
    — `src/flask/ctx.py:410–414`
13. Back in `Flask.wsgi_app`: `Flask.full_dispatch_request(ctx)` is called
    — `src/flask/app.py:992`
14. `Flask.full_dispatch_request` fires `request_started` signal, then calls `Flask.preprocess_request(ctx)`
    — `src/flask/app.py:1013–1014`
15. `Flask.preprocess_request` runs all registered `url_value_preprocessors` then all `before_request` handlers; returns early if any handler returns a value
    — `src/flask/app.py:1366–1392`
16. If no early return, `Flask.dispatch_request(ctx)` is called
    — `src/flask/app.py:966`
17. `Flask.dispatch_request` looks up `view_functions[rule.endpoint]`, wraps it with `Flask.ensure_sync` if async, and calls it with `view_args`
    — `src/flask/app.py:990`
18. The **view function** executes and returns a value (`str`, `dict`, `Response`, tuple, etc.)
19. `Flask.full_dispatch_request` passes the return value to `Flask.finalize_request(ctx, rv)`
    — `src/flask/app.py:1019`
20. `Flask.finalize_request` calls `Flask.make_response(rv)`
    — `src/flask/app.py:1039`
21. `Flask.make_response` normalizes the return value into a `Response` object — converts `str`/`bytes` to `Response`, `dict`/`list` to JSON via `app.json.response(rv)`, or coerces a Werkzeug response
    — `src/flask/app.py:1224–1364`
22. `Flask.finalize_request` calls `Flask.process_response(ctx, response)`
    — `src/flask/app.py:1041`
23. `Flask.process_response` runs `after_this_request` callbacks, then all `after_request` handlers (in reverse registration order), then calls `session_interface.save_session(app, session, response)`
    — `src/flask/app.py:1394–1418`
24. `SecureCookieSessionInterface.save_session` calls `URLSafeTimedSerializer.dumps(dict(session))` and sets the `Set-Cookie` header on the response
    — `src/flask/sessions.py:337–385`
25. `Flask.finalize_request` fires `request_finished` signal
    — `src/flask/app.py:1042–1044`
26. `Flask.wsgi_app` calls `response(environ, start_response)` — the Werkzeug `Response.__call__` that writes status, headers, and body to the WSGI output
    — `src/flask/app.py:1604`
27. `Flask.wsgi_app` calls `ctx.pop(error)` in the `finally` block
    — `src/flask/app.py:1616`
28. `AppContext.pop` calls `app.do_teardown_request(ctx, exc)` then `app.do_teardown_appcontext(ctx, exc)`, resets `_cv_app`, and fires `appcontext_popped`
    — `src/flask/ctx.py:446–504`

---

## 2. Rendering a Template (`render_template`)

1. View function calls `render_template("template.html", **context)`
   — `src/flask/templating.py:136`
2. `render_template` gets the active context via `app_ctx._get_current_object()`
   — `src/flask/templating.py:146`
3. Calls `ctx.app.jinja_env.get_or_select_template(template_name_or_list)` (Jinja2)
   — `src/flask/templating.py:147`
4. Jinja2's `Environment.get_or_select_template` calls the loader's `get_source`
   — Jinja2 internals
5. `DispatchingJinjaLoader.get_source` dispatches to `_get_source_fast` (or `_get_source_explained` if `EXPLAIN_TEMPLATE_LOADING` is set)
   — `src/flask/templating.py:57–62`
6. `DispatchingJinjaLoader._get_source_fast` iterates `_iter_loaders`, checking the app's `jinja_loader` first, then each blueprint's `jinja_loader`
   — `src/flask/templating.py:88–96`
7. The matching `FileSystemLoader` (Jinja2 built-in) reads the template file from disk and returns the source string
8. Jinja2 compiles the template source into a `Template` object (cached after first load)
9. `render_template` calls `_render(ctx, template, context)`
   — `src/flask/templating.py:148`
10. `_render` calls `app.update_template_context(ctx, context)`
    — `src/flask/templating.py:125`, implementation at `src/flask/app.py:590`
11. `Flask.update_template_context` runs all registered `template_context_processors` (including `_default_template_ctx_processor`, which injects `request`, `g`, `session`) and merges their output into `context`
    — `src/flask/app.py:603–618`; `src/flask/templating.py:21–33`
12. `_render` fires `before_render_template` signal
    — `src/flask/templating.py:126–128`
13. `_render` calls `template.render(context)` (Jinja2), which executes the compiled template and returns the rendered string
    — `src/flask/templating.py:129`
14. `_render` fires `template_rendered` signal
    — `src/flask/templating.py:130–132`
15. The rendered HTML string is returned to the view function

---

## 3. Registering a Route (`@app.route`)

1. Developer applies `@app.route("/path", methods=["GET"])` decorator to a view function
   — `src/flask/sansio/scaffold.py` (`Scaffold.route`)
2. `Scaffold.route` calls `self.add_url_rule(rule, endpoint, f, **options)`
   — `src/flask/sansio/scaffold.py`
3. `App.add_url_rule` (overrides scaffold): resolves the endpoint name (defaults to `f.__name__`), creates a `Rule` object, adds it to `self.url_map`, and stores `view_func` in `self.view_functions[endpoint]`
   — `src/flask/sansio/app.py` (`App.add_url_rule`)
4. `url_map.add(rule)` (Werkzeug `Map.add`) compiles the URL pattern into a regex and registers it for routing

---

## 4. Blueprint Registration (`app.register_blueprint`)

1. `app.register_blueprint(bp, url_prefix="/api")`
   — `src/flask/sansio/app.py` (`App.register_blueprint`)
2. `App.register_blueprint` creates a `BlueprintSetupState(bp, app, options, first_registration)`
   — `src/flask/sansio/blueprints.py` (`BlueprintSetupState.__init__`)
3. `Blueprint.register(app, options)` is called, which iterates `bp.deferred_functions` — all route/error-handler/filter registrations deferred at blueprint definition time
   — `src/flask/sansio/blueprints.py` (`Blueprint.register`)
4. Each deferred function is called with `BlueprintSetupState`; they call back into `state.add_url_rule(...)`, `state.app.add_url_rule(...)`, etc., registering all routes and handlers on the app's `url_map` and `view_functions`
5. Blueprint is appended to `app.blueprints` dict and its name added to any nested blueprint's list

---

## 5. Session Read and Write

### Reading the session (on request start, step 10–11 of Request Handling above)

1. `AppContext._get_session()` is called during `ctx.push()`
   — `src/flask/ctx.py:381`
2. Calls `SecureCookieSessionInterface.open_session(app, request)`
   — `src/flask/sessions.py:323`
3. `open_session` calls `self.get_signing_serializer(app)` → creates `URLSafeTimedSerializer` with the app's `SECRET_KEY`
   — `src/flask/sessions.py:303`
4. Reads cookie value via `request.cookies.get(self.get_cookie_name(app))`
   — `src/flask/sessions.py:327`
5. Calls `serializer.loads(val, max_age=...)` (itsdangerous): verifies HMAC signature, checks expiry, then deserializes with `TaggedJSONSerializer`
   — `src/flask/sessions.py:332`
6. Returns a `SecureCookieSession` dict; stored as `ctx._session`

### Writing the session (on response finalization, step 23–24 above)

1. `Flask.process_response` calls `session_interface.save_session(app, session, response)`
   — `src/flask/app.py:1416`
2. `SecureCookieSessionInterface.save_session` checks `should_set_cookie(app, session)` (skip if unmodified and not permanent-refresh)
   — `src/flask/sessions.py:369`
3. Calls `get_signing_serializer(app).dumps(dict(session))` → `TaggedJSONSerializer` serializes, itsdangerous signs with HMAC
   — `src/flask/sessions.py:373`
4. Calls `response.set_cookie(name, val, expires=..., ...)` (Werkzeug) to add the `Set-Cookie` header
   — `src/flask/sessions.py:374–384`
5. Adds `Vary: Cookie` to response headers
   — `src/flask/sessions.py:385`

---

## 6. URL Building (`url_for`)

1. Template or view calls `url_for("endpoint", key=value)`
   — `src/flask/helpers.py:200`
2. `helpers.url_for` delegates to `current_app.url_for(endpoint, ...)`
   — `src/flask/helpers.py:244`
3. `Flask.url_for` runs `self.inject_url_defaults(endpoint, values)` — applies any `@app.url_defaults` callbacks
   — `src/flask/app.py:1202`
4. Calls `url_adapter.build(endpoint, values, ...)` (Werkzeug `MapAdapter.build`) — matches endpoint to a URL rule and substitutes variable parts
   — `src/flask/app.py:1205`
5. If `BuildError` is raised, calls `self.handle_url_build_error(error, endpoint, values)` — tries registered `url_build_error_handlers`, otherwise re-raises
   — `src/flask/app.py:1216`
6. Appends `#anchor` fragment if `_anchor` was given
   — `src/flask/app.py:1218–1220`
7. Returns the URL string

---

## 7. Serving a Static File

1. Request arrives for a path matching `<static_url_path>/<filename>` (registered in `Flask.__init__`)
   — `src/flask/app.py:358–363`
2. Route dispatches to `Flask.send_static_file(filename)`
   — `src/flask/app.py:392`
3. `send_static_file` calls `self.get_send_file_max_age(filename)` to get cache TTL from `SEND_FILE_MAX_AGE_DEFAULT`
   — `src/flask/app.py:409`
4. Calls `send_from_directory(self.static_folder, filename, max_age=max_age)`
   — `src/flask/app.py:410`, `src/flask/helpers.py:543`
5. `helpers.send_from_directory` calls `_prepare_send_file_kwargs(**kwargs)` to inject app context values (`environ`, `USE_X_SENDFILE`, `response_class`, `_root_path`)
   — `src/flask/helpers.py:582`, `src/flask/helpers.py:402`
6. Delegates to `werkzeug.utils.send_from_directory(directory, path, **kwargs)` — validates the path with `safe_join`, raises 404 if not found
7. Werkzeug's `send_file` reads the file, sets `Content-Type`, `ETag`, `Last-Modified`, `Cache-Control` headers, and returns a `Response` with the file contents or a 304 Not Modified

---

## 8. Error Handling

### HTTP exception (e.g., `abort(404)`)

1. View or middleware calls `abort(404)` — `src/flask/helpers.py:281`
2. `helpers.abort` calls `ctx.app.aborter(404)` (a Werkzeug `Aborter` instance)
   — `src/flask/helpers.py:299`
3. `Aborter.__call__` raises `werkzeug.exceptions.NotFound`
4. Exception propagates up through `Flask.dispatch_request` and is caught in `Flask.full_dispatch_request`
   — `src/flask/app.py:1017`
5. `Flask.full_dispatch_request` calls `Flask.handle_user_exception(ctx, e)`
   — `src/flask/app.py:1018`
6. `Flask.handle_user_exception` detects an `HTTPException` and calls `Flask.handle_http_exception(ctx, e)`
   — `src/flask/app.py:887`
7. `Flask.handle_http_exception` calls `self._find_error_handler(e, request.blueprints)` — searches registered error handlers by exception type and MRO, scoped first to matching blueprint then app-wide
   — `src/flask/app.py:860`
8. If a handler is found, calls it via `self.ensure_sync(handler)(e)` and returns the response value
   — `src/flask/app.py:863`
9. If no handler, returns the exception itself (Werkzeug `HTTPException` is a valid WSGI response)
10. Result is passed to `Flask.finalize_request` which runs `after_request` hooks and saves the session normally

### Unhandled exception (500)

1. Any non-HTTP exception raised in a view is caught in `Flask.full_dispatch_request`
   — `src/flask/app.py:1017`
2. `Flask.handle_user_exception` is called; no matching handler found, so `raise` re-raises
   — `src/flask/app.py:892`
3. Exception propagates to the outer `try` in `Flask.wsgi_app` and is caught there
   — `src/flask/app.py:1598–1600`
4. `Flask.handle_exception(ctx, e)` is called
   — `src/flask/app.py:897`
5. `handle_exception` fires `got_request_exception` signal, then checks `PROPAGATE_EXCEPTIONS` — if true (debug/testing mode), re-raises
   — `src/flask/app.py:926–938`
6. Otherwise calls `self.log_exception(ctx, exc_info)` to log the traceback
   — `src/flask/app.py:940`
7. Creates `InternalServerError(original_exception=e)`, checks for a registered 500 handler, then calls `Flask.finalize_request(ctx, server_error, from_error_handler=True)`
   — `src/flask/app.py:942–948`

---

## 9. `flask run` CLI Command

1. User runs `flask run` in the terminal
2. Click invokes `cli.main()` — `src/flask/__main__.py` calls `cli.main()`
   — `src/flask/cli.py`
3. `FlaskGroup` (Click group subclass) processes `--app` option or `FLASK_APP` env var
4. `ScriptInfo.load_app()` is called; it imports the module and calls `find_best_app(module)`
   — `src/flask/cli.py:41`
5. `find_best_app` looks for `app`, `application` attributes, then any `Flask` instance, then `create_app()` / `make_app()` factory functions
   — `src/flask/cli.py:41–91`
6. The `run_command` Click command is invoked with host/port/debug options
7. `run_command` calls `app.run(host, port, debug, ...)` — `src/flask/app.py:632`
   — or directly calls `werkzeug.serving.run_simple`
8. `Flask.run` calls `cli.load_dotenv()` if enabled, resolves host/port from `SERVER_NAME` config, sets debug mode
   — `src/flask/app.py:709–743`
9. `Flask.run` calls `werkzeug.serving.run_simple(host, port, app, use_reloader=..., use_debugger=..., threaded=True)`
   — `src/flask/app.py:748`
10. Werkzeug's development server loop starts accepting connections; each connection calls `Flask.__call__` (step 1 of Request Handling)
