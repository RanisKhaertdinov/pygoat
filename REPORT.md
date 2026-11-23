## Task 1 — Fork the Repository

We created a public fork of PyGoat for our team.

**Team members:** Ranis Haertdinov, Ilya Pushkarev, Anton Bugaev, Salim Shakirov  
**Repository:** [Team PyGoat fork](https://github.com/RanisKhaertdinov/pygoat)

## Task 2 — Create the Base Pipeline

A push triggered the GitHub Actions workflow. Bandit scanned the Python source code, and Docker Scout scanned the Docker image built from the repository.

| Scanner | Findings | Report |
|---|---|---|
| Bandit | 53 total — 23 High, 30 Medium | [Bandit JSON](reports/bandit-report.json) |
| Docker Scout | 119 total — 14 Critical, 105 High | [Docker Scout SARIF](reports/scout-report.sarif) |

Bandit returned exit code 1 because it found issues that met the configured severity and confidence thresholds. The workflow still uploaded the report.

**Pipeline run:** [Task 2 — Bandit and Docker Scout](https://github.com/RanisKhaertdinov/pygoat/actions/runs/45200029446)

## Task 3 — DAST

**Author:** Anton Bugaev

We added a `dast_scan` job to [`.github/workflows/lab5.yml`](.github/workflows/lab5.yml). The job does not use `needs` on Bandit or Docker Scout. Those jobs fail when they find issues, and that is expected in this lab. If DAST waited for them, it would never run.

What the job does:

1. Pull and start `pygoat/pygoat:latest` on port `8000` (official image, as allowed in the lab).
2. Wait until `http://127.0.0.1:8000` answers (curl loop, about 60 seconds max).
3. Run an OWASP ZAP baseline scan with `zaproxy/action-baseline@v0.14.0`.
4. Upload the ZAP output as the `zap-report` artifact (HTML, JSON, Markdown).
5. Stop the container.

**Pipeline run:** [Task 3 — DAST](https://github.com/RanisKhaertdinov/pygoat/actions/runs/45244450974)  
Job **Run DAST Scan** finished successfully (~2m25s). Bandit and Docker Scout failed in the same run because of findings — that is expected.

**Report in the repo:** [reports/zap/report_md.md](reports/zap/report_md.md)

### ZAP summary

| Risk | Alerts |
|---|---|
| High | 0 |
| Medium | 3 |
| Low | 8 |
| Informational | 6 |

Medium findings from the scan:

| Alert | CWE | What it means |
|---|---|---|
| Content Security Policy (CSP) Header Not Set | CWE-693 | Responses do not send a CSP header, so the browser has weaker XSS protections. |
| Absence of Anti-CSRF Tokens | CWE-352 | Some forms have no anti-CSRF token, so a third-party site could try to submit them. |
| Source Code Disclosure - Python | CWE-540 | A response looks like it exposes Python source, which helps an attacker learn how the app works. |

## Task 4 — Vulnerability Analysis

### Ranis Haertdinov — CVE-2020-14343 in PyYAML

Docker Scout found **PyYAML 5.1** in our PyGoat image. It rated this issue **9.3/10 (Critical)**.

**What is the problem?** PyYAML reads YAML files. In version 5.1, an attacker’s YAML file can run code if the app reads it with `FullLoader`. PyYAML 5.4 fixed this issue. Our scan found the affected version. We did not test whether PyGoat reads user files this way.

**CVSS score:** Amazon Linux gives this issue **9.8/10 (Critical)**. The vector below is from Amazon Linux.

`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

| Metric | Meaning |
|---|---|
| Attack Vector (AV:N) | The attacker can send the YAML file over a network. |
| Attack Complexity (AC:L) | The attack needs no special timing. |
| Privileges Required (PR:N) | The attacker needs no account or special permissions. |
| User Interaction (UI:N) | No other user needs to click or open anything. |
| Scope (S:U) | The code runs with the app’s permissions. |
| Confidentiality (C:H) | The attacker could read private data the app can access. |
| Integrity (I:H) | The attacker could change data the app can edit. |
| Availability (A:H) | The attacker could stop the app. |

**Real-world impact:** If a service uses `FullLoader` to read YAML from users, someone could send a file that makes the service read private data, change records, or stop working.

**Evidence:** [Docker Scout report](reports/scout-report.sarif) · [Pipeline run](https://github.com/RanisKhaertdinov/pygoat/actions/runs/45200029446)

**Sources:** [GitHub Advisory](https://github.com/advisories/GHSA-8q59-q68h-6hv4) · [Amazon Linux](https://explore.alas.aws.amazon.com/CVE-2020-14343.html)

### Anton Bugaev — CVE-2024-34069 in Werkzeug

Docker Scout found **Werkzeug 2.1.2** in our PyGoat image. It rated this issue **7.5/10 (High)**.

**What is the problem?** Werkzeug ships a debugger for development. In versions before **3.0.3**, an attacker who controls a domain and a subdomain can try to get a developer to open that site and enter the debugger PIN. If that works, the attacker can use the debugger even when the app only listens on localhost, and can run code on the developer’s machine. The attacker also needs to guess a URL that triggers the debugger. Our image has Werkzeug **2.1.2**, which is in the affected range (`< 3.0.3`). The fix is **3.0.3**.

**CVSS score:** NVD / GitHub Advisory give this issue **7.5/10 (High)**. The vector below is from NVD (CVSS 3.1).

`CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:H/I:H/A:H`

| Metric | Meaning |
|---|---|
| Attack Vector (AV:N) | The attack starts over the network (attacker-controlled site). |
| Attack Complexity (AC:H) | The attack is hard: the attacker needs a matching domain/subdomain setup, a guessed debugger URL, and the PIN. |
| Privileges Required (PR:N) | The attacker needs no account in our app. |
| User Interaction (UI:R) | A developer must open the attacker’s site and enter the debugger PIN. |
| Scope (S:U) | The impact stays on the same system (the developer machine / process). |
| Confidentiality (C:H) | Successful RCE can expose private data on that machine. |
| Integrity (I:H) | The attacker can change files or application state. |
| Availability (A:H) | The attacker can stop or break the process. |

**Privilege level needed:** No login to PyGoat. The attacker needs the victim (usually a developer with the Werkzeug debugger on) to interact with the attacker’s domain and enter the PIN.

**Real-world impact:** A developer running a local Flask/Werkzeug app with the debugger enabled could be tricked into giving an attacker a remote shell on their machine. In a team that debugs web apps locally, that can lead to stolen source code, tokens, or further access to company systems.

**Evidence:** [Docker Scout report](reports/scout-report.sarif) · [Pipeline run with Scout findings](https://github.com/RanisKhaertdinov/pygoat/actions/runs/45244450974)

**Sources:** [NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-34069) · [GitHub Advisory GHSA-2g68-c3qc-8985](https://github.com/advisories/GHSA-2g68-c3qc-8985) · [Werkzeug fix](https://github.com/pallets/werkzeug/security/advisories/GHSA-2g68-c3qc-8985)
