# Localhost Isolation Provider

No actual isolation. The agent runs directly on the operator's machine. This is the starting point: it lets the credential brokering loop work end-to-end without container complexity, and serves as a skeleton/reference implementation for the isolation provider interface.

## What It Provides

- A working isolation provider interface for the broker to target.
- File injection via direct filesystem writes.
- Validation of the full credential lifecycle without container overhead.

## What It Does Not Provide

- No filesystem isolation — the agent has access to the operator's full filesystem.
- No credential isolation from the host — host credentials are visible.
- No network isolation.

## Supported Injection Types

| Injection Type | Implementation |
|---|---|
| `file` | Write directly to filesystem |
| `env` | Export in shell profile |

## Backend Identity

Implicit — the broker runs as the operator's OS user. No authentication needed.

## Adoption Phase Relevance

Used in Adoption Phase 1 (solo practitioner, local broker). Replaced by real isolation providers as deployment matures.

The localhost provider is architecturally important: it implements the full isolation provider interface so the broker's injection code paths are exercised from day one, even before containers are involved.
