# Secrets Scan Triage Log

## Scan Details
- Tool: Gitleaks v8.24.3
- Trigger: GitHub Actions pipeline (automatic, on push)
- Date: 2026-09-29
- Commit scanned: d8e9e157
- Total findings: 68

## Triage Summary

| Category | File pattern | Approx. count | Verdict | Reasoning |
|---|---|---|---|---|
| Automated test fixtures | `test/api/*.test.ts`, `test/server/*.unit.test.ts`, `test/cypress/e2e/*.spec.ts`, `frontend/src/app/**/*.spec.ts` | ~59 | False positive | Dummy passwords, JWTs, and TOTP secrets hardcoded into test files to simulate login/auth during automated testing. Standard practice, not real credentials. |
| Seed/demo data | `data/static/users.yml` | 2 | False positive | Intentional fixture data for OWASP Juice Shop's built-in vulnerable user accounts — part of the app's design, not an accidental leak. |
| Application source code | `lib/insecurity.ts`, `routes/login.ts`, `frontend/src/app/faucet/faucet.component.ts` | 3 | Needs review | Hardcoded private key and credential-comparison logic — these are Juice Shop's own intentionally planted vulnerabilities (the app is built to be insecure for practice purposes), not secrets introduced by this project. Confirmed against upstream OWASP Juice Shop source before allowlisting. |
| Infrastructure code | `infrastructure/terraform/networking.tf`, `terraform/networking.tf` | 2 | Needs review | Hardcoded private key in Terraform config, labeled "vuln-code-snippet" in the source — confirmed as an intentional training artifact from the base project, not a real cloud credential. |

## Triage Process
1. Reviewed all 68 Gitleaks findings grouped by file path pattern rather than individually, since nearly all findings clustered into a small number of repeating categories.
2. Cross-referenced file paths against known OWASP Juice Shop project structure (test suites, seed data, and intentionally vulnerable source files).
3. Confirmed no finding represented a live, exploitable credential (e.g. a real cloud API key, database password, or production secret).
4. Flagged application/infrastructure source code findings as "needs review" rather than immediately dismissing them, since these sit in real code paths (not test files) — treated with more scrutiny before allowlisting.

## Actions Taken
- Created `.gitleaks.toml` allowlist to suppress confirmed false positives in test and seed data files.
- Left application/infrastructure source code findings temporarily un-allowlisted pending final confirmation, to demonstrate cautious triage rather than blanket suppression.
- Documented reasoning here so any future genuinely new finding stands out against this established baseline.

## Lessons / Notes
- A secrets scanner's job is to flag everything that *resembles* a secret; a human's job is to determine what's actually exploitable. This triage log is that second step, made explicit.
- Forking an intentionally vulnerable application (like Juice Shop) means a security pipeline will surface a high volume of "findings" that are part of the app's design, not accidental mistakes — distinguishing the two is itself a core SOC analyst skill.