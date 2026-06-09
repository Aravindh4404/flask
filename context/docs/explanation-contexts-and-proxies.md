# Contexts and Proxies

This document explains why Flask uses context-local proxies (`request`, `g`,
`current_app`, `session`) instead of passing objects through every function call.

---

## The problem

A web framework must handle many requests. During each request, you need access
to:

- The request data (HTTP method, path, headers, body)
- The application (config, logger, extensions)
- Per-request scratch space (data loaded by one function, used by another)
- The user's session

The obvious approach is to pass these as function arguments:

```python
def handle_login(app, request):
    user = load_user(app, request.form["username"])
    ...

def load_user(app, username):
    return app.db.query(User).filter_by(username=username).first()
```

This becomes noise immediately. Every function in your app needs to accept `app`
and `request` — not because they use them, but because something they call might.

A second problem: **circular imports**. The view function in `views.py` needs the
app to call `app.config` or `app.extensions`, but `app.py` imports from
`views.py` to register routes. Import cycles cause `ImportError` at startup.

---

## Flask's solution: the active context

Flask maintains a **current application context** for each running request (and
any CLI command or manual `with app.app_context()` block). The context stores:

- The `Flask` app itself
- A `g` object — per-context scratch space
- The current `Request` (only during a request)
- The current `Session` (only during a request)

The proxies (`request`, `g`, `current_app`, `session`) are Werkzeug `LocalProxy`
objects. When you access `request.method`, the proxy looks up the context stored
in a `contextvars.ContextVar`, retrieves the live `Request` object, and forwards
the attribute access. There is no global state — each OS thread (or async task)
has its own context variable value.

```
Thread A:  [AppContext(app, g, request_A, session_A)]
Thread B:  [AppContext(app, g, request_B, session_B)]
```

Both threads share the same `app` object but each has its own `Request` and
`g`.

---

## What this means in practice

### You can import proxies at module level

```python
# views.py
from flask import request, g, current_app

@app.route("/")
def index():
    # At this point the proxy is resolved — it points to the live request
    return current_app.config["GREETING"]
```

The import of `request` does not resolve the proxy. Resolution happens when you
*access an attribute* on the proxy. This means you can import at the top of
the file without any circular import issue.

### Accessing a proxy outside a context raises RuntimeError

```python
# This fails with RuntimeError: Working outside of application context.
from flask import current_app
print(current_app.config)
```

Fix: push a context first.

```python
with app.app_context():
    print(current_app.config)  # works
```

### `g` is per-context, not per-request

`g` is attached to the **application context**, which is pushed once per request.
Data stored on `g` disappears when the request ends.

```python
@app.before_request
def load_user():
    g.user = db.get_user(session.get("user_id"))

@app.route("/profile")
def profile():
    return f"Hello {g.user.name}"
```

`g` is not thread-safe across contexts — it is safe to use within a single
request because a single worker handles one request at a time.

---

## The lifecycle

```
Request arrives
│
├─ App context pushed
│   ├─ current_app → app
│   └─ g → fresh _AppCtxGlobals()
│
├─ Request context populated
│   ├─ request → Request(environ)
│   └─ session → loaded from cookie
│
├─ before_request hooks run
├─ View function executes
├─ after_request hooks run
├─ Response sent
│
├─ Request context torn down (teardown_request fires)
└─ App context torn down (teardown_appcontext fires)
```

In Flask 3.x, the request context and app context are unified — there is one
`AppContext` object that holds both app data and request data.

---

## When you need `_get_current_object()`

The proxy forwards attribute access transparently, but it is not the object
itself. Code that does identity checks (`x is request`) or that passes the proxy
to a function expecting the actual object may misbehave.

Call `_get_current_object()` to get the real object:

```python
# Pass the real request to a Celery task (tasks run outside the request context)
task.apply_async(args=[request._get_current_object()])
```

---

## Trade-offs

The context-proxy approach trades transparency for ergonomics.

**What you gain:**
- No argument threading — views and helpers stay clean
- No circular imports — proxies are resolved lazily
- Thread safety — each thread/task gets its own context

**What you give up:**
- Implicit dependency on context — a function that uses `request` will fail
  silently if called outside a request context (unless you test for it with
  `has_request_context()`)
- Slight magic — newcomers wonder "where does `request` come from?"

This is the intentional trade-off Flask makes. Django passes `request` through
every view and form explicitly; Flask makes it implicit. Neither is universally
better — Flask's approach fits small-to-medium apps and keeps view code lean.

---

## Related

- [Core API Reference — Context Proxies](reference-core-api.md#context-proxies)
- [App Factory Pattern](explanation-app-factory-pattern.md) — how to avoid the circular import problem for the app itself
