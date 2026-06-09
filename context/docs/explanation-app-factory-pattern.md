# The App Factory Pattern

This document explains why you should create the Flask app inside a function
(an "app factory") rather than at module level, and when the pattern is necessary.

---

## The direct approach — and its limits

The simplest Flask app creates the `Flask` instance at module level:

```python
# app.py
from flask import Flask

app = Flask(__name__)
app.config["SECRET_KEY"] = "dev"

@app.route("/")
def index():
    return "Hello!"
```

This works for small scripts. It breaks down in three situations:

### 1. Testing with different configurations

You want one configuration for tests (in-memory database, `TESTING=True`) and a
different one for production. With a module-level `app`, you have one global
instance and no clean way to reconfigure it between tests. Running one test
modifies the `app` object that the next test sees.

### 2. Circular imports in larger apps

```
app.py imports views.py (to register routes)
views.py imports app.py (to get `app` for `@app.route`)
```

Python resolves circular imports only when the module is fully loaded. Depending
on import order, `app` might not exist yet when `views.py` imports it, causing
`ImportError` or `AttributeError`.

### 3. Multiple app instances

If you want to run two Flask apps in the same process (multi-tenant, API +
admin panel), a single module-level `app` is a shared global — both instances
collide.

---

## The factory function

Move app creation into a function:

```python
# app.py
from flask import Flask

def create_app(config_object=None):
    app = Flask(__name__)

    # Defaults
    app.config.from_mapping(
        SECRET_KEY="dev",
        DATABASE="sqlite:///app.db",
    )

    # Override with provided config
    if config_object is not None:
        app.config.from_object(config_object)

    # Also load from instance/config.py if it exists
    app.config.from_pyfile("config.py", silent=True)

    # Register extensions
    from .extensions import db
    db.init_app(app)

    # Register blueprints
    from .auth import bp as auth_bp
    from .main import bp as main_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(main_bp)

    return app
```

The `flask` CLI discovers the factory automatically if it's named `create_app`
or `make_app`:

```
flask --app app run
```

---

## Testing with the factory

Each test creates its own `app` instance with a test config:

```python
# tests/conftest.py
import pytest
from yourapp import create_app

@pytest.fixture
def app():
    app = create_app({
        "TESTING": True,
        "DATABASE": "sqlite:///:memory:",
    })
    with app.app_context():
        init_db()
    yield app

@pytest.fixture
def client(app):
    return app.test_client()
```

Each test function gets a fresh app and database. Configuration never leaks
between tests.

---

## Breaking circular imports

With the factory pattern, `views.py` no longer imports `app`. Instead, it
creates a `Blueprint` and uses `current_app` from Flask's proxy system:

```python
# auth/views.py
from flask import Blueprint, current_app, request

bp = Blueprint("auth", __name__)

@bp.route("/login", methods=["POST"])
def login():
    key = current_app.config["SECRET_KEY"]  # proxy — no import needed
    ...
```

`current_app` is resolved at request time, not import time. No circular
dependency.

---

## Extension initialization with `init_app`

Flask extensions support the factory pattern via `init_app(app)`:

```python
# extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager

db = SQLAlchemy()
login_manager = LoginManager()
```

```python
# create_app in app.py
from .extensions import db, login_manager

def create_app(config=None):
    app = Flask(__name__)
    ...
    db.init_app(app)
    login_manager.init_app(app)
    return app
```

The extension object (`db`) is created once, then attached to whichever app
instances are created. Each app gets its own extension state. This is why
well-written extensions always document `init_app` support.

---

## Project layout with the factory

```
yourapp/
├── __init__.py        ← create_app lives here
├── extensions.py      ← extension objects (no app yet)
├── auth/
│   ├── __init__.py    ← Blueprint definition
│   └── views.py
├── main/
│   ├── __init__.py
│   └── views.py
└── models.py
instance/
└── config.py          ← secrets, not in git
tests/
└── conftest.py
```

---

## When you don't need the factory

If your app is a single file, a quick experiment, or a tool that will never
be tested or deployed in multiple configurations — the direct approach is fine.

The factory adds indirection. Don't add it before you need it.

---

## Trade-offs

| | Direct (`app = Flask(__name__)`) | Factory |
|---|---|---|
| Simplicity | Simpler, less boilerplate | More setup |
| Testability | Hard — one global config | Easy — fresh app per test |
| Circular imports | Prone | Avoided via `current_app` + blueprints |
| Multiple instances | Not possible | Straightforward |
| Extension compat | Some extensions don't support it | Industry standard |

---

## Related

- [How to organize with Blueprints](howto-blueprints.md)
- [Configuration Reference](reference-configuration.md)
- [Core API Reference](reference-core-api.md)
