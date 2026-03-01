# Credential Brokering

The central component of the framework. The credential broker manages the lifecycle of scoped, time-limited credentials across all backend systems.

## Interface

The broker exposes a consistent interface regardless of backend:

**Credential lifecycle (MCP tools for working agents):**
```
request_access(provider, resource, permission, reason, duration) → scoped credential or denial
revoke(credential_id) → early revocation
list_active() → currently active credentials
```

**Audit and analysis (MCP tools for specialized agents, CLI / API for operators):**
```
audit_log(filter) → query audit trail
```

Working agents are configured with only the credential lifecycle tools. The `audit_log` tool is not exposed to working agents — they have no reason to query the audit trail during task execution. However, `audit_log` is available as an MCP tool for specialized agents tuned for audit analysis, compliance review, or manifest improvement (see below). It is also available via CLI and API for operator use.

The `provider`, `resource`, `permission`, and `reason` fields are the same four fields used in [task manifests](07-task-manifest.md), [operator overrides](06-tiered-approval.md), and [audit logs](09-audit-logging.md). This vocabulary is consistent across the entire framework. The `resource` and `permission` values are opaque strings interpreted by the provider — the broker routes the request and manages the lifecycle, but the provider gives them meaning.

**Critical design principle: the credential management flow does not surface secrets into the model's context.** The broker interface returns metadata about issued credentials (status, expiry, what was configured) — not the actual secret material (tokens, certificates, passwords). Secrets exist in the container's filesystem where CLI tools consume them, and the agent *could* read them (e.g., `cat ~/.kube/config`), but the framework's MCP interface never puts them into the conversation. Credential injection is a side effect of the `request_access` call, handled by the broker and [isolation provider](01-isolation.md) deterministically — not by the model. The model needs the *capability* (e.g., "kubectl is now configured for cluster X"), not the *secret* (e.g., the certificate bytes). This ensures that:

- The default flow does not leak secrets through model outputs, conversation logs, or prompt injection attacks. Secrets only enter the model's context if the agent explicitly reads credential files from the filesystem.
- Credential injection is deterministic and correct every time (broker code, not model inference).
- The model's context window is not polluted with secret material it has no reason to interpret.

Example `request_access` response (what the model sees):

```json
{
  "status": "approved",
  "provider": "kubernetes",
  "resource": "customer-a-staging",
  "permission": "readonly",
  "expires": "2026-02-22T15:30:00Z",
  "credential_id": "cred-abc123",
  "injected": {
    "kubeconfig": "~/.kube/config",
    "context": "customer-a-staging"
  },
  "note": "kubectl is now configured for cluster customer-a-staging with readonly access"
}
```

The model knows what tools it can now use, which context to target, and when the credential expires. The underlying certificate or token is in the container's filesystem for CLI tools to use, but is not surfaced through the MCP response.

## Credential Lifecycle

All credentials — both initial (declared in the manifest) and mid-task (discovered during work) — flow through the same `request_access` path. There is no separate pre-provisioning step.

1. **Request** — the agent calls `request_access` via MCP. At task start, the agent makes one call per credential declared in the manifest. Mid-task, the agent makes additional calls as it discovers new needs.
2. **Evaluate** — the framework determines the [approval tier](06-tiered-approval.md). Manifest-declared credentials are pre-approved (the operator approved them when reviewing the manifest) and pass through immediately. Out-of-manifest requests are evaluated against the tier classification and approval budget.
3. **Issue** — the backend [credential provider](03-credential-providers.md) creates a scoped, time-limited credential and returns a credential bundle describing what needs to be injected. In the single-process deployment, the broker calls the provider directly. As the broker [decomposes at scale](11-adoption-and-scaling.md#broker-decomposition-at-scale), a credential worker executes the provider plugin and encrypts the bundle before passing it on.
4. **Inject** — the bundle is handed to the [isolation provider](01-isolation.md), which materializes the files in the target environment (localhost filesystem in early phases, container filesystem later). At scale, an [isolation worker](11-adoption-and-scaling.md#broker-decomposition-at-scale) decrypts the bundle and executes the isolation plugin. Secrets are placed where CLI tools expect them — they are not returned through the MCP response.
5. **Respond** — the MCP tool returns metadata to the agent: status, expiry, credential ID, and what was configured. The agent now has this information in its context and knows what capabilities are available.
6. **Monitor** — the framework tracks credential expiry.
7. **Revoke** — credentials are revoked when the agent's container exits (normal completion, crash, or operator cancellation), the TTL expires, or early revocation is triggered. Container exit is the primary trigger — the framework ties credential lifecycle to container lifecycle, so active credentials for a task are automatically revoked when the container stops. See [Failure Modes](12-open-questions.md#failure-modes) in Open Questions for edge cases (broker crash, orphaned credentials).

## Time Limits

Credentials should be as short-lived as possible.

- If the backend supports short TTLs natively (e.g., Vault dynamic secrets, AWS STS assume-role with 15-minute duration, Teleport short-lived certificates), use them directly.
- If the backend's minimum TTL is longer than desired (e.g., GitHub fine-grained tokens with 1-day minimum), the framework must be able to revoke or rotate credentials early.
- Credential renewal should be possible for long-running tasks, subject to the same approval tier as the original request.

## MCP Integration

The broker exposes `request_access`, `revoke`, and `list_active` as MCP tools. The agent CLI discovers them like any other MCP server. This is the most natural integration for AI agent CLIs because:

- The agent already knows how to discover and call MCP tools — no custom integration needed.
- MCP tool descriptions serve as built-in documentation for the agent on how to request credentials.
- The MCP tool response is the natural place to return credential metadata (status, expiry, what was configured) without surfacing secret material. The broker orchestrates injection before responding and returns only the bundle's metadata to the agent.
- MCP servers can be configured per-environment via the agent's MCP config, making the broker automatically available whether running on localhost or inside a container.

**Important constraint:** The broker's MCP surface must be **tools only**. MCP also supports *resources* (read-only data the model can access). Credentials must not be exposed as MCP resources, as that would surface secret material into the model's context as part of the framework's own interface.

**Transport:** In the single-process deployment, the MCP server runs on the host as part of the broker process. As the broker [decomposes at scale](11-adoption-and-scaling.md#broker-decomposition-at-scale), the MCP gateway becomes a separate stateless component that proxies between agents and the control plane, scaling horizontally. The transport between container and MCP server is a Unix socket bind-mounted into the container (MCP supports stdio and streamable HTTP transports).

**Fallback options** for non-MCP-compatible agents:
- **HTTP API on a Unix socket** — the broker listens on a socket that's bind-mounted into the container. Works with any agent that can make HTTP calls. The same "inject as side effect, return metadata" pattern applies.
- **CLI wrapper** — a `broker` CLI inside the container communicates with the host broker via a socket or pipe. The agent runs `broker request-access --provider github --resource org/repo ...` as a shell command and parses the JSON metadata response.

Both fallback options implement the same broker interface and the same metadata-only response pattern. The MCP approach is preferred because it requires no additional tooling inside the container and integrates naturally with how AI agents discover capabilities.

---

## Open Questions

### MCP Integration Sub-questions

- How is the MCP server configured in the devcontainer? Can the framework inject the MCP server config into the devcontainer's agent CLI configuration automatically?
- For the Unix socket transport, what are the security implications of a bind-mounted socket? Can other processes in the container abuse it?
- Should the MCP server support notifications (MCP has a notification mechanism) to proactively inform the agent about expiring credentials?
