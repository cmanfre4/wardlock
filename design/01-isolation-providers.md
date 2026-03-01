# Isolation Providers

The agent runs in an isolated environment (container or VM) that only has access to what the current task requires, rather than inheriting the operator's full host credentials and filesystem.

## First-Party Providers

Isolation is modular, providing consistent semantics across varying container and VM technologies.

| Provider | Description | Spec |
|---|---|---|
| Localhost | No actual isolation. Reference implementation for the provider interface. | localhost.md |
| Devcontainer | Docker-based, low overhead, good CLI/IDE integration. First real isolation provider. | devcontainer.md |
| Kubernetes | Agent containers as K8s pods. Multi-agent and cloud-native scenarios. | kubernetes.md |
| Remote VM | Full VM-level isolation for stronger security guarantees. | remote-vm.md |

## What Isolation Provides

- A shell/environment with only the CLI tools needed for the task (kubectl, tsh, vault, terraform, gh, aws, etc.)
- No direct access to the host filesystem beyond explicitly mounted project directories
- No inherited credentials from the host — all credentials are injected by the credential broker

## Provider Contract

Each isolation provider implements a consistent interface that the broker calls to manage isolated environments and deliver credentials into them.

### Environment Lifecycle

- `setup(config)` → environment handle — creates and starts the isolated environment. The `config` includes what CLI tools to install, what volumes to mount, and how to make the broker's MCP server available inside the environment.
- `teardown(handle)` — stops and destroys the environment. Triggers credential revocation for all active credentials scoped to this environment.

### Credential Injection

- `inject(handle, bundle)` → injection result — materializes a credential bundle in the environment. The bundle contains typed injection entries; the isolation provider implements the mechanics for each type it supports. Returns success or an error if an unsupported injection type is encountered.
- `cleanup(handle, revocation_bundle)` → cleanup result — removes artifacts described by a revocation bundle from the environment. Deletes files, unsets env vars, and performs any provider-specific cleanup.

### Capabilities

- `supported_injection_types()` → list of types — declares which injection types (`file`, `env`, and any future types) this provider can handle. The broker checks this before routing a bundle, so unsupported types fail fast with a clear error rather than silently dropping entries.

### Injection Type Implementation

Each isolation provider implements the injection types it supports differently, based on the mechanics of its container or VM technology:

| Injection Type | Localhost | Devcontainer | Kubernetes Pod | Remote VM |
|---|---|---|---|---|
| `file` | Write directly to filesystem | Write to bind-mounted shared volume | Create Secret, mount into pod | scp to host |
| `env` | Export in shell profile | Container env config or wrapper script | Pod env spec or ConfigMap | Export in shell profile |

If an isolation provider encounters an injection type it doesn't support, it returns an error. This lets the system add new injection types incrementally without requiring all isolation providers to update simultaneously.

Isolation providers are **credential-agnostic**. They don't know or care whether the bundle contains a GitHub token or a Kubernetes certificate — they just materialize files and env vars according to the bundle's typed entries. The credential provider decides *what* goes in the container; the isolation provider decides *how* it gets there.

### Relationship to Credential Injection

Credential injection is a three-way collaboration described in Credential Injection:

1. The **credential provider** produces the bundle (what needs to exist).
2. The **broker** orchestrates the flow (routing, splitting metadata from secrets).
3. The **isolation provider** materializes the bundle (how it gets into the container).

The broker completes injection before returning the `request_access` MCP response, so by the time the agent sees "kubectl is now configured," the kubeconfig is already in place.

In the single-process deployment, secrets pass through the broker in memory during this handoff. As the broker decomposes at scale, credential workers encrypt bundles before enqueuing them, and isolation workers decrypt after dequeuing — the control plane never touches plaintext bundle contents.

## Communication

There are two distinct communication channels:

**Agent-to-broker** (resolved): The agent communicates with the credential broker via MCP. The broker exposes `request_access`, `revoke`, and `list_active` as MCP tools. See Credential Brokering and MCP Integration.

**CLI-to-broker** (resolved): The operator's CLI communicates with the broker over HTTP API — a separate surface from the agent's MCP interface. This API exists from Adoption Phase 1 and is self-documenting (OpenAPI spec, Swagger UI at `/docs`).

**Operator-facing** (open): The operator needs visibility into the agent's work and the ability to act on approval prompts. This channel must support:

- Real-time output streaming from the agent
- Approval prompts that the operator can act on (Tier 4 human approvals, out-of-budget escalations)
- Credential request notifications
- Task status and progress

The operator-facing channel is unresolved. Options include a local web UI, terminal integration, or Slack/webhook notifications. In solo mode, the operator is already at their terminal, so direct terminal integration may be sufficient. In organizational mode, asynchronous notification channels become more important.

---

## Open Questions

### Isolation Provider Contract Sub-questions

- Should `setup()` be synchronous (block until the environment is ready) or return a handle immediately with a readiness check? Devcontainer startup can take seconds to minutes depending on feature installation.
- How does the isolation provider report partial injection failure (e.g., one file written successfully, another failed due to permissions)?
- Should the provider contract include a `status(handle)` method for the broker to check environment health, or is that out of scope?
