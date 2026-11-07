# LiteLLM — CI/CD Supply Chain Hunt (P0 class)

**Date:** 2026-11-07
**Target:** BerriAI/litellm @ e9b1c110 (main)
**Program:** GitHub private vulnerability report only — Critical $1,500–$3,000 (P0 supply chain), High $500–$1,500 (P1)
**Issue class:** CI/CD supply chain (workflow triggers, script injection, secret exposure, artifact poisoning)
**Result:** NO FINDING — workflows are hardened. Details below.

## What was audited

All 40 workflows in `.github/workflows/`:

### 1. Dangerous triggers — NONE present
No `pull_request_target`, no `workflow_run`, no `issue_comment` anywhere. Every
secret-using workflow runs on `schedule` / `issues` / `push:main` / `workflow_dispatch`
(maintainer-only). Fork PRs never receive repository secrets.

### 2. Script injection — none
`${{ }}` inside `run:` blocks only ever expands `github.sha` and `matrix.*`
(compat-matrix-image.yml, image-scan.yml, publish-lint-base-counts.yml) — not
attacker-controlled event fields. Issue-body-driven workflows
(issue_label_claude_code.yml, issue-keyword-labeler.yml) pass data via `env:`,
never direct interpolation.

### 3. Actions hardening
- Actions pinned by commit SHA (e.g. actions/checkout@08eba0b27e820071cde6df949e0beb9ba4906955).
- `permissions: {}` as the default at workflow level; job-level permissions are minimal
  (e.g. `issues: write` only).
- `persist-credentials: false` on checkouts.
- They run zizmor (GitHub Actions security scanner) on themselves (zizmor.yml).

### 4. Fork-dependency guard
`guard-fork-dependencies.yml` blocks fork PRs from modifying uv.lock / pyproject.toml
(the dependency-confusion vector) — a mitigation they built proactively.

### 5. AI-agent surface
No workflow runs an AI agent with tool access on untrusted input. Claude usage is
limited to a label-adding workflow with `permissions: {}` + repo check.

## Also checked (adjacent classes, no finding)

- **SSRF via request-body `api_base`**: the request-body credential override path
  (`explicit_request_credentials`, openai_files_endpoints/common_utils.py:714) is gated by
  `forward_llm_provider_auth_headers`, which defaults to **False**
  (litellm_pre_call_utils.py:1533). Client-supplied `api_base` replaces the deployment's
  only alongside a client-supplied `api_key` (documented BYOK feature, opt-in) — not a
  default-config vulnerability.
- **MCP OAuth redirect handling** (mcp_server/discoverable_endpoints.py:563): redirect_uri
  validated by `validate_trusted_redirect_uri` (same-origin / loopback / https-only
  allowlist / native-client schemes), with a documented threat model. Hardened.

## Conclusion

No payout-ready finding in the CI/CD supply-chain class for BerriAI/litellm @ e9b1c110.
