# Getting Started with Flask

You'll build a working notes app: a server that stores text notes in memory,
returns them as JSON, and lets you create and delete them over HTTP. By the end
you'll have a running Flask server, know how routing and views work, and
understand how to inspect and test what you built.

Total time: approximately 20 minutes.

---

## What you'll need

- Python 3.10 or later: `python --version`
- pip: `pip --version`
- A terminal

---

## Step 1: Install Flask

Create a project folder and a virtual environment:

```
mkdir notes-app
cd notes-app
python -m venv .venv
```

Activate it:

```
# macOS / Linux
source .venv/bin/activate

# Windows
.venv\Scripts\activate
```

Install Flask:

```
pip install flask
```

Verify:

```
flask --version
```

Expected: `Flask 3.x.x` (or similar).

---

## Step 2: Write your first view

Create `app.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def index():
    return {"message": "Notes API", "version": "1.0"}
```

Run it:

```
flask --app app run --debug
```

Expected output:
```
 * Serving Flask app 'app'
 * Debug mode: on
 * Running on http://127.0.0.1:5000
```

Open `http://127.0.0.1:5000` in a browser or run:

```
curl http://127.0.0.1:5000
```

Expected: `{"message": "Notes API", "version": "1.0"}`

Flask returned the dict as a JSON response automatically. You built your first
working endpoint in 6 lines.

---

## Step 3: Add an in-memory data store

Replace the contents of `app.py`:

```python
from flask import Flask, request, jsonify, abort

app = Flask(__name__)

# In-memory store: {id: {"id": int, "text": str}}
notes = {}
_next_id = 1


@app.route("/")
def index():
    return {"message": "Notes API", "version": "1.0"}


@app.route("/notes", methods=["GET"])
def list_notes():
    return jsonify(list(notes.values()))


@app.route("/notes/<int:note_id>", methods=["GET"])
def get_note(note_id):
    note = notes.get(note_id)
    if note is None:
        abort(404)
    return jsonify(note)
```

Restart the server (or let `--debug` hot-reload it) and try:

```
curl http://127.0.0.1:5000/notes
```

Expected: `[]` (empty list)

```
curl http://127.0.0.1:5000/notes/999
```

Expected: `404 NOT FOUND`

What you just learned:
- `<int:note_id>` in the route captures a URL segment and converts it to an `int`
- `abort(404)` stops the view and returns a 404 response
- `jsonify()` creates a `Content-Type: application/json` response

---

## Step 4: Accept POST requests to create notes

Add these functions to `app.py`:

```python
@app.route("/notes", methods=["POST"])
def create_note():
    global _next_id

    data = request.get_json(silent=True)
    if not data or "text" not in data:
        abort(400, "Request body must be JSON with a 'text' field.")

    note = {"id": _next_id, "text": data["text"]}
    notes[_next_id] = note
    _next_id += 1

    return jsonify(note), 201


@app.route("/notes/<int:note_id>", methods=["DELETE"])
def delete_note(note_id):
    note = notes.pop(note_id, None)
    if note is None:
        abort(404)
    return "", 204
```

Test it:

```
# Create a note
curl -X POST http://127.0.0.1:5000/notes \
     -H "Content-Type: application/json" \
     -d '{"text": "Buy oat milk"}'
```

Expected:
```json
{"id": 1, "text": "Buy oat milk"}
```

```
# List all notes
curl http://127.0.0.1:5000/notes
```

Expected:
```json
[{"id": 1, "text": "Buy oat milk"}]
```

```
# Delete it
curl -X DELETE http://127.0.0.1:5000/notes/1
```

Expected: empty body, status 204.

```
# Confirm it's gone
curl http://127.0.0.1:5000/notes/1
```

Expected: 404.

---

## Step 5: Add a custom 404 handler

Right now, a 404 returns Werkzeug's default HTML page. For an API, JSON is better.

Add to `app.py`:

