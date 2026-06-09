# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- JavaScript (ES Modules) — `extension-kind-gate.mjs` (zero-dependency CI gate, Node builtins only)
- TypeScript — configured via `tsconfig.json` targeting `src/` (no `src/` directory present in this extracted mirror; TS config is present for monorepo integration)
- JSON — agent OAS spec at `cinatra/oas.json`

**Secondary:**
- YAML — GitHub Actions workflows at `.github/workflows/ci.yml` and `.github/workflows/release.yml`

## Runtime

**Environment:**
- Node.js 24 (pinned in CI via `actions/setup-node@v4`, `node-version: "24"`)

**Package Manager:**
- pnpm (managed via corepack — `corepack enable` / `corepack pnpm`)
- `.npmrc` file present — note existence only; contents not read
- No committed lockfile (CI uses `--no-frozen-lockfile`)

## Frameworks

**Core:**
- None — this is a source-mirror/extracted extension repo. The agent runtime logic is defined declaratively in `cinatra/oas.json` (Cinatra agentspec v26.1.0 OAS Flow format). Executable implementation lives in the Cinatra monorepo.

**Testing:**
- Not detected — no test script, test files, or test framework config present in this repo. Tests run in the monorepo.

**Build/Dev:**
- TypeScript compiler (`tsc`) — configured in `tsconfig.json`, targets `ES2023`, `ESNext` modules, `bundler` resolution, outputs to `dist/`

## Key Dependencies

**Critical:**
- No runtime dependencies declared in `package.json`. `cinatra.dependencies: []`
- All `@cinatra-ai/*` packages are host-internal and only available within the Cinatra monorepo workspace. They are NOT published to any registry and are not declared as dependencies in this extracted repo.

**Infrastructure:**
- `extension-kind-gate.mjs` — self-contained, zero-dependency CI gate using only Node built-ins (`node:fs`, `node:path`). Performs OAS retired-primitive scan for `kind: "agent"` extensions and BPMN shape validation for `kind: "workflow"` extensions.

## Configuration

**Environment:**
- No `.env` files detected. Agent inputs are passed at runtime via the Cinatra A2A dispatch protocol (see `cinatra/oas.json` inputs).
- Required agent input: `oasJson` (string, required). Optional inputs: `agent_run_id`, `packageJson`, `packageSlug`, `policyVersion`.

**Build:**
- `tsconfig.json` — standalone strict TypeScript config, target ES2023, moduleResolution bundler, strict mode, verbatimModuleSyntax, outputs to `dist/`, roots in `src/`
- `package.json` — ESM package (`"type": "module"`), `cinatra.apiVersion: "cinatra.ai/v1"`, `cinatra.kind: "agent"`

## Platform Requirements

**Development:**
- Node.js 24
- pnpm (via corepack)
- Source mirrors with `@cinatra-ai/*` peers are built and typechecked only within the Cinatra monorepo (not standalone-installable)

**Production:**
- Deployed via the Cinatra Marketplace through the `registry.cinatra.ai` registry
- Release pipeline: GitHub Release tag → `cinatra-ai/.github` reusable workflow → marketplace MCP proxy → `extension-submit-for-review` → promotion saga
- Requires org secret `CINATRA_MARKETPLACE_VENDOR_TOKEN` (managed by Cinatra org infra)

---

*Stack analysis: 2026-06-09*
