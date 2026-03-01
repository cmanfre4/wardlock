# Implementation Phases

The implementation is structured as a series of phases, each building on the last. Every phase produces something immediately usable for safer agent operation, and every line of code moves toward the full framework.

---

## Phase 1: Credential Scoping CLI

**Goal**: Replace the current workflow of passing broad personal credentials to agents with a simple way to generate narrower credentials on demand. This is useful immediately and establishes the core credential logic that all later phases build on.

**What gets built**:
- A TypeScript CLI (`wlk`) with two commands: `github` and `teleport`.
- GitHub: JWT signing, token exchange, git credential helper configuration, commit identity setup. See GitHub provider spec for implementation details.
- Teleport: `tsh kube login` wrapping, kubeconfig generation, TTL management. See Teleport provider spec for implementation details.
- No isolation, no broker, no MCP, no tiers, no manifests. The operator runs the CLI manually before starting an agent session.

**Prerequisites (manual, one-time)**:
- Create and install a GitHub App on target organization(s). See GitHub prerequisites.
- Define scoped Teleport roles for agent use. See Teleport role design.

**GitHub usage**:

```bash
npx wlk github \
  --app-id 12345 \
  --app-key ~/.config/wardlock/github-app-key.pem \
  --org my-org \
  --repos "eks-cluster-module,customer-a-infra,customer-b-infra" \
  --permissions "contents:write,actions:read"

# Output: credential configured, no secrets displayed
# GitHub credential helper configured for HTTPS access
# Scoped to: my-org/eks-cluster-module, my-org/customer-a-infra, my-org/customer-b-infra
# Permissions: contents:write, actions:read
# Commit identity: wardlock[bot]
# Expires: 2026-02-22T15:30:00Z (1 hour)
```

**Teleport usage**:

```bash
npx wlk teleport \
  --cluster customer-a-staging \
  --role readonly \
  --ttl 30m

# Output: kubeconfig configured
# Cluster: customer-a-staging
# Role: readonly
# Expires: 2026-02-22T15:00:00Z (30 minutes)
# Export with: export KUBECONFIG=/tmp/agent-kube-xxxx/config
```

**What this validates**:
- Does the GitHub App token flow work end-to-end (JWT → token → credential helper → git push)?
- Does the scoped git credential helper correctly restrict access to only in-scope repos?
- Are the Teleport roles granular enough for real tasks, or do the existing cluster RBAC roles need adjustment?
- Does the agent behave gracefully when it hits a permission boundary?

**Order of work**:
1. GitHub App setup (manual, one-time)
2. TypeScript project setup
3. GitHub credential CLI + git credential helper
4. Teleport role design and deployment
5. Teleport credential CLI
6. Integration test: run an actual agent session with both scoped credentials

---

## Phase 2: Minimal MCP Server

**Goal**: Wrap the Phase 1 credential logic as an MCP server, validating the MCP integration pattern and the metadata-only response model. This becomes the seed of the actual broker.

**What gets built**:
- A minimal MCP server exposing `request_access` and `list_active` as MCP tools.
- Two hardcoded providers (GitHub and Teleport) using the Phase 1 credential logic internally.
- Credential injection as a side effect of the MCP tool call — the MCP response contains metadata only, secrets are written to the filesystem.
- A **localhost isolation provider** that writes files directly to the operator's filesystem — no containers, no isolation, but it implements the full isolation provider interface and validates the injection/revocation lifecycle.
- No tiers, no manifests, no overrides, no mTLS, no container isolation. The operator configures Claude Code to use the MCP server and runs the agent locally.

**What changes from Phase 1**:
- Instead of the operator running `npx wlk github ...` manually before a session, the agent calls `request_access` via MCP during the session.
- The credential helper and kubeconfig are configured as side effects of the MCP call.
- The agent has awareness of what credentials it has and when they expire (metadata is in context).

