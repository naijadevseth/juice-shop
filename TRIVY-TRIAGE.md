# Dependency Scan Triage Log (Trivy)

## Scan Details
- Tool: Trivy (via aquasecurity/trivy-action, pinned to commit ed142fd0673e97e23eac54620cfb913e5ce36c25, v0.36.0)
- Trivy engine version: 0.70.0
- Scan type: filesystem scan, CRITICAL/HIGH severity
- Date: 2026-10-03

## Results
- Dependency vulnerabilities (CRITICAL/HIGH): 0 findings
- Secrets detected (Trivy's built-in secret scanner): 3 findings — all hardcoded RSA private keys in lib/insecurity.ts, infrastructure/terraform/networking.tf, and terraform/networking.tf

## Triage
The 3 secret findings are the same intentional Juice Shop teaching artifacts already identified and documented in SECURITY-TRIAGE.md (Gitleaks) and SAST-TRIAGE.md (Semgrep). No new findings.

## Actions Taken
- Narrowed Trivy's configuration to `scanners: vuln` only, since secret detection is already handled by Gitleaks. This avoids duplicate alerts across tools and keeps each scanner focused on a distinct responsibility in the pipeline.

## Lessons / Notes
- Multiple security tools can legitimately overlap in coverage (here, both Gitleaks and Trivy detect secrets). Rather than running duplicate checks, the pipeline was tuned so each tool has a clear, non-overlapping responsibility: Gitleaks for secrets, Semgrep for code patterns, Trivy for dependency vulnerabilities. This is a deliberate pipeline design decision, not an oversight.
- Zero dependency vulnerabilities found is a genuinely good result — it means the project's current package versions don't have publicly known CRITICAL/HIGH severity issues at time of scan.

## Update — Scan Coverage Issue Found

An initial run of the narrowed (vuln-only) Trivy scan reported "0 vulnerabilities" — but on closer inspection, this was misleading: Trivy found **zero files to scan at all** (`Number of language-specific files num=0`), not zero vulnerabilities in scanned files.

**Root cause:** `.npmrc` and `frontend/.npmrc` (inherited from the base Juice Shop project) contain `package-lock=false`, which prevents `package-lock.json` from ever being generated during `npm install`. Trivy's dependency scanner relies on reading lockfiles to know what package versions are actually installed — with no lockfile present, it had nothing to analyze.

**Fix:** added a dedicated pipeline step to generate lockfiles specifically for scanning purposes (`npm install --package-lock-only --package-lock=true`), run just before the Trivy step. This overrides the `.npmrc` setting only for that one command, without changing how the application installs normally elsewhere in the pipeline.

**Why this matters / lesson learned:** a scan that reports "clean" isn't automatically trustworthy — it's important to confirm the scan actually had something to scan in the first place. A silently empty scan is a more dangerous failure mode than a scan that clearly errors out, because it looks identical to a genuinely clean result. This is a real example of the kind of verification a SOC/DevSecOps analyst should build the habit of doing: checking scan *coverage*, not just scan *output*.

## Results — Full Dependency Scan (after lockfile fix)

With lockfiles correctly generated, Trivy identified real findings:

| Target | Total | Critical | High |
|---|---|---|---|
| package-lock.json (root) | 51 | 9 | 42 |
| frontend/package-lock.json | 5 | 0 | 5 |

## Triage

**Verdict: Accepted risk — intentional, by design.**

Nearly every flagged library (lodash 2.4.2, jsonwebtoken 0.1.0/0.4.0, crypto-js 3.3.0, tar 6.2.1, multer 1.4.5, express-jwt 0.1.3, ws 7.4.6, socket.io-parser 4.0.5, moment 2.0.0, marsdb 0.6.11, decompress 4.2.1) is deliberately pinned to an old, known-vulnerable version. This isn't accidental — OWASP Juice Shop has a specific built-in challenge category ("Vulnerable Components," mapping to OWASP Top 10 A06) that exists specifically to teach identification and exploitation of outdated dependencies. Upgrading these packages would silently break the app's own teaching challenges.

**Reasoning applied:**
- Checked whether the flagged libraries correspond to known Juice Shop challenge dependencies rather than incidental transitive packages — they do, consistently across the list (jsonwebtoken, lodash, multer, crypto-js are all well-documented Juice Shop "vulnerable component" teaching targets).
- No action taken to patch/upgrade these dependencies, since doing so would remove intentional training content from the forked project — this mirrors the same reasoning already applied to the Semgrep SAST findings (routes/, lib/insecurity.ts, etc.).
- This scan still has real value even with everything "accepted": it demonstrates the pipeline can correctly detect and enumerate real, current CVEs against actual installed versions — which is the whole point of running it.

## Actions Taken
- No dependency upgrades applied (see reasoning above).
- Findings documented here as the final artifact of the dependency-scanning stage, completing the project's three-pillar security pipeline (secrets, SAST, dependencies).