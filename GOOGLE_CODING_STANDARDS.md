# Google API Coding Standards

Rules for projects that use Google Sheets, Docs, Drive, Photos, or related Google Workspace APIs.
Read `COMMON_CODING_STANDARDS.md` and `PYTHON_STANDARDS.md` first.

---

## Authentication — Never Trigger OAuth from Background Calls

Non-auth endpoints must never start an OAuth redirect or open a browser. If the access
token is missing or expired when a background or data endpoint is called, raise a
specific exception (e.g. `NotAuthenticatedError`) and return it to the caller as a 401.
Interactive OAuth flows belong exclusively in dedicated auth endpoints that the user
explicitly invokes.

---

## HttpError — Re-raise from Service Layer, Catch Explicitly in Routes

Service and repository methods that call Google APIs should catch `HttpError` from
`googleapiclient.errors` to log context, then re-raise. The route handler is responsible
for mapping it to an HTTP response code (typically 502 for a Sheets API error):

```python
# In a sheets service/repository method:
from googleapiclient.errors import HttpError

try:
    result = self._service.spreadsheets().get(...).execute()
except HttpError as exc:
    logger.warning("Sheets API error in read_application_grid: %s", exc)
    raise  # route handler decides the response code

# In the route handler:
from googleapiclient.errors import HttpError

try:
    data = sheets_client.read_application_grid()
except NotAuthenticatedError as exc:
    return jsonify({"error": str(exc)}), 401
except HttpError as exc:
    logger.warning("Sheets API error: %s", exc)
    return jsonify({"error": "Google Sheets API error"}), 502
except Exception as exc:
    logger.warning("Unexpected error: %s", exc)
    return jsonify({"error": "Unexpected error"}), 500
```

Never expose raw `str(exc)` from an `HttpError` to the client — the message may contain
internal project IDs or quota details.

---

## Grid Data Ranges — Never Hardcode a Column Letter Limit

Do not specify a static column letter bound (e.g. `A:Z`, `A:AZ`) in the `ranges`
parameter of a `spreadsheets.get` call. Static ranges silently truncate sheets wider
than the specified bound, and the data loss is invisible in logs.

Use the sheet name alone as the range to let the API return the natural extent of the
populated data:

```python
# Wrong — silently drops columns past Z:
ranges=[f"{sheet_name}!A:Z"]

# Correct — returns all populated data:
ranges=[sheet_name]
```

If you need the exact column count for programmatic range construction (e.g. when
building a write range), request `gridProperties.columnCount` from the API:

```python
fields="sheets(properties(gridProperties(columnCount,rowCount)),...)"
# then:
column_count = sheet["properties"]["gridProperties"]["columnCount"]
```

Converting the 1-based count to a letter (27 → `AA`) is a standard bijective base-26
conversion — write it with `divmod(n - 1, 26)` in a loop, not a lookup table.

---

## Always Specify the `fields` Parameter

Every `spreadsheets.get` call must include a `fields` parameter that lists only the
properties actually used. Omitting `fields` returns the entire spreadsheet metadata
(including all sheets, named ranges, developer metadata, etc.) which is wasteful and
slower for large workbooks.

```python
# Wrong — fetches everything:
self._service.spreadsheets().get(spreadsheetId=sid).execute()

# Correct — fetches only what the code reads:
self._service.spreadsheets().get(
    spreadsheetId=sid,
    fields="sheets(properties(title,gridProperties(columnCount)))"
).execute()
```

---

## `formattedValue` vs `effectiveValue`

- Use `formattedValue` when you want the display string the user sees (respects number
  formats, date formats, currency symbols, etc.). Suitable for read-only UI mirrors.
- Use `effectiveValue` when you need the underlying type (`numberValue`, `boolValue`,
  `stringValue`, `formulaValue`) for computation or storage.

Never parse a `formattedValue` string to recover a number or date — use `effectiveValue`
directly.

---

## Color Values Are 0..1 Floats

Google Sheets color objects use floating-point channels in the range `0.0`–`1.0`, not
0–255 integers — convert with `round(channel * 255)` per channel, not a direct cast.

Always use `.get("red", 0)` defaults, never `color["red"]` — Google omits a channel key
entirely when its value is 0, so a pure-green cell raises `KeyError` on the red channel.

---

## Hidden Columns and Rows

Check `columnMetadata.hiddenByUser` (and `rowMetadata.hiddenByUser` for rows) before
including them in any UI representation. Filter hidden columns out of the display list
rather than relying on CSS to hide them — the data round-trips cleanly and indices stay
consistent.

```python
columns = [col for col in raw_columns if not col.get("hidden")]
```

---

## Write Operations — Use Batch APIs

For writing multiple rows or cell ranges, use `values.batchUpdate` (or
`spreadsheets.batchUpdate` for formatting) instead of multiple individual `values.update`
calls. Google Sheets API has per-user and per-project read/write quotas; batching reduces
round-trips and quota consumption.

---

## Scope Minimization

Request the narrowest OAuth scope that satisfies the feature:

| Need | Scope |
|------|-------|
| Read sheets only | `https://www.googleapis.com/auth/spreadsheets.readonly` |
| Read + write sheets | `https://www.googleapis.com/auth/spreadsheets` |
| Read Drive metadata | `https://www.googleapis.com/auth/drive.readonly` |
| Read + write Drive files | `https://www.googleapis.com/auth/drive` |
| Read Gmail | `https://www.googleapis.com/auth/gmail.readonly` |

Do not request `drive` scope if `spreadsheets` scope is sufficient. Unnecessary scopes
trigger broader consent dialogs and increase the attack surface if a token is leaked.