# Mini Project 4 — Penetration Test Report: Vuln Hub

**Student:** Hosia Stokes
**Course:** INF601 — Advanced Programming in Python
**Track:** A (Cybersecurity)
**Target:** Vuln Hub (provided practice web application)
**Date of testing:** October 2026
**Seed (FHSU username):** `h_stokes`

---

## 1. Authorization & Ethics Statement

I, Hosia Stokes, performed this penetration test only against the Vuln Hub
application that was provided as the authorized target for INF601 Mini Project 4.
The application was run locally on my own machine, bound to `127.0.0.1` (localhost),
and was never exposed to any network or the public internet. No other systems,
hosts, accounts, or services were tested or accessed. All testing was conducted
for the educational purpose of this assignment, within the scope defined by the
course. I understand that testing systems outside this authorized scope would be
an academic-honesty and conduct violation, and I did not do so.

Signed: *Hosia Stokes* — October 2026

---

## 2. Scope

- **In scope:** the Vuln Hub Flask application only, running locally at
  `http://127.0.0.1:5001`, started with `FLAG_SEED=h_stokes`.
- **Out of scope:** everything else — any other host, network, or service.
- **Goal:** identify and exploit the planted vulnerabilities, capture each
  `FLAG{...}` by actually exploiting the bug, and document findings with
  proof-of-concept evidence and remediation advice.

All six flags were verified with the provided checker:

```
python3 check_flags.py submission.txt --username h_stokes
# 6/6 flags valid for username h_stokes
```

---

## 3. Methodology

I used a guided, white-box approach, driving the exploration with Claude Code as
an AI assistant (see the AI Usage section in the README). For each of the six
challenges the loop was:

1. **Goal** — pick the next vulnerability from the challenge board on the home page.
2. **Recon / understand** — review how that feature worked (including reading the
   relevant part of the application source) to understand *why* it was exploitable.
3. **Exploit** — perform the actual attack myself in the browser (or via the
   address bar / DevTools) and capture the resulting `FLAG{...}`.
4. **Verify** — confirm the flag with `check_flags.py` and save proof-of-concept
   evidence (screenshots + the exact payload).

Tools used: a web browser (Chrome) and its built-in Developer Tools (for editing
cookies), the URL/address bar, and the application's own forms and endpoints.

---

## 4. Findings Summary

| # | Vulnerability | OWASP category | Severity |
|---|---|---|---|
| 1 | SQL Injection (login bypass) | A03: Injection | Critical |
| 2 | IDOR (insecure direct object reference) | A01: Broken Access Control | High |
| 3 | Stored Cross-Site Scripting (XSS) | A03: Injection | High |
| 4 | Broken Authorization (forgeable role cookie) | A01: Broken Access Control | Critical |
| 5 | Path Traversal | A01: Broken Access Control | High |
| 6 | Sensitive Data Exposure | A05: Security Misconfiguration | Medium |

---

## 5. Findings (Detail)

### Finding 1 — SQL Injection (Login Bypass)

- **Severity:** Critical
- **Endpoint:** `POST /login`
- **Flag:** `FLAG{sqli_f956adfcbf57}`
- **Evidence:** `poc/poc-01a-sqli-payload.png`, `poc/poc-01b-sqli-flag.png`

**Description.** The login form builds its SQL query by concatenating the
user-supplied username and password directly into the query string, with no
parameterization:

```
SELECT id, username, role FROM users WHERE username = '<input>' AND password = '<input>'
```

Because input is placed directly into the query, an attacker can inject SQL and
change what the query does.

**Exploit.** In the username field I entered:

```
admin' --
```

(and any value, e.g. `x`, in the password field). This turns the query into:

```
SELECT id, username, role FROM users WHERE username = 'admin' -- ' AND password = 'x'
```

The single quote closes the username string early, and `--` comments out the rest
of the line, deleting the password check. The database returns the `admin` row
with no password verification, so the application logs me in as **admin**. The
admin dashboard then displays the flag.

**Remediation.** Use parameterized queries / prepared statements so user input is
always treated as data, never as SQL (e.g. `db.execute("... WHERE username = ? AND
password = ?", (username, password))`). Also store password hashes, not plaintext,
and apply least-privilege database accounts.

---

### Finding 2 — IDOR (Insecure Direct Object Reference)

- **Severity:** High
- **Endpoint:** `GET /messages/<id>`
- **Flag:** `FLAG{idor_7a613c270b94}`
- **Evidence:** `poc/poc-02-idor.png`

**Description.** The message listing page correctly shows only the logged-in user's
own messages, but the message detail page fetches a message purely by the numeric
ID in the URL and never checks whether the requesting user owns that message:

```
SELECT id, owner, title, body FROM messages WHERE id = <id>   # no ownership check
```

**Exploit.** Logged in as the normal user **alice** (`alice / password123`), I
opened one of my own messages and observed the URL ended in a number. I changed
that number to `1`:

```
http://127.0.0.1:5001/messages/1
```

Message #1 belongs to **admin**, not alice, but the application returned it anyway,
exposing admin's private "Onboarding secret" message containing the flag. No code
injection and no elevated privileges were needed — just changing a number.

**Remediation.** Enforce an ownership/authorization check on every object access:
before returning a message, confirm the current user owns it (e.g. `WHERE id = ?
AND owner = ?`), and return 403/404 otherwise.

