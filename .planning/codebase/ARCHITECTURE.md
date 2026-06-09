<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│              A2A Callers (agent_source_lint,                  │
│              agent_source_publish, agent_source_compile)      │
└────────────────────────┬────────────────────────────────────┘
                         │ A2A dispatch (synchronous)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Cinatra Flow (OAS definition)               │
│                  `cinatra/oas.json`                          │
│  StartNode → scan_all (ApiNode) → OutputMessageNode → EndNode│
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP POST
                         ▼
┌─────────────────────────────────────────────────────────────┐
│   Platform API: {{CINATRA_BASE_URL}}/api/oas-lint/scan-all   │
│   (Runs: secrets scan, URL trust, llm-bridge wiring,        │
│    llm metadata drift, StartNode gate, version sync,        │
│    license check)                                           │
└─────────────────────────────────────────────────────────────┘
                         │ returns findings JSON
                         ▼
┌─────────────────────────────────────────────────────────────┐
│   OutputMessageNode: emits {"findings": [...]}              │
│   EndNode: exposes findings as output binding               │
└─────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| OAS Flow definition | Declares the agent's input/output contract and node graph | `cinatra/oas.json` |
| StartNode (`start`) | Accepts inputs: oasJson, packageJson, packageSlug, policyVersion, agent_run_id | `cinatra/oas.json` |
| ApiNode (`scan_all`) | POSTs all inputs to the platform lint API and retrieves findings | `cinatra/oas.json` |
| OutputMessageNode (`emit_output`) | Formats findings as a JSON agent message | `cinatra/oas.json` |
| EndNode (`end`) | Exposes `findings` as the agent's output binding | `cinatra/oas.json` |
| Extension kind gate | Standalone CI pre-publish sanity validator (agent OAS parse + banned-primitive scan; workflow BPMN shape check) | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Cinatra declarative Flow agent — no imperative runtime code; logic lives in the platform API behind an ApiNode.

**Key Characteristics:**
- The agent is a pure Flow definition (OAS JSON); all scan logic executes server-side at `{{CINATRA_BASE_URL}}/api/oas-lint/scan-all`
- Zero runtime TypeScript in this repo — `tsconfig.json` targets a `src/` directory that does not exist yet
- The only code asset is `extension-kind-gate.mjs`: a self-contained Node.js CI gate that uses no `@cinatra-ai/*` packages and no external dependencies (Node builtins only)
- The agent is the sole authorised emitter of `severity: "blocker"` findings across the entire Cinatra platform

## Layers

**Flow Definition Layer:**
- Purpose: Declares the agent contract — inputs, outputs, node graph, control-flow edges, data-flow edges
- Location: `cinatra/oas.json`
- Contains: StartNode, ApiNode, OutputMessageNode, EndNode, all edge wiring
- Depends on: Cinatra platform runtime (`{{CINATRA_BASE_URL}}`)
- Used by: A2A callers (`agent_source_lint`, `agent_source_publish`, `agent_source_compile`)

