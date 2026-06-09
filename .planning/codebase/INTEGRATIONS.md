# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**Cinatra Platform (A2A dispatch):**
- This agent is dispatched synchronously via Agent-to-Agent (A2A) by three internal Cinatra platform callers: `agent_source_lint`, `agent_source_publish`, and `agent_source_compile`.
- SDK/Client: Host-internal Cinatra runtime (not a public SDK; runs only within the Cinatra monorepo/platform).
- Auth: Inherited from the platform execution context; no explicit API key in this repo.

**Cinatra Marketplace:**
- Used for publishing this extension.
- Submission path: GitHub Release → `cinatra-ai/.github` reusable workflow → marketplace MCP proxy → `extension-submit-for-review` → registry.cinatra.ai
- Auth: `CINATRA_MARKETPLACE_VENDOR_TOKEN` org secret (GitHub Actions; secret exists at org level — not in this repo)

## Data Storage

**Databases:**
- Not applicable — this agent is stateless and purely deterministic. It receives an OAS JSON payload and returns a `ReviewFinding[]` array. No database connection.

**File Storage:**
- Not applicable.

**Caching:**
- None.

## Authentication & Identity

**Auth Provider:**
- Not applicable for runtime operation. The agent runs within the Cinatra platform's trusted execution context.
- CI/CD: GitHub OIDC (`id-token: write` permission) is used for build-provenance attestation during marketplace release (`.github/workflows/release.yml`).

## Monitoring & Observability

**Error Tracking:**
- Not detected — no error tracking SDK (e.g., Sentry, Datadog) configured in this repo.

**Logs:**
- CI gate (`extension-kind-gate.mjs`) writes to stdout/stderr via `console.log` / `console.error`. Agent execution logs are handled by the Cinatra platform runtime.

## CI/CD & Deployment

**Hosting:**
- Cinatra Marketplace / `registry.cinatra.ai`

**CI Pipeline:**
- GitHub Actions — `.github/workflows/ci.yml`
  - Triggers: push and pull_request to `main`
  - Node 24 + corepack/pnpm
  - Steps: classify repo (first-party peer detection), install (skipped for source mirrors), typecheck (skipped for source mirrors), test (skipped for source mirrors), `npm pack --dry-run`, agent OAS validation gate via `node extension-kind-gate.mjs --package-root .`
- GitHub Actions — `.github/workflows/release.yml`
  - Triggers: GitHub Release published, or manual `workflow_dispatch` against a tag
  - Delegates to reusable workflow: `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main`
  - Permissions: `contents: read`, `id-token: write`, `attestations: write`

## Environment Configuration

**Required env vars:**
- None required at runtime in this repo. The agent receives all inputs via A2A protocol fields (`oasJson`, `packageJson`, `packageSlug`, `policyVersion`).

**Secrets location:**
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — GitHub org-level secret, inherited by the release workflow via `secrets: inherit`

## Webhooks & Callbacks

**Incoming:**
- Not applicable — this agent is invoked synchronously via Cinatra's A2A dispatch, not via HTTP webhooks.

**Outgoing:**
- Not detected — the agent returns findings inline to the caller; no outgoing webhooks configured.

---

*Integration audit: 2026-06-09*
