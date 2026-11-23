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