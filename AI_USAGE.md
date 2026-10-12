## AI Usage

I used **Claude Code** to help drive the authorized penetration test of the
instructor-provided Vuln Hub app, which runs only on my own machine. I remained
responsible for the work and can explain every step below.

### What I did
- Set up and ran the target locally with `FLAG_SEED=cdmauch`; confirmed scope and
  that I was authorized to test only this app.
- Directed the engagement, approved each action, and provided my FHSU username so
  the flags could be validated.
- Reviewed every request, response, and flag; ran `check_flags.py` output
  (`6/6 valid`); and verified the findings match the app's behavior.

### What Claude Code did (under my direction)
- Read `app.py` and the Track A `README.md` to locate the six planted bugs.
- Executed one finding at a time with `curl`, capturing the exact request and the
  actual response for each:
  1. **SQL injection** — login bypass with `username=admin' -- -`.
  2. **IDOR** — read admin's `/messages/1` from a normal `alice` session.
  3. **Stored XSS** — posted a `<script>…document.cookie…/grab</script>` comment;
     the simulated admin bot leaked its cookie to `/grab`.
  4. **Broken authorization** — forged `auth=base64("hacker:superadmin")` to reach
     `/vault`.
  5. **Path traversal** — `/download?file=../private/employee_notes.txt`.
  6. **Sensitive data exposure** — `/robots.txt` → `/backups/vulnhub.sql.bak`.
- Saved per-finding PoCs and raw responses under `poc/`, wrote `submission.txt`,
  and drafted this report (`report.md`).

### What I verified myself
- No flags were invented: every flag was read from a live HTTP response produced
  by the exploit, then independently confirmed with
  `python check_flags.py submission.txt --username cdmauch` (6/6 VALID).
- All testing stayed within scope (localhost, this app only).