---

### Finding 3 — Stored Cross-Site Scripting (XSS)

- **Severity:** High
- **Endpoint:** `POST /guestbook` (stored), triggered when the admin views the page
- **Flag:** `FLAG{xss_2976401e3578}`
- **Evidence:** `poc/poc-03-xss-grab.png`

**Description.** Guestbook comments are rendered without output escaping (the
template uses the `| safe` filter), so any HTML or JavaScript in a comment is
rendered as live code in the browser of everyone who views the page — including a
privileged admin who reads the guestbook.

**Exploit.** I posted the following comment:

```html
<script>fetch('/grab?c=' + document.cookie)</script>
```

Because the comment is stored and rendered as real HTML, the script executed in the
admin's browser when the admin viewed the guestbook. `document.cookie` read the
admin's session cookie, and `fetch('/grab?c=' + ...)` sent it to the `/grab`
endpoint, which logs received data. Visiting `/grab` revealed the captured admin
cookie, which contained the flag (`session=admin; flag=FLAG{xss_2976401e3578}`).
This demonstrates session/cookie theft, which in a real application enables account
takeover.

**Remediation.** Escape/encode all user-supplied output by default (do not use
`| safe` on untrusted content). Add a Content-Security-Policy to restrict script
execution, and mark session cookies `HttpOnly` so JavaScript cannot read them.

---

### Finding 4 — Broken Authorization (Forgeable Role Cookie)

- **Severity:** Critical
- **Endpoint:** `GET /vault`
- **Flag:** `FLAG{authz_0fa6edc9f226}`
- **Evidence:** `poc/poc-04-authz-vault.png`

**Description.** The application stores the user's identity and role in an `auth`
cookie that is only base64-encoded — not signed or encrypted. The `/vault` page
grants access to anyone whose role is `superadmin`, and it trusts the role value
straight from this client-side cookie. No normal login ever issues the
`superadmin` role, so the only way in is to forge the cookie.

**Exploit.** Logged in as **alice**, my `auth` cookie decoded from base64 to:

```
alice:user
```

I changed the role and re-encoded it:

```
alice:superadmin  ->  base64  ->  YWxpY2U6c3VwZXJhZG1pbg==
```

I replaced my `auth` cookie value with the forged one (via browser DevTools /
`document.cookie`) and visited `/vault`. The application read role = `superadmin`
and granted access ("Access granted, superadmin."), revealing the flag — all while
still logged in as the ordinary user alice.

**Remediation.** Never trust client-side data for authorization decisions. Keep the
role server-side (in a session store), or cryptographically sign the cookie (e.g. a
signed JWT or Flask's signed sessions) so tampering is detected and rejected.

---

### Finding 5 — Path Traversal

- **Severity:** High
- **Endpoint:** `GET /download?file=<name>`
- **Flag:** `FLAG{traversal_890c8b642c8b}`
- **Evidence:** `poc/poc-05-traversal.png`

**Description.** The download endpoint builds a file path by joining the user-
supplied `file` parameter onto a base directory with no sanitization:

```
path = os.path.join(APP_DIR, "public", <file>)
```

Supplying `../` sequences lets an attacker climb out of the intended `public/`
folder and read other files on the server.

**Exploit.** I requested:

```
http://127.0.0.1:5001/download?file=../private/employee_notes.txt
```

This resolved to the sibling `private/` directory and served a confidential file
("CONFIDENTIAL - employees only") containing the flag.

**Remediation.** Reject or sanitize `..` and path separators in user input; resolve
the final path and confirm it stays within the intended base directory (e.g.
`os.path.realpath` + prefix check), or map requests to an allow-list of known-safe
filenames instead of using raw user input.

---

### Finding 6 — Sensitive Data Exposure

- **Severity:** Medium
- **Endpoint:** `GET /robots.txt` → `GET /backups/vulnhub.sql.bak`
- **Flag:** `FLAG{exposure_2980e5bbe521}`
- **Evidence:** `poc/poc-06-exposure.png`

**Description.** A sensitive database backup was left in a directory that is served
directly, and `/robots.txt` advertised that directory. `robots.txt` is intended to
ask crawlers not to index paths, but listing a path there effectively discloses it
to anyone.

**Exploit.** I read `http://127.0.0.1:5001/robots.txt`, which contained
`Disallow: /backups/`. Following that breadcrumb, I requested the leftover backup:

```
http://127.0.0.1:5001/backups/vulnhub.sql.bak
```

The file ("database backup ... should NOT be world-readable ... left in /backups/
by mistake") contained the flag inside an `INSERT INTO secrets VALUES (...)` line.

**Remediation.** Do not store backups or other sensitive files under a web-served
directory. Remove them from the deployed application, restrict access, and do not
rely on `robots.txt` to hide anything — it is public and only guides crawlers.

---

## 6. Conclusion

All six planted vulnerabilities were identified and exploited, and all six flags
were captured and verified (6/6 valid for `h_stokes`). The most severe issues
(SQL injection and the forgeable authorization cookie) allow full account and
privilege compromise. The common theme across the findings is **trusting user-
controlled input** — whether in SQL queries, object IDs, rendered output, cookies,
file paths, or exposed files. Consistent use of parameterized queries, server-side
authorization checks, output encoding, signed sessions, input sanitization, and
proper file handling would remediate the application.
