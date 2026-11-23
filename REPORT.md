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
