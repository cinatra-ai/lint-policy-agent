# Testing Patterns

**Analysis Date:** 2026-06-09

## Overview

This repo is an extracted Cinatra agent source mirror. It declares `cinatra.dependencies: []` (no host-internal `@cinatra-ai/*` peers), making it a **standalone repo** in CI terms. No test files are tracked in this extracted repo — the monorepo owns the test suite for the underlying agent implementation. The CI gate (`ci.yml`) runs `pnpm test --if-present`, which exits 0 when no `test` script is defined.

## Test Framework

**Runner:** Not detected (no `jest.config.*`, `vitest.config.*`, or `test` script in `package.json`)

**Assertion Library:** Not detected

**Run Commands:**
```bash
corepack pnpm test --if-present   # CI command — exits 0 if no test script defined
node extension-kind-gate.mjs --package-root .   # Manual gate invocation
```

## Test File Organization

**Location:** No test files present in this extracted repo.

**Naming:** Not applicable.

## What Is Verified in CI

Although there are no test files, `ci.yml` enforces several correctness properties:

1. **Dependency-shape gate** — inline Node `-e` script validates that no `@cinatra-ai/*` or `@cinatra/*` packages appear in `dependencies`/`devDependencies`/`optionalDependencies`, and that any first-party peer is marked `peerDependenciesMeta.optional`.

2. **Typecheck** — `tsc --noEmit` (via `tsconfig.json`) when TypeScript sources are present. Currently skipped because no `.ts`/`.tsx` files are tracked.

3. **Pack dry-run** — `npm pack --dry-run` validates publishable package shape.

4. **Agent OAS validation gate** — `node extension-kind-gate.mjs --package-root .` scans `cinatra/oas.json` for retired CRM primitives in LLM-visible fields (`system`, `user`, `description`). This is the authoritative pre-publish correctness check for this repo.

## Testability Design of `extension-kind-gate.mjs`

The gate file is structured for testability despite having no test suite in this repo:

- All validator functions are **pure** and **exported**: `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate`, `parseArgs`.
- `main()` is only called when the file is invoked directly (guarded by the `invokedDirectly` check), so importing the module for unit tests produces no side effects.
- Error output is `string[]` arrays, not thrown exceptions, making assertion straightforward.

**Example of how tests could be structured (based on exported API):**
```javascript
import { validateBpmnSanity } from "./extension-kind-gate.mjs";

// Happy path
assert.deepEqual(validateBpmnSanity(validBpmnXml), []);

// Error path
const errs = validateBpmnSanity("<notbpmn/>");
assert(errs.some(e => e.includes("not a BPMN document")));
```

## Mocking

**Framework:** Not applicable (no test suite).

**What would need mocking in a test suite:**
- `readFileSync` / `existsSync` from `node:fs` — the validators call these for file I/O. A test suite would either use real temp files or mock the `node:fs` module.
- `readdirSync` — used by `findWorkflowSidecars` to walk directories.

**What NOT to mock:**
- The regex/parsing logic in `validateBpmnSanity` and `scanOasString` — these are pure string-in/array-out and should be tested with real inputs.

## Coverage

**Requirements:** None enforced (no coverage tooling configured).

**Gaps:** The entire `extension-kind-gate.mjs` logic — retired-primitive scanning, BPMN sanity checking, workflow package shape validation — has no automated test coverage in this extracted repo. Coverage is owned by the monorepo.

## Test Types

**Unit Tests:** Not present in this repo. The monorepo contains unit tests for the underlying scanner logic.

**Integration Tests:** Not present. The CI `extension-kind-gate.mjs` invocation (`node extension-kind-gate.mjs --package-root .`) acts as a lightweight integration smoke test against the actual `cinatra/oas.json`.

**E2E Tests:** Not used.

## CI Gate as Functional Verification

The most meaningful "test" in this repo is the CI agent OAS validation gate in `.github/workflows/ci.yml` (job `kind-gates`, step "Agent OAS validation gate"):

```bash
node extension-kind-gate.mjs --package-root .
```

This:
- Parses `cinatra/oas.json`
- Walks all LLM-visible string fields (`system`, `user`, `description`)
- Asserts no banned primitive tokens (e.g., `lists_list`, `contacts_get`) appear
- Asserts no banned typeHints (e.g., `@cinatra-ai/entity-accounts:account`) appear
- Exits 0 (pass) or 1 (violation)

---

*Testing analysis: 2026-06-09*
