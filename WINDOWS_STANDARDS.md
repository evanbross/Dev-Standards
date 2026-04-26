# Windows Standards

Rules for Windows service development and system automation. Read `COMMON_CODING_STANDARDS.md` first.

---

## Line Endings

Windows projects must use CRLF (`\r\n`) line endings for all text files. Configure this
in `.gitattributes` at the repo root so it is enforced for every contributor regardless
of their editor settings:

```
* text=auto eol=crlf
```

Set the matching Git config so the working copy always checks out CRLF:

```
git config core.autocrlf true
```

If a file has been committed with LF endings, normalize it with:

```
git add --renormalize .
```

Never commit files with mixed line endings — they produce noisy diffs and can break
Windows-only tooling (batch scripts, PowerShell scripts, SCM-invoked executables) that
parse line endings literally.

---

## pywin32 Windows Services

When writing a `win32serviceutil.ServiceFramework` subclass, `SvcDoRun` must call
`self.ReportServiceStatus(win32service.SERVICE_RUNNING)` as its **first line** — before
any config loading, logging setup, or other initialization. The SCM will kill the process
if it doesn't receive this acknowledgement within ~30 seconds, producing "The service did
not respond to the start or control request in a timely fashion."

Always install pywin32 services using the venv Python directly
(`.venv\Scripts\python.exe service.py install`), never via `uv run python`. The `uv run`
wrapper causes the SCM to register the wrong executable path.

The SCM does not add the script's directory to `sys.path` when launching a service. Any
service module that imports local files must explicitly prepend its own directory at the
very top of the file, before any local imports:

```python
sys.path.insert(0, str(Path(__file__).resolve().parent))
```

---

## NSSM

When configuring NSSM (`nssm install` / `nssm set Application`), always use **absolute
paths** for both the executable and the script. The SCM has no working directory context,
so relative paths like `.venv\Scripts\python.exe` will fail with "The system cannot find
the file specified."
