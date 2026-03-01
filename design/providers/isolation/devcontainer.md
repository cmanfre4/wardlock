# Devcontainer Isolation Provider

Docker-based isolation using the devcontainer specification. Low overhead, widely understood, good CLI/IDE integration. The first real isolation provider, providing filesystem and credential isolation from the host.

## What It Provides

- A shell/environment with only the CLI tools needed for the task (kubectl, tsh, vault, terraform, gh, aws, etc.)
- No direct access to the host filesystem beyond explicitly mounted project directories
- No inherited credentials from the host — all credentials are injected by the credential broker

## Supported Injection Types

| Injection Type | Implementation |
|---|---|
| `file` | Write to bind-mounted shared volume |
| `env` | Container env config or wrapper script |

## Configuration

The framework generates a `devcontainer.json` from the task manifest, mapping credential providers to devcontainer features (e.g., GitHub provider requires `git`, Kubernetes/Teleport provider requires `kubectl` + `tsh`).

## Communication

The broker's MCP server is made available inside the container via a Unix socket bind-mounted into the container. The agent CLI discovers it like any other MCP server.

## Backend Identity

| Adoption Phase | Identity Source |
|---|---|
| Solo | Operator's Docker socket |
| Organizational | Docker API TLS (client cert) or service account |

## Adoption Phase Relevance

The primary isolation provider from Adoption Phase 2 onward. Also used in solo mode when the operator wants real isolation without moving to a remote broker.
