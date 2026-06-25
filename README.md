# Lint Policy Agent

Lint Policy Agent runs Cinatra platform policy scanners against an agent definition and returns a structured findings list. Dispatch it synchronously via A2A (agent-to-agent) by invoking `agent_source_lint`, `agent_source_publish`, or `agent_source_compile`. Pass it an OAS Flow body as `oasJson` (required), and optionally a `packageJson` string and a `packageSlug` for version-sync checks. It calls `/api/oas-lint/scan-all` on your Cinatra instance and returns a `findings` JSON array where each entry carries a `severity`, `code`, `message`, and optional field pointer. A `severity: "blocker"` finding means the checked operation — save, compile, or publish — must not proceed. If the API call fails or returns no findings, treat it as a scan error and halt. The agent requires a running Cinatra instance reachable at `CINATRA_BASE_URL` and an authenticated A2A session; it has no additional credentials of its own.

## Works with

- Cinatra platform (dispatched via A2A by `agent_source_lint`, `agent_source_publish`, `agent_source_compile`)

## Capabilities

- Scan an agent definition for hardcoded secrets and credential leaks
- Flag untrusted URLs and misconfigured tool wiring
- Catch missing required fields and version mismatches between spec and manifest
- Verify the package license is present and well-formed
- Return a normalized findings list with severity, code, message, and field pointer
- Mark blocker-severity findings that must halt save, compile, or publish
