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

For DAST I added a third job, `dast_scan`, in [`.github/workflows/lab5.yml`](.github/workflows/lab5.yml).

I did not put `needs: [sast_scan, image_scan]` on it. Bandit and Docker Scout exit with failure when they find issues (that is normal for this lab). If DAST depended on them, GitHub Actions would skip it every time.

Building our own image in CI was flaky earlier, so I used the option from the lab handout: pull the official image and run it on the runner.

```bash
docker pull pygoat/pygoat:latest
docker run --rm -d --name pygoat -p 8000:8000 pygoat/pygoat:latest
```

Before scanning I wait until the app answers. The step curls `http://127.0.0.1:8000/` in a loop (up to ~60s). If it never comes up, the job fails and prints `docker logs`.

Then I run OWASP ZAP baseline against that URL with `zaproxy/action-baseline@v0.14.0`. `fail_action` is false so alerts are written to the report even when ZAP finds medium issues. The report is uploaded as the `zap-report` artifact (HTML, JSON, Markdown). A copy is also in the repo: [reports/zap/report_md.md](reports/zap/report_md.md).

**Pipeline run:** [Task 3 — DAST](https://github.com/RanisKhaertdinov/pygoat/actions/runs/45244450974)

In that run **Run DAST Scan** is green (~2m25s). Bandit and Scout are red in the same run because of findings — expected.

### What ZAP reported

Target: `http://127.0.0.1:8000`

| Risk | Count |
|---|---|
| High | 0 |
| Medium | 3 |
| Low | 8 |
| Informational | 6 |

Medium alerts:

1. **CSP Header Not Set (CWE-693)** — pages like `/`, `/login/`, `/auth` do not send a `Content-Security-Policy` header. Without CSP the browser does less to block XSS / injection of scripts.
2. **Absence of Anti-CSRF Tokens (CWE-352)** — ZAP flagged forms that have no CSRF token, including Google login (`/accounts/google/login/`, POST). A malicious site could try to submit those forms in the user’s browser session.
3. **Source Code Disclosure - Python (CWE-540)** — on `POST /accounts/password/reset/` ZAP decided the response looks like Python source. That leaks how the endpoint is implemented and makes further attacks easier.

Low alerts (short list): missing `X-Content-Type-Options`, cookies without `HttpOnly`, server version in the `Server` header, application error pages, missing Permissions-Policy / CORP / COEP. Full list is in the ZAP Markdown report linked above.

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

Docker Scout found **Werkzeug 2.1.2** in the image (`pkg:pypi/werkzeug@2.1.2`). Severity in the Scout report: **7.5 High**. Affected range: `< 3.0.3`. Fixed in **3.0.3**.

**What is the problem?** Werkzeug has an interactive debugger used while developing. In vulnerable versions, if a developer opens a page on a domain/subdomain controlled by an attacker and types in the debugger PIN, the attacker can reach that debugger even when the app is bound to localhost. From there they can run code on the developer machine. They also need to hit a URL that actually opens the debugger. We only confirmed the vulnerable package version from the scan; we did not reproduce the full attack against PyGoat.

**CVSS score:** NVD lists **7.5 High** (CVSS 3.1):

`CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:H/I:H/A:H`

| Metric | Meaning |
|---|---|
| Attack Vector (AV:N) | Starts from the network (attacker hosts the evil page). |
| Attack Complexity (AC:H) | Needs the domain/subdomain trick, a debugger URL, and the PIN. Not a one-click script-kiddie bug. |
| Privileges Required (PR:N) | No account inside PyGoat. |
| User Interaction (UI:R) | Someone has to open the attacker’s page and enter the PIN. |
| Scope (S:U) | Impact is on that same machine/process. |
| Confidentiality (C:H) | Code execution can dump secrets / files. |
| Integrity (I:H) | Attacker can change data or code. |
| Availability (A:H) | Attacker can kill or break the process. |

**Privilege level:** no app login. The weak point is a developer with the Werkzeug debugger enabled who interacts with the attacker’s site.

**Real-world impact:** Local Flask debugging is common. If this works, the attacker gets a shell on the laptop — source, `.env`, SSH keys, cloud tokens, whatever is there.

**Evidence:** [Docker Scout report](reports/scout-report.sarif) · [Pipeline run](https://github.com/RanisKhaertdinov/pygoat/actions/runs/45244450974)

**Sources:** [NVD CVE-2024-34069](https://nvd.nist.gov/vuln/detail/CVE-2024-34069) · [GHSA-2g68-c3qc-8985](https://github.com/advisories/GHSA-2g68-c3qc-8985)
