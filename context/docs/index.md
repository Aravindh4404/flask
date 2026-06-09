# Flask Documentation

Flask is a lightweight WSGI web framework for Python. It is designed to make
getting started quick and easy, with the ability to scale up to complex
applications.

**Version**: 3.2.0.dev | **Python**: 3.10+  
**License**: BSD-3-Clause | **Source**: [github.com/pallets/flask](https://github.com/pallets/flask)

---

## Learn

Start here if you are new to Flask.

| Document | What it covers |
|---|---|
| [Getting Started Tutorial](tutorial-getting-started.md) | Build a working notes app from scratch — install to first HTTP response |

## Do

Step-by-step guides for specific tasks.

| Document | When to use it |
|---|---|
| [How to organize your app with Blueprints](howto-blueprints.md) | Splitting a growing app into reusable modules |
| [How to handle errors](howto-error-handling.md) | Custom 404/500 pages, JSON error responses, exception handlers |
| [How to write tests](howto-testing.md) | Using Flask's test client and pytest fixtures |

## Reference

Precise technical descriptions of every public API.

| Document | What it covers |
|---|---|
| [Core API Reference](reference-core-api.md) | `Flask`, `Blueprint`, proxies, helpers, signals |
| [Configuration Reference](reference-configuration.md) | Every built-in config key with type, default, and effect |

## Understand

Background reading on how and why Flask works the way it does.

| Document | What it covers |
|---|---|
| [Contexts and Proxies](explanation-contexts-and-proxies.md) | Why Flask uses context-local proxies instead of passing objects through every call |
| [The App Factory Pattern](explanation-app-factory-pattern.md) | Why you should create the Flask app inside a function, not at module level |

---

## Quick reference

```python
from flask import Flask, request, jsonify, redirect, url_for

app = Flask(__name__)

@app.route("/")
def index():
    return "Hello, Flask!"

@app.route("/greet/<name>")
def greet(name):
    return f"Hello, {name}!"

@app.route("/data", methods=["GET", "POST"])
def data():
    if request.method == "POST":
        return jsonify(request.json)
    return redirect(url_for("index"))
```

Run with:

```
flask --app app run --debug
```
