# Coding Conventions

**Analysis Date:** 2026-06-09

## Overview

This repo is a Cinatra agent extension (source mirror). The primary implementation file is `extension-kind-gate.mjs` — a self-contained ESM script using only Node.js builtins. There is no `src/` directory; the agent's runtime logic lives in `cinatra/oas.json`.

## Naming Patterns

**Files:**
- `kebab-case` for script files: `extension-kind-gate.mjs`
- `camelCase` for JSON data keys and OAS identifiers: `oasJson`, `packageSlug`, `agentspec_version`
- `snake_case` for OAS node IDs and flow connection names: `scan_all`, `start_to_scan`, `agent_run_id`

**Functions:**
- `camelCase` for all exported and internal functions: `parseArgs`, `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `walkLlmStrings`, `scanOasString`
- Short, verb-noun form: `parseArgs`, `runGate`, `findWorkflowSidecars`

**Variables:**
- `camelCase` for local variables: `packageRoot`, `bpmnPrefixes`, `openTags`, `rootAttrs`
- `SCREAMING_SNAKE_CASE` for module-level constants: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`

**Types:**
- No explicit TypeScript types in this file (it is `.mjs`). `tsconfig.json` references a `src/` directory for TypeScript sources not present in this extracted repo.

## Code Style

**Formatting:**
- No Prettier or ESLint config detected in this extracted repo. Style is enforced by the monorepo toolchain during extraction.
- Indentation: 2 spaces throughout `extension-kind-gate.mjs`
- Trailing commas in multi-line arrays and objects

**Linting:**
- Not detected (no `.eslintrc*`, `biome.json`, or `.prettierrc` present)

## Module System

**Type:** ESM (`"type": "module"` in `package.json`)
- All imports use named `import { ... } from "node:fs"` style with `node:` protocol prefix
- No CommonJS `require()` except inside inline Node `-e` scripts in CI (`ci.yml` uses `require()` in shell one-liners targeting Node's CJS REPL mode)

## Import Organization

**Order in `extension-kind-gate.mjs`:**
1. Node built-in modules only (no third-party imports): `node:fs`, `node:path`

**Path Aliases:**
- None (zero-dependency design constraint — must run unauthenticated before registry is reachable)

## Error Handling

**Patterns:**
- Functions are pure: they return `string[]` error arrays rather than throwing. Example: `validateAgent`, `validateWorkflow`, `validateBpmnSanity` all return `string[]`.
- File I/O is wrapped in `try/catch`; caught errors are formatted as `err instanceof Error ? err.message : String(err)` and pushed into the error array.
- Early returns from error arrays (`return errors`) are used when a subsequent check would be invalid (e.g., returning after parse failure before attempting to walk the parsed value).
- `main()` catches all unexpected errors and exits with code 1 via a top-level try/catch.
- Exit codes: `0` = pass, `1` = violation(s), `2` = dependency-shape regression (CI only).

## Comments

**When to Comment:**
- Section headers use `// ---...--- ` banners to delimit logical sections within the file.
- Complex regex patterns are accompanied by inline comments explaining what they match.
- Functions export with JSDoc-style block comments (`/** ... */`) describing purpose, purity, and return type.
- CI YAML steps have multi-line inline comments (`#`) explaining the skip logic and rationale.

**JSDoc:**
- Used selectively on exported functions (`validateAgent`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`).
- Describes purity: "Pure: returns string[] errors" or "Pure-ish: returns string[]".

## Function Design

**Size:** Functions are small and focused. The largest (`validateBpmnSanity`) is ~80 lines handling a single concern (XML well-formedness + BPMN shape check).

**Parameters:** Single `packageRoot: string` parameter for validator functions; `(xml: string)` for pure string processors.

**Return Values:** Consistent `string[]` error arrays from all validator functions; `{ kind, errors }` object from `runGate`.

## Module Design

**Exports:** Named exports only — `parseArgs`, `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate` are all exported for testability.

**Entry Guard:** The file uses `invokedDirectly` check (`resolve(process.argv[1]) === resolve(new URL(import.meta.url).pathname)`) before calling `main()`, following the Node.js ESM "main module" pattern. This allows the file to be imported for testing without side effects.

**Barrel Files:** Not applicable (single-file implementation).

## Self-Containment Constraint

**Critical design rule:** `extension-kind-gate.mjs` MUST remain zero-dependency (Node builtins only). This is documented in the file header and enforced by the CI design — the gate runs unauthenticated before any registry is available.

---

*Convention analysis: 2026-06-09*
