# How to Handle Errors

Flask lets you register handlers for HTTP status codes and exception classes.
This document covers custom error pages, JSON error responses, and catching
unexpected exceptions.

## Prerequisites

- Flask installed (`pip install flask`)
- A Flask app or blueprint

---

## Steps

### 1. Register a handler for an HTTP status code

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.errorhandler(404)
def page_not_found(error):
    return render_template("errors/404.html"), 404

@app.errorhandler(403)
def forbidden(error):
    return render_template("errors/403.html"), 403

@app.errorhandler(500)
def internal_server_error(error):
    return render_template("errors/500.html"), 500
```

The handler receives the `HTTPException` object as `error`. You can read
`error.description` for the default error message.

### 2. Return JSON errors for API routes

If your app serves both HTML and JSON, check the `Accept` header to decide
which format to return:

```python
from flask import jsonify, request
from werkzeug.exceptions import HTTPException

@app.errorhandler(HTTPException)
def handle_http_exception(error):
    if request.accept_mimetypes.best_match(["application/json", "text/html"]) == "application/json":
        return jsonify(code=error.code, name=error.name, description=error.description), error.code
    return render_template(f"errors/{error.code}.html"), error.code
```

A single `HTTPException` handler catches all HTTP errors (400, 404, 405, 500, …).

### 3. Handle a specific exception class

```python
class InsufficientFundsError(Exception):
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount

@app.errorhandler(InsufficientFundsError)
def handle_insufficient_funds(error):
    return jsonify(
        error="insufficient_funds",
        balance=error.balance,
        needed=error.amount,
    ), 402
```

Any view function that raises `InsufficientFundsError` will be caught here.

### 4. Register handlers on a blueprint

Handlers registered on a blueprint apply only to routes in that blueprint:

```python
from flask import Blueprint

bp = Blueprint("api", __name__, url_prefix="/api")

@bp.errorhandler(404)
def api_not_found(error):
    return jsonify(error="not_found"), 404
```

App-level handlers are a fallback for blueprints that don't define their own.

### 5. Raise HTTP errors in view functions

Use `abort()` to trigger an error handler immediately:

```python
from flask import abort

@app.route("/admin")
def admin():
    if not g.user or not g.user.is_admin:
        abort(403)          # triggers the 403 handler
    return render_template("admin.html")
```

`abort()` raises a `werkzeug.exceptions.HTTPException` subclass, which is caught
by the matching error handler.

### 6. Log unexpected exceptions

For 500 errors in production, log the traceback before rendering the response:

```python
import logging

@app.errorhandler(500)
def server_error(error):
    app.logger.exception("Unhandled exception")
    return render_template("errors/500.html"), 500
```

Flask's built-in error handler already logs unhandled exceptions in debug mode.
The custom handler lets you add structured logging or alerting in production.

---

## Verification

Test your handlers with `app.test_client()`:

```python
def test_404(client):
    response = client.get("/nonexistent-path")
    assert response.status_code == 404
    assert b"Page Not Found" in response.data

def test_json_404(client):
    response = client.get("/nonexistent-path", headers={"Accept": "application/json"})
    assert response.status_code == 404
    assert response.json["code"] == 404
```

---

## Troubleshooting

**Handler not called for 500 errors during testing**

By default, `TESTING=True` re-raises exceptions instead of running the 500
handler. This is intentional — you want the full traceback in tests, not the
error page. Test the 500 handler by temporarily setting `PROPAGATE_EXCEPTIONS`
to `False`:

```python
app.config["PROPAGATE_EXCEPTIONS"] = False
```

**Blueprint error handler not called for app-level routes**

Blueprint handlers only apply to routes registered on that blueprint. Register
the handler on the app for it to apply globally.

**`abort(404)` renders a blank page instead of your template**

The handler's return value must be a tuple of `(response_or_string, status_code)`.
Returning just `render_template("404.html")` without the status code returns a
200 response — the browser sees "200 OK" with error page content. Always include
the status code: `return render_template("404.html"), 404`.

---

## Related

- [Core API Reference — Error handlers](reference-core-api.md#error-handlers)
- [How to write tests](howto-testing.md)