```python
@app.errorhandler(404)
def not_found(error):
    return jsonify(error="not_found", message=str(error)), 404


@app.errorhandler(400)
def bad_request(error):
    return jsonify(error="bad_request", message=str(error)), 400
```

Test:

```
curl http://127.0.0.1:5000/notes/999
```

Expected:
```json
{"error": "not_found", "message": "404 Not Found: ..."}
```

---

## Step 6: Inspect your routes

Flask can list every registered route:

```
flask --app app routes
```

Expected output:
```
Endpoint     Methods    Rule
-----------  ---------  ----------------
index        GET        /
list_notes   GET        /notes
create_note  POST       /notes
get_note     GET        /notes/<int:note_id>
delete_note  DELETE     /notes/<int:note_id>
static       GET        /static/<path:filename>
```

---

## Step 7: Write a test

Install pytest:

```
pip install pytest
```

Create `tests/test_notes.py`:

```python
import pytest
from app import app as flask_app


@pytest.fixture
def client():
    flask_app.config["TESTING"] = True
    with flask_app.test_client() as client:
        yield client


def test_list_empty(client):
    response = client.get("/notes")
    assert response.status_code == 200
    assert response.get_json() == []


def test_create_and_get(client):
    # Create
    response = client.post("/notes", json={"text": "Hello"})
    assert response.status_code == 201
    note = response.get_json()
    assert note["text"] == "Hello"
    note_id = note["id"]

    # Get by id
    response = client.get(f"/notes/{note_id}")
    assert response.status_code == 200
    assert response.get_json()["text"] == "Hello"


def test_delete(client):
    response = client.post("/notes", json={"text": "Delete me"})
    note_id = response.get_json()["id"]

    response = client.delete(f"/notes/{note_id}")
    assert response.status_code == 204

    response = client.get(f"/notes/{note_id}")
    assert response.status_code == 404


def test_create_missing_text(client):
    response = client.post("/notes", json={"wrong": "field"})
    assert response.status_code == 400
```

Run:

```
pytest -v
```

All four tests should pass.

---

## What you built

You have a running Flask API with:
- **GET /notes** — list all notes
- **POST /notes** — create a note (JSON body with `text`)
- **GET /notes/{id}** — get a single note
- **DELETE /notes/{id}** — delete a note
- Custom JSON error responses for 400 and 404
- A pytest test suite that exercises the happy path and edge cases

The final `app.py`:

```python
from flask import Flask, request, jsonify, abort

app = Flask(__name__)

notes = {}
_next_id = 1


@app.route("/")
def index():
    return {"message": "Notes API", "version": "1.0"}


@app.route("/notes", methods=["GET"])
def list_notes():
    return jsonify(list(notes.values()))


@app.route("/notes/<int:note_id>", methods=["GET"])
def get_note(note_id):
    note = notes.get(note_id)
    if note is None:
        abort(404)
    return jsonify(note)


@app.route("/notes", methods=["POST"])
def create_note():
    global _next_id
    data = request.get_json(silent=True)
    if not data or "text" not in data:
        abort(400, "Request body must be JSON with a 'text' field.")
    note = {"id": _next_id, "text": data["text"]}
    notes[_next_id] = note
    _next_id += 1
    return jsonify(note), 201


@app.route("/notes/<int:note_id>", methods=["DELETE"])
def delete_note(note_id):
    note = notes.pop(note_id, None)
    if note is None:
        abort(404)
    return "", 204


@app.errorhandler(404)
def not_found(error):
    return jsonify(error="not_found", message=str(error)), 404


@app.errorhandler(400)
def bad_request(error):
    return jsonify(error="bad_request", message=str(error)), 400
```

---

## Next steps

- **Persist data** — swap the in-memory `dict` for SQLite with Flask-SQLAlchemy
- **Split the app** — see [How to organize with Blueprints](howto-blueprints.md)
- **Understand contexts** — see [Contexts and Proxies](explanation-contexts-and-proxies.md)
- **Full API reference** — see [Core API Reference](reference-core-api.md)
