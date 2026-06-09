# How to Write Tests

Flask provides a test client and a CLI runner for testing your application
without starting a real server. This guide uses pytest.

## Prerequisites

- Flask installed
- pytest installed: `pip install pytest`
- An app using the [app factory pattern](explanation-app-factory-pattern.md) is
  strongly recommended, but not strictly required

---

## Steps

### 1. Create a test configuration and fixtures

```python
# tests/conftest.py
import pytest
from yourapp import create_app

@pytest.fixture
def app():
    app = create_app({
        "TESTING": True,
        "SECRET_KEY": "test-secret",
        "DATABASE": "sqlite:///:memory:",
    })

    with app.app_context():
        from yourapp.db import init_db
        init_db()

    yield app

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def runner(app):
    return app.test_cli_runner()
```

The `app` fixture creates a fresh app for each test. `client` is a
`FlaskClient` for HTTP requests. `runner` is for CLI commands.

### 2. Test a GET route

```python
# tests/test_main.py

def test_index(client):
    response = client.get("/")
    assert response.status_code == 200
    assert b"Hello" in response.data
```

`response.data` is the raw response body as `bytes`. Use `response.get_data(as_text=True)`
for a string.

### 3. Test a POST route with form data

```python
def test_login(client):
    response = client.post("/auth/login", data={
        "username": "alice",
        "password": "secret",
    })
    assert response.status_code == 302  # redirect on success
    assert response.headers["Location"] == "/"
```

### 4. Test a JSON API endpoint

```python
def test_create_item(client):
    response = client.post(
        "/api/items",
        json={"name": "Widget", "price": 9.99},
    )
    assert response.status_code == 201
    data = response.get_json()
    assert data["name"] == "Widget"
```

Passing `json=` sets `Content-Type: application/json` automatically.

### 5. Test with session data

Use `client.session_transaction()` to read or write session data directly:

```python
def test_profile_requires_login(client):
    response = client.get("/profile")
    assert response.status_code == 302  # redirected to login

def test_profile_with_session(client):
    with client.session_transaction() as sess:
        sess["user_id"] = 1
    response = client.get("/profile")
    assert response.status_code == 200
```

### 6. Test URL building with `url_for`

Outside a request, `url_for` requires an active request context. Use
`app.test_request_context`:

```python
def test_url_for(app):
    with app.test_request_context():
        from flask import url_for
        assert url_for("main.index") == "/"
        assert url_for("auth.login") == "/auth/login"
```

### 7. Test CLI commands

```python
def test_init_db_command(runner):
    result = runner.invoke(args=["init-db"])
    assert result.exit_code == 0
    assert "Initialized" in result.output
```

The `runner.invoke(args=["command-name"])` runs a registered `flask` CLI command.

### 8. Test file uploads

```python
import io

def test_upload(client):
    data = {
        "file": (io.BytesIO(b"file contents"), "test.txt"),
    }
    response = client.post(
        "/upload",
        data=data,
        content_type="multipart/form-data",
    )
    assert response.status_code == 200
```

---

## Verification

Run the full test suite:

```
pytest -v
```

All tests should pass. If a test fails with `RuntimeError: Working outside of
application context`, wrap the failing code with `with app.app_context():` or
use the `app` fixture.

---

## Troubleshooting

**`RuntimeError: Working outside of application context`**

A helper function calls `current_app`, `g`, or `url_for` outside a pushed
context. Either:
- Add `with app.app_context():` in the fixture or test, or
- Check that you're using the `app` fixture and calling the code inside a view
  (which has a context automatically)

**Session data not persisting between requests in tests**

`FlaskClient` preserves cookies between requests by default (`use_cookies=True`).
If sessions are not persisting, check that `SECRET_KEY` is set in your test
config. An empty or `None` secret key causes session signing to fail silently.

**`response.data` is empty for streaming responses**

Streaming responses (`stream_with_context`, `Response(generate())`) send data
lazily. Force eager evaluation in tests:

```python
response = client.get("/stream")
data = b"".join(response.response)
```

**`PROPAGATE_EXCEPTIONS` and error handler testing**

With `TESTING=True`, exceptions raised in view functions propagate to the test
instead of being caught by error handlers. To test your 500 handler specifically:

```python
app.config["PROPAGATE_EXCEPTIONS"] = False
response = client.get("/route-that-raises")
assert response.status_code == 500
```

---

## Related

- [Core API Reference — Testing](reference-core-api.md#testing)
- [App Factory Pattern](explanation-app-factory-pattern.md)
- [How to handle errors](howto-error-handling.md)
