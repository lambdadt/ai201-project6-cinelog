# AGENTS.md — CineLog

CineLog is a Flask REST API for a community film tracking app backed by SQLite (via Flask-SQLAlchemy).

## Commands

```bash
# Create venv & install dependencies
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Run the app (starts Flask dev server on port 5000)
python app.py

# Run all tests
pytest tests/

# Run a single test file
pytest tests/test_collection.py

# Run a single test function
pytest tests/test_collection.py::test_add_to_collection_creates_entry

# Run tests matching a keyword expression
pytest tests/ -k "duplicate"

# Verbose test output
pytest tests/ -v
```

There is **no linter, formatter, or type checker** configured in this project. There is no `package.json`, no ESLint/Prettier, no `mypy.ini`, and no `pyproject.toml`. Stick to the conventions below rather than relying on tool enforcement.

---

## Project Structure

```
.
├── app.py                  # Flask app factory (create_app) + db init
├── models.py               # All SQLAlchemy models (single file)
├── services/               # Business logic, one file per domain
│   ├── collection_service.py
│   └── watchlist_service.py
├── routes/                 # HTTP endpoints as Flask Blueprints
│   ├── films.py
│   ├── collection.py
│   └── watchlist/
│       └── watchlist.py
├── tests/                  # Pytest test files
│   └── test_collection.py
└── instance/               # Runtime data — gitignored (contains cinelog.db)
```

**Application factory pattern** (`app.py:10`): The `db = SQLAlchemy()` instance is created at module level and initialized inside `create_app()`. Blueprints are imported inside `create_app()` to avoid circular imports, then registered with `url_prefix`.

---

## Code Style

### Imports

- **Order**: standard library → third-party → local, with a blank line between each group.
- **All imports are absolute** (e.g. `from app import db`, `from models import Film`). Never use relative imports (`from .models import …`).
- **Multi-line imports** use parenthesized grouping for 3+ names:

```python
from services.collection_service import (
    add_to_collection,
    remove_from_collection,
    get_collection,
    FilmNotFoundError,
)
```

- **Late imports** (inside functions) are used in `app.py:23-25` to break circular dependency between the app factory and blueprints that import `db`.

### Naming

| Category | Convention | Examples |
|---|---|---|
| Files / directories | `snake_case` | `collection_service.py`, `test_collection.py` |
| Functions | `snake_case` | `add_to_collection()`, `get_watchlist()` |
| Service functions | `verb_to_noun` pattern | `add_to_collection()`, `remove_from_collection()` |
| Classes (models, exceptions) | `PascalCase` | `User`, `Film`, `CollectionEntry` |
| Exception classes | `PascalCase` + `Error` suffix | `FilmNotFoundError`, `AlreadyInCollectionError` |
| Flask Blueprints | `snake_case` + `_bp` suffix | `films_bp`, `collection_bp`, `watchlist_bp` |
| DB columns | `snake_case` | `film_id`, `date_added`, `poster_url` |
| Test functions | `test_` prefix, descriptive | `test_add_to_collection_duplicate_raises` |

### Docstrings

All public modules and functions use **Google-style docstrings** with `Args:`, `Returns:`, and `Raises:` sections. File-level module docstrings are also required. See `services/collection_service.py:1-5` for module docstrings and `services/collection_service.py:27-41` for function docstrings.

### Error Handling

- **Custom exceptions** are defined at the top of each service file (e.g., `collection_service.py:12-24`). Route handlers catch these with `try/except` and return structured JSON errors with appropriate HTTP status codes.
- **Error response format**: `{"error": "<message>"}`.
- **HTTP status code mapping**: `400` (bad request / missing body), `404` (not found), `409` (conflict / duplicate), `201` (created).
- **Every service function that can fail must raise a typed exception**, not return `None` or a bare `False`.
- **Every route that calls a service must have try/except** that catches the relevant exception types and returns proper HTTP errors.

### Models

- All models are defined in `models.py`. Each model class has a `to_dict()` method for JSON serialization.
- UUIDs are generated via `str(uuid.uuid4())` (see `models.py:13-14`).
- Timestamps use `datetime.now(timezone.utc)` via `db.Column(default=…)`.
- `to_dict()` calls `.isoformat()` on datetime fields.

### Routes

- Each route file defines a single `Blueprint`. View functions route on the blueprint.
- URL patterns use `/<resource_id>` with a leading slash, relative to the blueprint's `url_prefix`.
- JSON request bodies are parsed with `request.get_json()`. Query parameters use `request.args.get()`.
- Always validate that required body fields exist before calling services, returning `{"error": "…"}, 400` if missing.

---

## Tests

- **Framework**: pytest (>=8.0.0). No config file — relies on defaults.
- **Isolation**: Every test uses an in-memory SQLite database (`sqlite:///:memory:`). The `app` fixture creates the DB, yields, then drops it.
- **Pattern**: Tests call service functions directly (not through HTTP). Assertions wrap in `with app.app_context():`.
- **Fixtures**: Use `@pytest.fixture` with `yield` and app-context blocks for setup (see `tests/test_collection.py:22-53`).
- **Required coverage** per `CONTRIBUTING.md:79-86` — each new service function needs:
  1. Happy path test
  2. Duplicate / conflict test
  3. Nonexistent ID test
- **`pytest.raises()`** is used to assert exception types and messages (see `test_collection.py:86-87`).

---

## Git Conventions

From `CONTRIBUTING.md`:

- **Conventional Commits**: `<type>: <imperative description>` — types: `feat`, `fix`, `test`, `docs`, `refactor`, `chore`.
- **One logical change per commit**. Use `git rebase -i` to squash messy history before PR.
- **No merge commits**. Rebase onto `origin/main` instead of merging main into your branch.
- **PR descriptions** must include: what the feature does, design decisions, and manual testing steps.
