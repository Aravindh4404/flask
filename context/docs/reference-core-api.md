# Core API Reference

This document covers every class, function, and proxy that Flask exports from
`flask`. All names shown here are importable directly from `flask`.

---

## Flask Application

### `Flask(import_name, **options)`

The central object. Implements a WSGI application and acts as a registry for
routes, error handlers, template configuration, and extensions.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `import_name` | `str` | required | Name of the application package, usually `__name__` |
| `static_folder` | `str \| PathLike \| None` | `'static'` | Folder served at `static_url_path`. Set to `None` to disable |
| `static_url_path` | `str \| None` | `'/<static_folder>'` | URL prefix for static files |
| `static_host` | `str \| None` | `None` | Required when using `host_matching=True` with a static folder |
| `host_matching` | `bool` | `False` | Enable host-based routing |
| `subdomain_matching` | `bool` | `False` | Match routes by subdomain relative to `SERVER_NAME` |
| `template_folder` | `str \| None` | `'templates'` | Folder containing Jinja2 templates |
| `instance_path` | `str \| None` | `<root>/instance/` | Path to the instance folder |
| `instance_relative_config` | `bool` | `False` | Treat relative config paths as relative to `instance_path` |
| `root_path` | `str \| None` | auto-detected | Override the detected root path. Set only for namespace packages |

**Usage**

```python
# Single module — __name__ is always correct
app = Flask(__name__)

# Package (yourapplication/__init__.py) — hardcode the package name
app = Flask("yourapplication")
```

**Key instance attributes**

| Attribute | Type | Description |
|---|---|---|
| `app.config` | `Config` | The app's configuration dictionary |
| `app.debug` | `bool` | Whether debug mode is active |
| `app.testing` | `bool` | Whether test mode is active |
| `app.secret_key` | `str \| bytes \| None` | HMAC key used to sign session cookies |
| `app.url_map` | `werkzeug.routing.Map` | The Werkzeug URL routing map |
| `app.extensions` | `dict` | Dictionary where extensions register state |
| `app.cli` | `click.Group` | Click command group for `flask` CLI commands |
| `app.json` | `JSONProvider` | JSON encoder/decoder |

---

### Routing

**`@app.route(rule, **options)`**

Decorator that registers a view function for a URL rule.

```python
@app.route("/users/<int:user_id>", methods=["GET", "POST"])
def user(user_id):
    ...
```

**`@app.get(rule)`** / **`@app.post(rule)`** / **`@app.put(rule)`** / **`@app.delete(rule)`** / **`@app.patch(rule)`**

Shorthand decorators for single HTTP methods.

```python
@app.get("/users/<int:user_id>")
def get_user(user_id):
    ...
```

**`app.add_url_rule(rule, endpoint, view_func, **options)`**

Programmatic equivalent of `@app.route`. Use this when you can't use the decorator
(for example, when registering class-based views).

```python
app.add_url_rule("/hello", view_func=HelloView.as_view("hello"))
```

**URL variable converters**

| Converter | Example | Matches |
|---|---|---|
| `string` | `<name>` (default) | Any text without a slash |
| `int` | `<int:id>` | Positive integers |
| `float` | `<float:value>` | Positive floats |
| `path` | `<path:subpath>` | Like `string` but accepts slashes |
| `uuid` | `<uuid:token>` | UUID strings |

---

### Request lifecycle hooks

**`@app.before_request`**

Runs before every request. Return a response to short-circuit the view.

```python
@app.before_request
def check_auth():
    if not request.headers.get("Authorization"):
        return "Unauthorized", 401
```

**`@app.after_request`**

Runs after every successful request. Must accept and return a `Response` object.

```python
@app.after_request
def add_cors_header(response):
    response.headers["Access-Control-Allow-Origin"] = "*"
    return response
```

**`@app.teardown_request`**

Runs after every request, even if an exception was raised. Receives the exception
or `None`. Return value is ignored.

```python
@app.teardown_request
def close_db(exc):
    db = g.pop("db", None)
    if db is not None:
        db.close()
```

