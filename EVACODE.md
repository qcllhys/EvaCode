# EvaCode repository instructions

## Architecture

- Keep UI, model clients, tool execution and permission decisions decoupled.
- Normalize provider-specific streaming events before they reach the Agent runtime.
- Preserve tool call and tool result pairing when transforming conversation history.
- Keep external integrations behind adapters and avoid provider logic in core modules.

## Safety invariants

- Every tool must declare the correct `read`, `write` or `command` category.
- File writes and shell commands must pass through `PermissionChecker`.
- Never place credentials, private endpoints or machine-specific paths in tracked files.
- Remote and MCP capabilities must fail closed when configuration is invalid.

## Validation

- Add the smallest relevant regression test for behavior changes.
- Run `uv run pytest -q` before completing repository-wide changes.
- Run `uv build` when package metadata or distribution files change.
