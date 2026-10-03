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