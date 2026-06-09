# How to Organize Your App with Blueprints

Blueprints let you split a Flask application into reusable modules — each with
its own routes, error handlers, templates, and static files — and register them
all on a single `Flask` app.

## Prerequisites

- Flask installed (`pip install flask`)
- Basic familiarity with Flask routes and the `Flask` object

---

## Steps

### 1. Create a blueprint

In each module that owns a slice of your app, create a `Blueprint` instead of
using `app` directly.

```python
# auth/views.py
from flask import Blueprint, request, redirect, url_for, session, g

bp = Blueprint(
    "auth",          # name — used as endpoint prefix: "auth.login"
    __name__,        # import name — used to locate template/static folders
    url_prefix="/auth",
)

@bp.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        session["user_id"] = authenticate(request.form)
        return redirect(url_for("main.index"))
    return render_template("auth/login.html")

@bp.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("auth.login"))
```

### 2. Register the blueprint on the app

```python
# app.py
from flask import Flask
from auth.views import bp as auth_bp
from main.views import bp as main_bp

def create_app():
    app = Flask(__name__)
    app.config["SECRET_KEY"] = "dev"

    app.register_blueprint(auth_bp)
    app.register_blueprint(main_bp)

    return app
```

You can override `url_prefix` at registration time without touching the blueprint:

```python
app.register_blueprint(auth_bp, url_prefix="/v2/auth")
```

### 3. Reference blueprint endpoints with the `blueprint.view` prefix

All endpoints defined inside a blueprint are namespaced. The format is
`"blueprint_name.view_function_name"`:

```python
url_for("auth.login")    # "/auth/login"
url_for("auth.logout")   # "/auth/logout"
url_for("main.index")    # "/"
```

In Jinja2 templates:

```html
<a href="{{ url_for('auth.logout') }}">Log out</a>
```

### 4. Add blueprint-scoped lifecycle hooks

Hooks on a blueprint run only for requests handled by that blueprint:

```python
@bp.before_request
def require_login():
    if g.user is None:
        return redirect(url_for("auth.login"))
```

```python
@bp.after_request
def add_header(response):
    response.headers["X-Content-Type-Options"] = "nosniff"
    return response
```

### 5. Serve blueprint-specific templates and static files

Give the blueprint its own `templates` and `static` folders:

```python
bp = Blueprint(
    "admin",
    __name__,
    template_folder="templates",   # admin/templates/
    static_folder="static",        # admin/static/
    static_url_path="/admin/static",
)
```

Flask will look for templates in the blueprint's `templates/` folder, falling
back to the app's `templates/` folder if not found. Name blueprint templates
with a subfolder to avoid collisions:

```
admin/templates/admin/dashboard.html  →  render_template("admin/dashboard.html")
```

### 6. Add blueprint CLI commands

```python
import click

@bp.cli.command("sync-users")
@click.option("--dry-run", is_flag=True)
def sync_users(dry_run):
    """Sync users from the external directory."""
    ...
```

Run with: `flask auth sync-users --dry-run`

---

## Verification

After registering blueprints, list all routes to confirm they are registered:

```
flask --app app routes
```

Expected output includes prefixed routes like `/auth/login`, `/auth/logout`.

---

## Troubleshooting

**`werkzeug.routing.BuildError: Could not build url for endpoint 'login'`**

You used `url_for("login")` but the view is in a blueprint. Use the namespaced
form: `url_for("auth.login")`.

**`TemplateNotFound: auth/login.html`**

The blueprint's `template_folder` is set but no `auth/` subfolder exists inside
it, or the blueprint was not given a `template_folder` at all. Check that the
path `auth/templates/auth/login.html` (or `templates/auth/login.html` in the
app root) exists.

**Circular import at startup**

If `app.py` imports the blueprint and the blueprint imports `app`, you have a
circular dependency. Fix: use the [app factory pattern](explanation-app-factory-pattern.md)
and import blueprints inside `create_app()`, not at module level.

---

## Related

- [App Factory Pattern](explanation-app-factory-pattern.md)
- [Core API Reference — Blueprint](reference-core-api.md#blueprint)
- [How to write tests](howto-testing.md)