**`@app.teardown_appcontext`**

Runs when the application context is popped. Used for cleanup that should happen
at the end of every context (request or explicit `with app.app_context()`).

---

### Error handlers

**`@app.errorhandler(code_or_exception)`**

Register a function to handle an HTTP status code or exception class.

```python
@app.errorhandler(404)
def not_found(error):
    return render_template("404.html"), 404

@app.errorhandler(ValueError)
def handle_value_error(exc):
    return jsonify(error=str(exc)), 400
```

**`app.register_error_handler(code_or_exception, f)`**

Programmatic equivalent of `@app.errorhandler`.

---

### Context management

**`app.app_context()`**

Returns a context manager that pushes an application context. Required before
using `current_app`, `g`, or config outside of a request.

```python
with app.app_context():
    db.create_all()
```

**`app.test_request_context(path, **kwargs)`**

Returns a context manager that pushes a request context with a fake request.
Used in tests and the Flask shell.

```python
with app.test_request_context("/"):
    assert url_for("index") == "/"
```

---

### Testing

**`app.test_client(use_cookies=True)`**

Returns a `FlaskClient` for making requests against the app without a running
server.

```python
client = app.test_client()
response = client.get("/")
assert response.status_code == 200
```

**`app.test_cli_runner(mix_stderr=True)`**

Returns a `FlaskCliRunner` for invoking CLI commands in tests.

---

### Template rendering (app-level)

**`app.jinja_env`**

The Jinja2 `Environment`. Access to add custom filters, globals, and tests.

```python
app.jinja_env.filters["datetimeformat"] = format_datetime
```

**`@app.template_filter(name)`** / **`@app.template_global(name)`** / **`@app.template_test(name)`**

Register a Jinja2 filter, global, or test callable.

```python
@app.template_filter("reverse")
def reverse_filter(s):
    return s[::-1]
```

---

## Blueprint

### `Blueprint(name, import_name, **options)`

A collection of routes, error handlers, and other app features that can be
registered on an app one or more times.

```python
from flask import Blueprint

bp = Blueprint("auth", __name__, url_prefix="/auth")

@bp.route("/login", methods=["GET", "POST"])
def login():
    ...
```

**Parameters** (same as `Flask` except):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | required | Name identifying the blueprint and its endpoints |
| `url_prefix` | `str \| None` | `None` | URL prefix prepended to all routes on this blueprint |
| `subdomain` | `str \| None` | `None` | Subdomain matched for routes on this blueprint |
| `url_defaults` | `dict \| None` | `None` | Default values for URL variables |
| `cli_group` | `str \| None` | blueprint name | Name of the CLI group; `None` merges commands into the root group |

**Registering a blueprint**

```python
app.register_blueprint(bp)
app.register_blueprint(bp, url_prefix="/v2/auth")  # override at registration time
```

All routing and lifecycle decorators available on `Flask` are also available
on `Blueprint` with the same signatures.

---

## Context Proxies

These are context-local proxies. They point to objects that are active for the
current request or app context. See [Contexts and Proxies](explanation-contexts-and-proxies.md).

### `request`

The current HTTP request. Type: `flask.Request` (subclass of `werkzeug.Request`).

```python
from flask import request

@app.route("/submit", methods=["POST"])
def submit():
    name = request.form["name"]
    data = request.json         # dict from JSON body
    q = request.args.get("q")  # query string ?q=...
    ua = request.headers["User-Agent"]
    f = request.files["upload"]
    return name
```

**Key attributes**

| Attribute | Type | Description |
|---|---|---|
| `request.method` | `str` | HTTP method (`"GET"`, `"POST"`, …) |
| `request.path` | `str` | URL path (e.g. `"/users/42"`) |
| `request.args` | `ImmutableMultiDict` | Query string parameters |
| `request.form` | `ImmutableMultiDict` | POST form data |
| `request.json` | `Any \| None` | Parsed JSON body (requires `Content-Type: application/json`) |
| `request.data` | `bytes` | Raw request body |
| `request.files` | `ImmutableMultiDict` | Uploaded files as `FileStorage` objects |
| `request.headers` | `Headers` | Request headers (case-insensitive) |
| `request.cookies` | `ImmutableMultiDict` | Request cookies |
| `request.remote_addr` | `str \| None` | Client IP address |
| `request.url` | `str` | Full request URL |

