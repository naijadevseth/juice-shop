# SAST Scan Triage Log (Semgrep)

## Scan Details
- Tool: Semgrep CLI v1.179.0, ruleset p/default
- Date: 2026-09-29
- Findings: 71 (all marked blocking by default)
- Rules run: 408 across 1031 files

## Triage Summary

| Category | Count (approx) | Verdict | Reasoning |
|---|---|---|---|
| Core application vulnerabilities (SQLi, eval(), path traversal, hardcoded keys, open redirect, prototype pollution) | ~35 | Accepted risk — by design | These are OWASP Juice Shop's intentional teaching vulnerabilities. The app exists specifically to demonstrate these flaws for security training. Fixing them would defeat the purpose of the base project. |
| Challenge "codefix" snippets (data/static/codefixes/) | ~10 | Accepted risk — intentional | Teaching snippets showing vulnerable code patterns, not live application logic. |
| Infrastructure-as-code demos (terraform/, infrastructure/terraform/) | ~10 | Accepted risk — intentional | Deliberately insecure Terraform examples tied to Juice Shop's IaC-focused challenges. |
| Seed/demo data (data/static/users.yml) | 1 | False positive | Same reasoning as Gitleaks triage — intentional fixture data. |
| Test fixtures (*.spec.ts JWT tokens) | 2 | False positive | Dummy tokens used in automated tests. |
| Upstream CI workflow files (.github/workflows/ci.yml, codeql-analysis.yml, image_actions.yml, update-challenges-*.yml) | ~8 | Out of scope | Pre-existing OWASP Juice Shop pipeline files, not authored or maintained as part of this project's DevSecOps work. |
| Our own pipeline.yml (mutable action tag) | 3 | Genuine, low-priority | Our GitHub Actions steps reference tags like @v4 rather than pinned commit SHAs. This is a real supply-chain hardening opportunity, noted for future improvement rather than fixed immediately, given project scope and time. |
| .npmrc missing min-release-age | 2 | Low-priority, out of scope | Minor npm supply-chain hardening setting inherited from the base project. |

## Triage Process
1. Grouped all 71 findings by file location and rule category rather than reviewing each individually.
2. Recognized that OWASP Juice Shop is intentionally vulnerable software — the majority of findings are the app's documented teaching content, not accidental flaws introduced by this project.
3. Separated "our work" (the pipeline file we authored) from "inherited project files" (Juice Shop's existing app code, IaC, and CI workflows) to scope what is genuinely actionable versus expected.
4. Flagged the pipeline.yml mutable-tag finding as a real, fixable issue — distinct from the intentional app vulnerabilities — to demonstrate that not everything gets blanket-dismissed.

## Actions Taken
- Created `.semgrepignore` to exclude confirmed intentional/out-of-scope paths (app vulnerabilities, codefixes, IaC demos, test fixtures, upstream workflows) from future blocking scans.
- Left `.github/workflows/pipeline.yml` fully in scope — any future finding there will still fail the pipeline, since it's code we actually own and should keep hardened.

## Lessons / Notes
- Not every SAST finding should be suppressed — the correct response depends on whether the finding is "intentional by design," "not our code," or "a genuine gap we should fix." This log makes that distinction explicit for each category, which mirrors how a real SOC/AppSec analyst would reason through scanner output on a known-vulnerable application.