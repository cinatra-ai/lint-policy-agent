# Lint Policy Agent

Run the platform's policy scanners against an agent definition and report what they find. The agent vets a definition for hardcoded secrets, untrusted URLs, missing required fields, version-sync issues, and licensing problems, and returns a structured list of findings — including the blockers that should stop a save, compile, or publish.

## Capabilities

- Scan an agent definition for hardcoded secrets and credential leaks
- Flag untrusted URLs and misconfigured tool wiring
- Catch missing required fields and version mismatches between spec and manifest
- Verify the package's license is present and well-formed
- Return a structured findings list with severity, code, message, and field pointer
- Mark hard-blocker findings that should stop save, compile, or publish
