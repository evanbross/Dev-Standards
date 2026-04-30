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
