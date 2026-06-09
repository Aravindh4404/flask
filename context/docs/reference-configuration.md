# Configuration Reference

Flask's configuration is a dict subclass (`flask.Config`) accessible as
`app.config`. Only uppercase keys are treated as configuration. Keys defined here
are built-in Flask keys; extensions and your application can add their own.

## Loading configuration

```python
# From a Python object or module
app.config.from_object("yourapp.config.ProductionConfig")

# From a .py file (relative to root_path)
app.config.from_pyfile("config.cfg")

# From environment variables prefixed with FLASK_
app.config.from_prefixed_env()
# FLASK_SECRET_KEY=abc  →  app.config["SECRET_KEY"] = "abc"
# FLASK_DB__HOST=localhost  →  app.config["DB"]["HOST"] = "localhost"

# From a file path in an environment variable
app.config.from_envvar("YOURAPP_SETTINGS")

# Direct assignment
app.config["SECRET_KEY"] = "dev"
app.config.update(SECRET_KEY="dev", TESTING=True)
```

## Built-in configuration keys

### Core

| Key | Type | Default | Description |
|---|---|---|---|
| `DEBUG` | `bool \| None` | `None` | Enable debug mode. Overrides to `True` when set via `FLASK_DEBUG` env var. Never enable in production |
| `TESTING` | `bool` | `False` | Enable test mode. Exceptions propagate instead of being caught by error handlers |
| `SECRET_KEY` | `str \| bytes \| None` | `None` | HMAC signing key for session cookies and other cryptographic features. **Required for sessions.** Use a long random value |
| `SECRET_KEY_FALLBACKS` | `list[str \| bytes] \| None` | `None` | Previous secret keys still accepted for verification (key rotation) |
| `PROPAGATE_EXCEPTIONS` | `bool \| None` | `None` | Re-raise exceptions instead of catching them. Defaults to `True` when `TESTING` or `DEBUG` is `True` |
| `TRUSTED_HOSTS` | `list[str] \| None` | `None` | Allowed values for the `Host` header. Requests with untrusted hosts are rejected with 400 |

### Server

| Key | Type | Default | Description |
|---|---|---|---|
| `SERVER_NAME` | `str \| None` | `None` | Host and port the app is served at (e.g. `"example.com"` or `"localhost:8000"`). Required for `url_for(_external=True)` outside of request context |
| `APPLICATION_ROOT` | `str` | `"/"` | URL prefix the app is mounted at behind a proxy (e.g. `"/myapp"`) |
| `PREFERRED_URL_SCHEME` | `str` | `"http"` | Scheme used by `url_for(_external=True)` outside of a request context |

### Sessions

| Key | Type | Default | Description |
|---|---|---|---|
| `PERMANENT_SESSION_LIFETIME` | `timedelta \| int` | `timedelta(days=31)` | Expiry of permanent sessions. `int` is treated as seconds |
| `SESSION_COOKIE_NAME` | `str` | `"session"` | Name of the session cookie |
| `SESSION_COOKIE_DOMAIN` | `str \| None` | `None` | Domain for the session cookie. `None` uses the request domain |
| `SESSION_COOKIE_PATH` | `str \| None` | `None` | Path for the session cookie. `None` uses `APPLICATION_ROOT` |
| `SESSION_COOKIE_HTTPONLY` | `bool` | `True` | Prevent JavaScript access to the session cookie |
| `SESSION_COOKIE_SECURE` | `bool` | `False` | Only send the session cookie over HTTPS. **Enable in production** |
| `SESSION_COOKIE_SAMESITE` | `str \| None` | `None` | `"Strict"`, `"Lax"`, or `"None"`. Protects against CSRF |
| `SESSION_COOKIE_PARTITIONED` | `bool` | `False` | Set the `Partitioned` attribute (CHIPS). Chrome privacy sandbox requirement |
| `SESSION_REFRESH_EACH_REQUEST` | `bool` | `True` | Reset cookie expiry on every request. Disable to only refresh when session data changes |

### File serving

| Key | Type | Default | Description |
|---|---|---|---|
| `USE_X_SENDFILE` | `bool` | `False` | Use `X-Sendfile` header for `send_file`. Let nginx/Apache serve the file. Server must support it |
| `SEND_FILE_MAX_AGE_DEFAULT` | `timedelta \| int \| None` | `None` | Cache control `max-age` for `send_file`. `None` means browsers use conditional requests (usually better) |
| `MAX_CONTENT_LENGTH` | `int \| None` | `None` | Max allowed request body size in bytes. `None` is unlimited. Clients exceeding this get a 413 error |

### Forms

| Key | Type | Default | Description |
|---|---|---|---|
| `MAX_FORM_MEMORY_SIZE` | `int` | `500_000` | Max total size (in bytes) of non-file form data |
| `MAX_FORM_PARTS` | `int` | `1_000` | Max number of fields in a multipart form |

### Debugging

| Key | Type | Default | Description |
|---|---|---|---|
| `TRAP_BAD_REQUEST_ERRORS` | `bool \| None` | `None` | Raise `BadRequest` exceptions instead of returning a 400 response. Defaults to `True` in debug mode |
| `TRAP_HTTP_EXCEPTIONS` | `bool` | `False` | Raise all `HTTPException` subclasses instead of returning their default response |
| `EXPLAIN_TEMPLATE_LOADING` | `bool` | `False` | Log detailed information about how templates are located. Useful when templates are not found |

### Templates

| Key | Type | Default | Description |
|---|---|---|---|
| `TEMPLATES_AUTO_RELOAD` | `bool \| None` | `None` | Reload templates when they change on disk. Defaults to `True` in debug mode |

### Cookies

| Key | Type | Default | Description |
|---|---|---|---|
| `MAX_COOKIE_SIZE` | `int` | `4093` | Warn when a cookie exceeds this size in bytes. Most browsers reject cookies over 4KB. Set to `0` to disable |
| `PROVIDE_AUTOMATIC_OPTIONS` | `bool` | `True` | Respond to `OPTIONS` requests automatically with an `Allow` header listing accepted methods |

---

## Patterns

### Class-based configuration

```python
class Config:
    TESTING = False
    SECRET_KEY = "dev"

class ProductionConfig(Config):
    SECRET_KEY = "prod-secret"  # use secrets.token_hex(32) in reality
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_SAMESITE = "Lax"

app.config.from_object(ProductionConfig)
```

### Instance folder (secrets outside the repo)

```python
app = Flask(__name__, instance_relative_config=True)
app.config.from_pyfile("config.cfg", silent=True)
```

Create `instance/config.cfg`:
```python
SECRET_KEY = "actual-secret"
DATABASE_URI = "postgresql://..."
```

The `instance/` folder is outside the package and should not be committed to git.

### Environment variables

```python
app.config.from_prefixed_env()
```

Set in the shell:
```
FLASK_SECRET_KEY=abc123 flask run
FLASK_DEBUG=1 flask run
```

Nested keys use double-underscore:
`FLASK_DB__HOST=localhost` sets `app.config["DB"]["HOST"] = "localhost"`.

---

## Related

- [Core API Reference](reference-core-api.md)
- [App Factory Pattern](explanation-app-factory-pattern.md) — recommended way to create the app and configure it