**Example MCP interaction**:
```
Agent calls: request_access(provider: "github", resource: "org/eks-cluster-module", permission: "contents:write", reason: "Update module code")

Agent receives:
{
  "status": "approved",
  "credential_id": "cred-gh-abc123",
  "expires": "2026-02-22T15:30:00Z",
  "injected": {
    "credential_helper": "configured for https://github.com/org/eks-cluster-module",
    "commit_identity": "wardlock[bot]"
  },
  "note": "git configured for HTTPS access to org/eks-cluster-module with contents:write. Commits attributed to wardlock[bot]."
}
```

**What this validates**:
- Does the MCP integration work smoothly with Claude Code?
- Does the metadata-only response model hold up in practice — can the agent work effectively without secrets in the MCP response, even though they're accessible on the filesystem?
- What does the agent need to be told (via system prompt / context) to use `request_access` correctly?
- Is the `InjectionResult` metadata sufficient for the agent to understand its capabilities?
- Does the localhost isolation provider skeleton capture the right interface for future providers to implement?

**Order of work**:
1. Add MCP server infrastructure to the Phase 1 project (using the official TypeScript MCP SDK)
2. Implement the localhost isolation provider (file writes, revocation/cleanup)
3. Wrap GitHub credential logic as a `request_access` handler
4. Wrap Teleport credential logic as a `request_access` handler
5. Add `list_active` tool for the agent to check current credentials
6. Configure Claude Code to use the MCP server and test end-to-end

---

## Phase 3: Full MVP

**Goal**: Validate the complete framework loop — manifest → isolate → broker credentials → work → revoke — with the architecture described in this document.

**What gets built on top of Phase 2**:
- **Devcontainer isolation**: The framework generates a `devcontainer.json` from the task manifest, including required CLI tool features. The `devcontainer` CLI handles container creation and lifecycle. No inherited host credentials.
- **Task manifest**: YAML file defining expected credentials. Operator reviews and approves it before the task starts. Manifest generates startup instructions for the agent.
- **Tier 1 and 2 approval**: Auto-approve and rubber stamp. Skip adversarial review agents entirely.
- **Provider plugin interface**: Extract the hardcoded GitHub and Teleport logic into proper provider plugins speaking a defined protocol to the TypeScript broker.
- **mTLS**: Task-scoped client certificates for agent-to-broker authentication.
- **Approval budgets**: Per-tier limits on out-of-manifest credential requests.
- **File-based audit logging**: Append-only JSON lines log of all credential requests, approvals, and revocations.

**What this validates**:
- Can the full credential broker model support real multi-repo infrastructure tasks end-to-end?
- Does the task manifest capture enough information to be useful without being burdensome to author?
- Is the provider plugin interface general enough to support GitHub and Teleport without provider-specific leaks into the core?
- Does running in a devcontainer with brokered credentials feel meaningfully safer than the current manual approach?
- Is devcontainer startup time acceptable, or does feature installation overhead warrant caching/snapshotting?
- What's the approval UX friction like in practice — does the tier model reduce interruptions to a tolerable level?

## Questions to Resolve Before Phase 3

These must be answered before Phase 3 implementation can begin. Phases 1 and 2 can proceed without them.

1. **Provider plugin protocol** — provider plugins run on workers, not the control plane (see Broker Decomposition). The protocol is between a worker process and the provider plugin it loads. Options: Go function interface compiled into the worker binary (simplest), or a subprocess protocol like JSON over stdin/stdout (if plugin isolation is desired). The provider contract interface is defined, but the loading mechanism needs to be decided.

2. **Devcontainer configuration** — the framework generates a `devcontainer.json` from the task manifest, mapping credential providers to devcontainer features (e.g., GitHub provider → `git`, Teleport provider → `kubectl` + `tsh`). For Phase 3, the mapping from providers to features can be hardcoded. Open questions: should the operator be able to add additional features beyond what the providers require (e.g., language runtimes, editors)? Should the framework support a base devcontainer.json that gets extended per task?

3. **mTLS certificate management** — the framework generates task-scoped client certificates for agent-to-broker auth. In solo mode, the framework acts as its own CA. What library/tooling handles cert generation? How are certs injected into the devcontainer alongside the MCP configuration?

