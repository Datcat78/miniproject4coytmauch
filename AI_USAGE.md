## AI Usage

**Goal.** Run an authorized, local-only penetration test of the instructor-
provided Vuln Hub and produce this submission. I directed the work; **Claude
Code** (Anthropic's agentic CLI) executed the tool calls and drafted the
write-ups under my direction. The full phase-by-phase record (original session
plus later corrections) is in `AI_WORKFLOW.md`; the report's own account is
`PENTEST_REPORT.md` §9. This file and the README's `## AI Usage` section are the
same summary.

**How it ran (goal → tool calls → results → my review).** For each step I set a
goal; Claude ran the tool calls (`curl`, file read/write, `git`) and the real
results were saved under `poc/` and shown back to me; I then decided whether to
continue, approving each finding in turn.

**Original testing session.** Claude read `app.py` and the Track A README, then
exploited the six bugs one at a time with `curl` — SQLi login bypass; IDOR on
`/messages/1`; stored XSS through the guestbook to `/grab`; forged `superadmin`
cookie to `/vault`; path traversal on `/download`; exposed backup via
`/robots.txt` → `/backups/`. Each flag was read from the live response and saved
under `poc/`. Claude wrote `submission.txt` and drafted `PENTEST_REPORT.md`, then
initialized the Git repository.

**Later corrections (2026-10-11).** At my direction Claude corrected several
technical explanations in the report (the XSS bot is a simulation that scans
comments and makes a request, not a browser running JavaScript; `os.path.join`
concatenates while the OS resolves `..`; the authorization remediation now
requires session-signature verification; CWE-548 was removed for lack of
directory-listing evidence; the "secrets regenerate" claim was corrected to note
the Base64 cookie is deterministic). Claude also captured extra evidence (IDOR
and `/vault` baselines, a full three-stage XSS capture, honest backup-filename
provenance) and filled the signed name/date after I confirmed the authorization
statement.

**What I did vs. what Claude did.** Claude drafted all report prose, the PoCs, and
these AI-usage notes, and ran the exploits and `git`. What I have confirmed is
explicit: I supplied `FLAG_SEED=cdmauch` and my name, confirmed the authorization
statement is accurate, authorized the typed signature/date, directed each step,
and directed the technical corrections. Independent line-by-line review of every
captured request/response is my responsibility; the evidence is preserved under
`poc/` for that review.

**Integrity.** No tests, results, screenshots, or flag values were invented.
Every flag was read from a live HTTP response and independently validated by the
starter's `check_flags.py` (6/6 valid for `cdmauch`); none was taken from the
planted local files.
