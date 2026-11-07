# Hunt Log — 2026-11-07 — LiteLLM Proxy — issue class: missing authentication (P1 unauthenticated proxy access)

**Result: NO verified finding this session.** Full audit trail below. No report written (nothing proven to submit).

## Program record (Step 1)

- **Name:** LiteLLM bug bounty (BerriAI/litellm)
- **Program URL:** https://github.com/BerriAI/litellm/security/advisories/new (GitHub private vulnerability report — the ONLY submission channel; no HackerOne/Bugcrowd account needed)
- **Payout table:** Critical **$1,500–$3,000** (P0: supply-chain compromise of CI/CD → tampered PyPI/Docker/GHCR artifacts); High **$500–$1,500** (P1: unauthenticated access to protected data on a proxy instance, e.g. API keys); Medium/Low: N/A (P2 authenticated privilege escalation is accepted but unpaid)
- **Scope:** LiteLLM proxy (self-hosted). P0 = CI/CD supply chain; P1 = unauthenticated proxy access; P2 = authenticated malicious actions. Out of scope: attacks requiring misconfiguration (e.g. no `master_key` set), DoS, missing headers without impact, automated scan output
- **Triage:** "review all submissions promptly, follow up within 5 business days"
- **Launch date:** not found (program predates the 30-day window; found via GitHub SECURITY.md sweep, not an X announcement). Program is OPEN and actively paying — published advisories with researcher credits through Oct 2026 (e.g. GHSA-w4c7-34wj-hqm8, Oct 27, 2026)
- **Hard requirement:** every report MUST include a reproduction video (asciinema OK for CLI). Reports without a video are closed without review
- **Why picked:** the only program found where submission is GitHub-native AND the bounty is a published cash table. Scope is a public Python repo (clone + audit, no login). P1 class = $500–$1,500 per valid finding

## Method (ground truth, not grep)

1. Cloned `BerriAI/litellm` (main, depth 1).
2. Built a runnable env: `python3.12 -m venv ~/.venvs/litellm` + `pip install -e .` (the `[proxy]` extra fails — `litellm-proxy-extras==0.4.114` and `litellm-enterprise==0.1.83` are not yet published; install the base package + the extra's dep list minus those two).
3. **Pitfall (cost 20 min):** FastAPI 0.162 changed `include_router` — it appends a lazy `_IncludedRouter` wrapper instead of flattening routes, so `app.routes` shows only 10 APIRoutes and `isinstance(r, APIRoute)` misses everything. LiteLLM also lazy-registers 38 feature routers (`litellm/proxy/_lazy_features.py`). Fix: call `register_all_features(app, LAZY_FEATURES)`, then walk routes recursively through `route.original_router.routes`, duck-typing on `dependant`/`endpoint`/`methods`.
4. For every route, collected the full FastAPI dependency tree (nested `Depends` resolved) and flagged routes with no `user_api_key_auth` anywhere in the tree; then manually verified each flagged route for in-body auth or public-by-design status.

## Findings: none

Complete list of routes without `user_api_key_auth` in the dependency tree, and why each is NOT a vulnerability:

**Authenticated inside the handler body (verified by reading the code):**
- `/gemini/{endpoint:path}`, `/langfuse/{endpoint:path}`, `/vertex_ai/discovery/{endpoint:path}` — pass-through routes call `await user_api_key_auth(...)` in the body (llm_passthrough_endpoints.py:394+, vertex_ai_endpoints/langfuse_endpoints.py:160+, llm_passthrough_endpoints.py:3090+)
- `/mcp`, `/mcp/proxy`, `/{mcp_server_name}/mcp`, `/toolset/{toolset_name}/mcp` — MCP protocol routes validate tokens internally and 403 on failure (proxy_server.py)
- `/realtime/calls`, `/v1/realtime/calls`, `/realtime/translations/calls` — own ephemeral-token auth (decrypt + validate, 401 on failure)
- `/v1/mcp/oauth/authorize` (POST) — `_byok_session_auth` dependency

**Public by design (OAuth / discovery / health / static):**
- OAuth: `/oauth/token`, `/oauth/revoke`, `/oauth/device_authorization`, `/oauth/client-metadata.json`, `/register`, `/{mcp_server_name}/register|token|authorize`, `/revoke`, `/v1/mcp/oauth/*`, `/claude_code_gateway/oauth/*`, `/.well-known/*`
- SSO/login: `/login`, `/v2/login`, `/v3/login`, `/v3/login/exchange`, `/sso/*` (incl. `/sso/key/generate` = Google OAuth login flow), `/onboarding/*`, `/fallback/login`
- Health: `/health/liveliness`, `/health/liveness`, `/health/readiness`, `/health/drain`
- Public info: `/public/*` (model hub, providers, skill hub, cost map), `/get/ui_settings`, `/get/ui_theme_settings`, `/get_favicon|image|logo_url`, `/v1/mcp/registry.json` (gated by `_is_public_registry_enabled()` + IP filtering), `/litellm/.well-known/litellm-ui-config`
- Public downloads: `/claude-code/marketplace.json`, `/claude-code/plugins/{name}/archive/{digest}.zip`, `/skill-hub/skills/{name}/archive|content`, `/v1/skills/{skill_id}/archive`
- Stubs: `/v1/logs`, `/v1/traces` POST → 410 Gone (deprecated OTLP ingestion)

**Conclusion:** the missing-auth class is fully covered in the current codebase. No unproven/theoretical finding was written up, per the program's own rule ("Highly speculative reports about theoretical damage" are ineligible) and the hunt rules (drop anything unproven).

## Next session options (same program)

1. **P0 supply chain ($1.5k–$3k):** audit `.github/workflows/*.yml` for `pull_request_target` + checkout of PR head, action/script injection via `${{ github.event.* }}`, artifact poisoning between build and publish, release-triggered workflows with untrusted input. This is the highest-paying class and CI config is small and fully reviewable from the clone.
2. **P1 deeper:** auth-bypass variants on the authenticated routes (e.g. `user_api_key_auth` edge cases: key format confusion, JWT/OIDC paths, team/user RBAC boundary conditions on management endpoints).
3. **P2 (unpaid but reputation):** authenticated privilege escalation — skip unless building signal.

## Submission readiness (for when a finding exists)

Per the owner's standing rule, the owner submits after his own review. Channel: GitHub private vulnerability report at https://github.com/BerriAI/litellm/security/advisories/new (requires the owner's GitHub login — his account, his submission). Report must include: summary, affected component, reproduction steps, **reproduction video**, impact, and the GitHub username for credit.
