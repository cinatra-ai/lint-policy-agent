# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
lint-policy-agent/
├── cinatra/                  # Cinatra platform extension artifacts
│   └── oas.json              # Agent Flow definition (inputs, outputs, node graph)
├── .github/
│   └── workflows/
│       ├── ci.yml            # Standalone CI: classify, typecheck, test, kind-gate
│       └── release.yml       # Marketplace publish via reusable org workflow
├── .planning/
│   └── codebase/             # GSD codebase map documents (this directory)
├── extension-kind-gate.mjs   # Zero-dependency CI validator (agent OAS + workflow BPMN)
├── package.json              # Package manifest with cinatra.kind metadata
├── tsconfig.json             # TypeScript config targeting src/ (not yet present)
├── .npmrc                    # pnpm: auto-install-peers=false
├── LICENSE                   # Apache-2.0
└── README.md                 # Agent overview and capabilities
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Cinatra platform sidecar artifacts required for the extension marketplace
- Contains: `oas.json` — the agent's full Flow definition including node graph, data-flow edges, I/O schema
- Key files: `cinatra/oas.json`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI and release automation
- Contains: `ci.yml` (build + kind gate), `release.yml` (marketplace submission trigger)
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

**`.planning/codebase/`:**
- Purpose: GSD codebase map documents consumed by plan and execute commands
- Contains: ARCHITECTURE.md, STRUCTURE.md
- Generated: No (written by gsd-map-codebase)
- Committed: Yes

## Key File Locations

**Entry Points:**
- `cinatra/oas.json`: Agent Flow definition — the runtime entry point for A2A dispatch
- `extension-kind-gate.mjs`: CLI entry point for CI pre-publish validation (`node extension-kind-gate.mjs --package-root .`)

**Configuration:**
- `package.json`: Package name, version, license, `cinatra.kind` = `"agent"`, `cinatra.dependencies` = `[]`
- `tsconfig.json`: Standalone strict TS config — targets `src/` (does not exist yet), outputs to `dist/`
- `.npmrc`: `auto-install-peers=false`

**Core Logic:**
- `extension-kind-gate.mjs`: All CI gate logic — arg parsing, agent OAS validation, workflow BPMN validation, dispatch

**Agent Definition:**
- `cinatra/oas.json`: Complete Cinatra Flow spec — `agentspec_version: 26.1.0`, `component_type: Flow`, node graph with StartNode → ApiNode → OutputMessageNode → EndNode

**CI/CD:**
- `.github/workflows/ci.yml`: Classify repo (source mirror vs standalone), install, typecheck, test, npm pack dry-run, kind gate
- `.github/workflows/release.yml`: Triggers reusable org release workflow on GitHub Release published event

## Naming Conventions

**Files:**
- Cinatra platform artifacts: `cinatra/<artifact-type>.json` (e.g., `cinatra/oas.json`)
- CI gate scripts: `extension-kind-gate.mjs` — kebab-case `.mjs` (ESM, zero dependencies)
- GitHub workflows: kebab-case `.yml` (e.g., `ci.yml`, `release.yml`)

**Directories:**
- Platform artifacts: `cinatra/` (reserved name, lowercase)
- Tooling: `.github/`, `.planning/` (dotfile convention for non-source directories)

## Where to Add New Code

**New source TypeScript (if any logic moves into this repo):**
- Implementation: `src/` (create directory; `tsconfig.json` already targets `src/**/*.ts`)
- Tests: `src/__tests__/` or co-located `*.test.ts`

**Additional Cinatra platform artifacts:**
- BPMN workflow (if kind changes to `"workflow"`): `cinatra/workflow.bpmn`
- OAS updates: `cinatra/oas.json` (only file — do not add other JSON to `cinatra/`)

**Additional CI gates:**
- Add steps to the `kind-gates` job in `.github/workflows/ci.yml`
- All gate logic must remain in `extension-kind-gate.mjs` (zero-dependency, Node builtins only)

**Utilities:**
- Self-contained gate helpers: append exported functions to `extension-kind-gate.mjs`
- If helper complexity grows: introduce `src/` and import via ESM (requires adjusting CI skip logic)

## Special Directories

**`cinatra/`:**
- Purpose: Platform-required extension sidecar artifacts
- Generated: `oas.json` is maintained by the agent author / extraction tooling
- Committed: Yes — the OAS is the agent's deployable artifact

**`dist/`:**
- Purpose: TypeScript compiler output (declared in `tsconfig.json`)
- Generated: Yes (by `tsc`)
- Committed: No (not yet present; no `src/` exists)

**`.planning/`:**
- Purpose: GSD planning and codebase map documents
- Generated: By GSD tooling
- Committed: Yes

---

*Structure analysis: 2026-06-09*