**CI Gate Layer:**
- Purpose: Pre-publish sanity checks run in extracted-repo CI, zero dependencies
- Location: `extension-kind-gate.mjs`
- Contains: `parseArgs`, `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `main`
- Depends on: Node.js builtins only (`fs`, `path`)
- Used by: `.github/workflows/ci.yml` (`kind-gates` job), can be invoked directly via `node extension-kind-gate.mjs --package-root .`

## Data Flow

### Primary Request Path (agent runtime)

1. A2A caller dispatches to agent with `oasJson` (required), optional `packageJson`, `packageSlug`, `policyVersion` — StartNode (`cinatra/oas.json` → `$referenced_components.start`)
2. `scan_all` ApiNode POSTs all five fields to `{{CINATRA_BASE_URL}}/api/oas-lint/scan-all` (`cinatra/oas.json` → `$referenced_components.scan_all`)
3. Platform API returns `findings` (JSON array of `ReviewFinding[]`)
4. `emit_output` OutputMessageNode emits `{"findings": <findings>}` as agent chat message
5. `end` EndNode exposes `findings` as the agent's output binding for A2A callers

### CI Gate Path (pre-publish)

1. `extension-kind-gate.mjs` — `parseArgs` resolves `--package-root` (defaults to cwd)
2. `runGate` reads `package.json`, dispatches to `validateAgent` (kind=`"agent"`) or `validateWorkflow` (kind=`"workflow"`)
3. For agents: `validateAgent` checks if `cinatra/oas.json` exists, parses it, walks all LLM-visible string fields (`system`, `user`, `description`), scans for banned CRM primitive names and legacy entity typeHints
4. Errors are printed and exit code 1 is returned; clean pass prints confirmation and exits 0

**State Management:**
- Stateless — no persistent state; every invocation is a fresh scan

## Key Abstractions

**ReviewFinding[]:**
- Purpose: Normalized finding records with severity (including `"blocker"`), code, message, and field pointer
- Produced by: Platform API at `{{CINATRA_BASE_URL}}/api/oas-lint/scan-all`
- Consumed by: A2A callers (`agent_source_lint`, `agent_source_publish`, `agent_source_compile`)

**`cinatra.kind` dispatch:**
- Purpose: The CI gate branches on `package.json → cinatra.kind` (`"agent"` | `"workflow"` | other) to apply the correct validation path
- Examples: `extension-kind-gate.mjs` → `runGate`

**Banned primitives list:**
- Purpose: Registry of retired CRM primitive names and legacy entity typeHints that must not appear in LLM-visible OAS string fields
- Location: `extension-kind-gate.mjs` — `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`

## Entry Points

**Agent runtime entry:**
- Location: `cinatra/oas.json` → `start_node.$component_ref: "start"`
- Triggers: A2A dispatch from `agent_source_lint`, `agent_source_publish`, `agent_source_compile`
- Responsibilities: Accept OAS + package inputs, delegate all scanning to the platform API, return findings

**CI gate CLI entry:**
- Location: `extension-kind-gate.mjs` → `main()` (runs when invoked directly)
- Triggers: `node extension-kind-gate.mjs --package-root .` in `.github/workflows/ci.yml` (`kind-gates` job)
- Responsibilities: Validate the extension package at the given root, exit 0 (pass) or 1 (fail)

## Architectural Constraints

- **No runtime TypeScript:** `tsconfig.json` references a `src/` directory that does not exist. All logic is either the platform API (external) or `extension-kind-gate.mjs` (pure JS ESM, Node builtins only).
- **Zero @cinatra-ai/* dependencies:** `package.json` declares `"dependencies": []`. The CI gate must remain dependency-free so it runs unauthenticated before the private registry is reachable.
- **Source mirror classification:** CI detects `@cinatra-ai/*` optional peers and skips standalone install/typecheck/test; the monorepo owns those steps. This repo currently has no peers, classifying it as standalone.
- **Singleton blocker authority:** This agent is the only one on the platform allowed to emit `severity: "blocker"`.
- **Global state:** None — `extension-kind-gate.mjs` uses only module-level constant arrays (`BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`) that are read-only.
- **Circular imports:** None — single file gate.

## Anti-Patterns

### Registering first-party deps as dependencies/devDependencies

**What happens:** `@cinatra-ai/*` packages appear in `dependencies` or `devDependencies` instead of optional `peerDependencies`.
**Why it's wrong:** Standalone extracted-repo CI runs unauthenticated; the private registry is unreachable, so install fails closed.
**Do this instead:** Declare `@cinatra-ai/*` packages as `peerDependencies` with `peerDependenciesMeta.<pkg>.optional: true`. CI enforces this in `ci.yml` (the "Classify repo" step).

### Using `pnpm dlx` for CI tools in extracted repos

**What happens:** A CI step runs `pnpm dlx @cinatra-ai/extension-tools` (the retired pattern).
**Why it's wrong:** `pnpm dlx` resolves the binary from the registry; fails before registry auth is established; `pnpm dlx` also errors on packages with multiple bins.
**Do this instead:** Use the self-contained `extension-kind-gate.mjs` shipped in the repo: `node extension-kind-gate.mjs --package-root .`

## Error Handling

**Strategy:** Collect-and-return — validation functions accumulate string error messages into an `errors: string[]` array and return it; no exceptions are thrown for business logic violations.

**Patterns:**
- I/O errors (file not found, JSON parse failures) are caught with try/catch and pushed as formatted strings into `errors`
- Missing optional files (e.g., `cinatra/oas.json` for agents) are treated as clean (no error); marketplace-side validation owns the "must exist" contract
- CI entry point wraps `main()` in a top-level try/catch and exits 1 on unexpected errors

## Cross-Cutting Concerns

**Logging:** `console.log` for success, `console.error` for violations — no logging library.
**Validation:** Pure functions returning `string[]`; no exceptions for business errors.
**Authentication:** Not applicable at the gate layer — CI gate is intentionally unauthenticated. Agent runtime auth is handled by the Cinatra platform at the ApiNode call.

---

*Architecture analysis: 2026-06-09*