---

### `g`

A namespace for data that lives for the duration of the current application
context (effectively: one request). Use it to store per-request state that
multiple functions need.

```python
from flask import g

@app.before_request
def load_user():
    g.user = db.get_user(session.get("user_id"))

@app.route("/profile")
def profile():
    return render_template("profile.html", user=g.user)
```

`g` is reset at the start of each request. Data stored on `g` does not persist.

---

### `session`

The current user's session. Backed by a signed cookie (using `SECRET_KEY`).
Behaves like a dict.

```python
from flask import session

@app.route("/login", methods=["POST"])
def login():
    session["user_id"] = verify_credentials(request.form)
    return redirect(url_for("index"))

@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("index"))
```

`session.permanent = True` extends the lifetime to `PERMANENT_SESSION_LIFETIME`.

---

### `current_app`

A proxy to the Flask application handling the current request (or app context).
Use this inside view functions, helper modules, and extensions to access `app`
without importing it directly.

```python
from flask import current_app

def send_email(to, subject, body):
    smtp_host = current_app.config["SMTP_HOST"]
    ...
```

---

## Helper Functions

### `url_for(endpoint, **values)`

Build a URL for a given endpoint. The `endpoint` is the function name (or
`"blueprint_name.function_name"` for blueprint views).

```python
url_for("index")                    # "/"
url_for("user", user_id=42)         # "/users/42"
url_for("static", filename="app.js")  # "/static/app.js"
url_for("index", _external=True)    # "http://example.com/"
```

Pass extra keyword arguments to add query string parameters:
`url_for("search", q="flask")` → `"/search?q=flask"`

---

### `redirect(location, code=302, Response=None)`

Return a redirect response.

```python
return redirect(url_for("index"))
return redirect("https://example.com", code=301)
```

---

### `abort(status, *args, **kwargs)`

Raise an HTTP exception to immediately stop processing and return an error
response. Triggers any registered error handler for that status code.

```python
abort(404)
abort(403, "You don't have permission.")
```

---

### `make_response(*args)`

Construct a `Response` object. Use when you need to set headers or cookies that
you can't set through a plain return value.

```python
response = make_response(render_template("index.html"), 200)
response.headers["X-Custom"] = "value"
response.set_cookie("session_id", "abc123")
return response
```

---

### `render_template(template_name_or_list, **context)`

Render a Jinja2 template from the `templates` folder. Pass keyword arguments
as template variables.

```python
return render_template("users/profile.html", user=user, posts=posts)
```

---

### `render_template_string(source, **context)`

Render a Jinja2 template from a string. Useful for testing or dynamic templates.

---

### `stream_template(template_name_or_list, **context)`

Like `render_template` but streams the response in chunks. Useful for large
templates where you want the browser to start rendering before the full response
is ready.

---

### `jsonify(*args, **kwargs)`

Create a JSON response. Accepts the same arguments as `json.dumps` — either a
single object or keyword arguments that are assembled into a dict.

```python
return jsonify({"user": user.id, "name": user.name})
return jsonify(users=[u.to_dict() for u in users])
```

Sets `Content-Type: application/json` automatically.

---

### `flash(message, category='message')`

Store a message for the next request. Messages are retrieved by calling
`get_flashed_messages()` in a template.

```python
flash("Login successful.", "success")
flash("Invalid password.", "error")
return redirect(url_for("index"))
```

In a Jinja2 template:

```html
{% for category, message in get_flashed_messages(with_categories=True) %}
  <div class="alert alert-{{ category }}">{{ message }}</div>
{% endfor %}
```

---

### `send_file(path_or_file, mimetype=None, as_attachment=False, download_name=None, ...)`

Send a file to the client.

