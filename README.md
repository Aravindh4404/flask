<div align="center"><img src="https://raw.githubusercontent.com/pallets/flask/refs/heads/stable/docs/_static/flask-name.svg" alt="" height="150"></div>

# Flask

Flask is a lightweight [WSGI] web application framework. It is designed
to make getting started quick and easy, with the ability to scale up to
complex applications. It began as a simple wrapper around [Werkzeug]
and [Jinja], and has become one of the most popular Python web
application frameworks.

Flask offers suggestions, but doesn't enforce any dependencies or
project layout. It is up to the developer to choose the tools and
libraries they want to use. There are many extensions provided by the
community that make adding new functionality easy.

[WSGI]: https://wsgi.readthedocs.io/
[Werkzeug]: https://werkzeug.palletsprojects.com/
[Jinja]: https://jinja.palletsprojects.com/

## A Simple Example

```python
# save this as app.py
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, World!"
```

```
$ flask run
  * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

## Documentation

The official Pallets documentation lives at [flask.palletsprojects.com](https://flask.palletsprojects.com/).

Structured Markdown docs (Diataxis format) are available in [context/docs/](context/docs/):

| Document | Type | Description |
|---|---|---|
| [Getting Started Tutorial](context/docs/tutorial-getting-started.md) | Tutorial | Build a working JSON API from scratch |
| [Core API Reference](context/docs/reference-core-api.md) | Reference | Flask, Blueprint, proxies, helpers, signals |
| [Configuration Reference](context/docs/reference-configuration.md) | Reference | Every built-in config key |
| [Contexts and Proxies](context/docs/explanation-contexts-and-proxies.md) | Explanation | Why `request`, `g`, `current_app` work without passing them around |
| [App Factory Pattern](context/docs/explanation-app-factory-pattern.md) | Explanation | When and why to create the app inside a function |
| [How to use Blueprints](context/docs/howto-blueprints.md) | How-to | Splitting a growing app into modules |
| [How to handle errors](context/docs/howto-error-handling.md) | How-to | Custom 404/500 pages, JSON errors |
| [How to write tests](context/docs/howto-testing.md) | How-to | pytest fixtures, test client, CLI runner |

## Donate

The Pallets organization develops and supports Flask and the libraries
it uses. In order to grow the community of contributors and users, and
allow the maintainers to devote more time to the projects, [please
donate today].

[please donate today]: https://palletsprojects.com/donate

## Contributing

See our [detailed contributing documentation][contrib] for many ways to
contribute, including reporting issues, requesting features, asking or answering
questions, and making PRs.

[contrib]: https://palletsprojects.com/contributing/
