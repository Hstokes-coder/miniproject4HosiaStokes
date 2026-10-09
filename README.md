# Mini Project 4 — Vuln Hub Penetration Test

**Student:** Hosia Stokes
**Course:** INF601 — Advanced Programming in Python

Track: A

This repository contains my work for Mini Project 4, Track A (Cybersecurity):
an authorized penetration test of the provided **Vuln Hub** practice application,
in which I captured all six planted flags by exploiting the planted vulnerabilities,
plus the written report and proof-of-concept evidence.

## Repository contents

| Path | What it is |
|---|---|
| `report.md` | Full pentest report: authorization statement, scope, methodology, 6 findings (with severity, exploit, evidence, remediation), conclusion. |
| `submission.txt` | The six captured flags, one per line. |
| `check_flags.py` | The provided checker (included so results can be re-verified). |
| `poc/` | Proof-of-concept screenshots, one (or two) per finding. |
| `README.md` | This file. |

## Results

All six flags were captured and verified (6/6 valid for username `h_stokes`):

| # | Vulnerability | Flag prefix |
|---|---|---|
| 1 | SQL Injection (login bypass) | `FLAG{sqli_...}` |
| 2 | IDOR | `FLAG{idor_...}` |
| 3 | Stored XSS | `FLAG{xss_...}` |
| 4 | Broken Authorization (forgeable cookie) | `FLAG{authz_...}` |
| 5 | Path Traversal | `FLAG{traversal_...}` |
| 6 | Sensitive Data Exposure | `FLAG{exposure_...}` |

## How to reproduce

The target (Vuln Hub) is not included in this repo; it is the provided starter
application. To reproduce:

1. From the Vuln Hub `track_a_vulnhub/` folder, start the app with my seed:
   ```bash
   PORT=5001 FLAG_SEED=h_stokes python3 app.py
   ```
   (Or via Docker: `docker run -p 127.0.0.1:5001:5000 -e FLAG_SEED=h_stokes inf601-vulnhub`.)
   Confirm there is **no** red "DEFAULT seed" banner.
2. Open `http://127.0.0.1:5001` in a browser.
3. Follow the exploit steps for each finding in `report.md`.
4. Verify the flags:
   ```bash
   python3 check_flags.py submission.txt --username h_stokes
   ```

## Authorization statement

All testing was performed only against the provided Vuln Hub application, running
locally on my own machine (bound to `127.0.0.1`), and never exposed to any network.
No other systems were tested. The full signed-style authorization/ethics statement
is at the top of `report.md`.

## Agentic workflow (how I directed Claude Code)

This project was done as an agentic workflow, driving Claude Code as an AI assistant
while I performed the actual exploitation. For each of the six challenges I followed
the same loop:

1. **Goal** — I chose the next vulnerability from the challenge board.
2. **Direct the agent** — I asked Claude to explain the vulnerability class and,
   where useful, to read the relevant part of the Vuln Hub source so I understood
   *why* the feature was exploitable (white-box recon).
3. **Review** — Claude explained the mechanism and gave the exact payload / steps;
   I reviewed the explanation until I understood it.
4. **Exploit (me)** — I performed the actual attack myself in the browser / address
   bar / DevTools and captured the `FLAG{...}`.
5. **Verify** — I confirmed each flag with `check_flags.py` and saved screenshots as
   proof. After each finding I explained the vulnerability back in my own words to
   confirm I understood it (not just copied it).

Concretely, the agent helped me: get the app running with the correct seed; fix an
early mix-up where I typed a payload into the terminal instead of the browser;
understand SQL injection, IDOR, stored XSS, broken authorization, path traversal,
and sensitive-data exposure; and organize this repository and report. I ran every
exploit and verified every flag myself.

## AI Usage

This report is AI-assisted by design, and I want to be honest about the split:

**What the AI (Claude Code) did:**
- Explained each vulnerability class and why Vuln Hub was vulnerable, including by
  reading the relevant portions of the application source (recon).
- Provided the exploit payloads/steps and corrected my misunderstandings (for
  example, clarifying that the IDOR finding used no SQL/XSS and no admin rights, and
  that the XSS victim was the admin, not alice).
- Drafted this report and README from the work we did, and set up the repository
  structure and file naming.

**What I (Hosia Stokes) did:**
- Ran the target locally with my own seed and performed every exploit myself in the
  browser / DevTools / address bar.
- Captured all six flags and verified them with `check_flags.py`.
- Reviewed every explanation and restated each vulnerability in my own words to
  confirm I understood it, and verified the report's accuracy against what I
  actually did. I can explain every finding at the Week 16 review.