```python
return send_file("reports/jan.pdf", as_attachment=True)
return send_file(io.BytesIO(pdf_bytes), mimetype="application/pdf")
```

---

### `send_from_directory(directory, path, **kwargs)`

Safely send a file from a directory. Prevents path traversal attacks by
refusing to serve paths that would escape `directory`.

```python
return send_from_directory("uploads", filename)
```

---

### `stream_with_context(generator_or_function)`

Wrap a response generator so it runs inside the current request context, keeping
`request`, `session`, and `g` accessible even after the view function returns.

```python
@app.get("/stream")
def streamed():
    @stream_with_context
    def generate():
        yield "chunk1"
        yield request.args["name"]  # request is still available
    return Response(generate())
```

---

### `get_flashed_messages(with_categories=False, category_filter=())`

Return and remove messages stored by `flash()`. Usually called in templates.

---

### `has_request_context()` / `has_app_context()`

Return `True` if a request context or app context is currently active.
Use to write code that works both inside and outside of a request.

---

## Signals

Flask emits signals (via [blinker](https://blinker.readthedocs.io/)) at key
points in the request lifecycle. Subscribe with `.connect(receiver)`.

| Signal | Sent when | Sender | Extra kwargs |
|---|---|---|---|
| `request_started` | Request context pushed, before dispatch | `app` | — |
| `request_finished` | After successful dispatch, before response sent | `app` | `response` |
| `request_tearing_down` | Request context torn down | `app` | `exc` |
| `got_request_exception` | An exception is raised during dispatch | `app` | `exception` |
| `appcontext_pushed` | App context pushed | `app` | — |
| `appcontext_popped` | App context popped | `app` | `exc` |
| `appcontext_tearing_down` | App context being torn down | `app` | `exc` |
| `before_render_template` | Before a template is rendered | `app` | `template`, `context` |
| `template_rendered` | After a template is rendered | `app` | `template`, `context` |
| `message_flashed` | A message is flashed | `app` | `message`, `category` |

```python
from flask import request_finished

def log_response(sender, response, **extra):
    sender.logger.info(f"{response.status_code} {request.path}")

request_finished.connect(log_response, app)
```

---

## Class-Based Views

### `View`

Base class for class-based views. Override `dispatch_request` to handle all
methods, or set `methods` to restrict which HTTP methods the view accepts.

```python
from flask.views import View

class UserView(View):
    methods = ["GET"]

    def dispatch_request(self, user_id):
        user = db.get_or_404(User, user_id)
        return render_template("user.html", user=user)

app.add_url_rule("/users/<int:user_id>", view_func=UserView.as_view("user"))
```

**Class attributes**

| Attribute | Type | Default | Description |
|---|---|---|---|
| `methods` | `Collection[str] \| None` | `None` | HTTP methods this view accepts |
| `decorators` | `list[Callable]` | `[]` | Decorators applied to the generated view function |
| `init_every_request` | `bool` | `True` | Create a new view instance per request. Set to `False` for efficiency if `self` is read-only |

### `MethodView`

Subclass of `View` that routes HTTP methods to same-named methods (`get`, `post`,
`put`, `delete`, `patch`).

```python
from flask.views import MethodView

class ItemAPI(MethodView):
    def get(self, item_id):
        return jsonify(db.get_or_404(Item, item_id).to_dict())

    def put(self, item_id):
        item = db.get_or_404(Item, item_id)
        item.update(request.json)
        db.session.commit()
        return jsonify(item.to_dict())

    def delete(self, item_id):
        item = db.get_or_404(Item, item_id)
        db.session.delete(item)
        db.session.commit()
        return "", 204

app.add_url_rule("/items/<int:item_id>", view_func=ItemAPI.as_view("item_api"))
```

---

## Related

- [Configuration Reference](reference-configuration.md) — every config key
- [Contexts and Proxies](explanation-contexts-and-proxies.md) — how `request`, `g`, `current_app` work
- [How to organize with Blueprints](howto-blueprints.md)
- [How to write tests](howto-testing.md)
