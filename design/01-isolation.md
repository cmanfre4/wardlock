# Isolation

The agent runs in an isolated environment (container or VM) that only has access to what the current task requires, rather than inheriting the operator's full host credentials and filesystem.

## Isolation Providers

Isolation is modular, providing consistent semantics across varying container and VM technologies.

- **Localhost** — no actual isolation. The agent runs directly on the operator's machine. This is the starting point: it lets the credential brokering loop work end-to-end without container complexity, and serves as a skeleton/reference implementation for the isolation provider interface.
- **Docker / Devcontainers** — low overhead, widely understood, good CLI/IDE integration. The first real isolation provider, providing filesystem and credential isolation from the host.
- **Kubernetes (EKS)** — more relevant for multi-agent or cloud-native scenarios. Better suited as a target provider in the modular layer rather than the starting point.
- **Remote VMs** — for scenarios where local resources are insufficient or stronger isolation guarantees are needed.

## What Isolation Provides

- A shell/environment with only the CLI tools needed for the task (kubectl, tsh, vault, terraform, gh, aws, etc.)
- No direct access to the host filesystem beyond explicitly mounted project directories
- No inherited credentials from the host — all credentials are injected by the credential broker

## Communication

There are two distinct communication channels:

**Agent-to-broker** (resolved): The agent communicates with the credential broker via MCP. The broker exposes `request_access`, `revoke`, and `list_active` as MCP tools. This is detailed in the [Credential Brokering](02-credential-brokering.md) and Agent-to-Broker Communication sections.

**Operator-facing** (open): The operator needs visibility into the agent's work and the ability to act on approval prompts. This channel must support:

- Real-time output streaming from the agent
- Approval prompts that the operator can act on (Tier 4 human approvals, out-of-budget escalations)
- Credential request notifications
- Task status and progress

The operator-facing channel is unresolved. Options include a local web UI, terminal integration, or Slack/webhook notifications. In solo mode, the operator is already at their terminal, so direct terminal integration may be sufficient. In organizational mode, asynchronous notification channels become more important.
