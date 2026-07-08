# Flask Coding Standards

Rules for Flask applications. Read `COMMON_CODING_STANDARDS.md` and `PYTHON_STANDARDS.md` first.

---

## Blueprint State Co-location

In-memory state (dataclasses, threading locks) and the background thread that mutates
them must live in the same file as the routes that read and control that state. If state
is defined in `app.py` or a separate module while its controlling routes are in a
blueprint, the state gets orphaned whenever routes move and the ownership relationship
becomes invisible.

```python
# routes/scan.py — state, lock, background worker, and routes all in one file
from dataclasses import dataclass, field
import threading

@dataclass
class ScanState:
    running: bool = False
    progress: int = 0

_scan_state = ScanState()
_scan_lock = threading.Lock()

def _run_scan() -> None:          # background worker
    ...

scan_bp = Blueprint("scan", __name__)

@scan_bp.route("/scan/status")    # route that reads the state
def scan_status():
    ...
```

---

## url_prefix at Registration, Not in the Constructor

Set `url_prefix` in `app.register_blueprint()`, not in `Blueprint()`. Route decorators
then use paths relative to the prefix, keeping blueprint files readable without needing
to know where they are mounted.

```python
# Wrong — prefix baked into the blueprint, coupling it to a specific mount point:
scan_bp = Blueprint("scan", __name__, url_prefix="/api")

@scan_bp.route("/scan/status")    # full path is /api/scan/status — not obvious

# Correct — prefix supplied at registration:
scan_bp = Blueprint("scan", __name__)

@scan_bp.route("/scan/status")    # relative path; mounting decides the prefix

# app.py:
app.register_blueprint(scan_bp, url_prefix="/api")
```

---

## Shared Blueprint Helpers Go in a Shared Module

Any helper function needed by two or more blueprints belongs in a shared utility module
(e.g. `backend/utils.py`), not in one of the blueprints. A blueprint that other
blueprints import from is no longer independently deployable and is one refactor away
from a circular import.

This also follows directly from the Python rule against importing underscore-prefixed
names: if you find yourself writing `from routes.some_bp import _helper`, the function
must move to a shared module and be made public there.

---

## CORS Is Not CSRF Protection

`CORS(app, resources={...})` controls which origins can *read* a response — it does not
stop a browser from *sending* a cross-origin state-changing request. A malicious page on
another origin can still issue `fetch(url, {method: "POST"})` or submit a form to your API;
the browser just won't let that page read the JSON back. Every POST/PUT/PATCH/DELETE route
is a CSRF target unless something else checks where the request came from.

Add one `before_request` guard (not a per-route decorator) that rejects unsafe-method
`/api/*` requests whose `Origin` (falling back to `Referer`) doesn't match either the
request's own host or the existing CORS allowlist. A request with neither header (test
clients, curl, internal tooling) has nothing forgeable to check and should pass through.

Extract the origin-matching decision into a plain function in a shared module rather than
inlining it in the hook. The hook itself has to live on the `app` object at import time,
and the entry-point module usually can't be imported directly in tests once it has side
effects (DB init, background threads, dev-server launch) — the extracted function can be
unit-tested with no Flask app involved at all.

```python
# shared module:
def is_trusted_api_write_origin(source: str | None, self_origin: str, cors_origins: list[str]) -> bool:
    if not source:
        return True
    parsed = urlparse(source)
    return f"{parsed.scheme}://{parsed.netloc}" in {self_origin, *cors_origins}

# app.py, registered once after CORS(app, ...):
@app.before_request
def _reject_cross_origin_api_writes():
    if not request.path.startswith("/api/") or request.method not in {"POST", "PUT", "PATCH", "DELETE"}:
        return None
    source = request.headers.get("Origin") or request.headers.get("Referer")
    if not is_trusted_api_write_origin(source, f"{request.scheme}://{request.host}", _cors_origins):
        return jsonify({"error": "Cross-origin request rejected"}), 403
```
