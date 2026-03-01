# Implementation Phases

The implementation is structured as a series of phases, each building on the last. Every phase produces something immediately usable for safer agent operation, and every line of code moves toward the full framework.

---

## Phase 1: Credential Scoping CLI

**Goal**: Replace the current workflow of passing broad personal credentials to agents with a simple way to generate narrower credentials on demand. This is useful immediately and establishes the core credential logic that all later phases build on.

**What gets built**:
- A TypeScript CLI (`wlk`) with two commands: `github` and `kubernetes`.
- GitHub: JWT signing, token exchange, git credential helper configuration, commit identity setup.
- Kubernetes: `tsh kube login` wrapping, kubeconfig generation, TTL management.
- No isolation, no broker, no MCP, no tiers, no manifests. The operator runs the CLI manually before starting an agent session.

**Prerequisites (manual, one-time)**:
- Create and install a GitHub App on target organization(s). Store the private key locally.
- Define scoped Teleport roles for agent use (see Teleport Role Design below).

**GitHub App setup — how it works**:

One App is sufficient for all tasks across an org. The App has three layers of scoping:

1. **App manifest** (set once at creation) — declares the maximum set of permissions the App *could ever* request (e.g., `contents: write`, `actions: read`, `pull_requests: write`). This is the ceiling. Set it to the broadest set of permissions any agent task might need.
2. **Installation** (set once per org) — when the App is installed on an org, the org admin approves the declared permissions and chooses which repos the App can access (all repos, or a specific subset). This is also a one-time step.
3. **Token request** (per `request_access` call) — each call to `POST /app/installations/{installation_id}/access_tokens` can narrow to a subset of repos and a subset of permissions from what the installation allows. Each token is independently scoped and independently expires in 1 hour.

This means from a single App installed once on an org, you can generate many concurrent tokens with different scopes:
- Token A: `contents:write` on `org/repo-a` only
- Token B: `actions:read` on `org/repo-b` and `org/repo-c` only
- Token C: `contents:write` + `pull_requests:write` on `org/repo-a` through `org/repo-d`

The App installation is the [broker's identity](05-broker-identity.md) with GitHub. Each `request_access` call derives a narrower token from it.

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

**Kubernetes usage**:

```bash
npx wlk kubernetes \
  --cluster customer-a-staging \
  --role readonly \
  --ttl 30m

# Output: kubeconfig configured
# Cluster: customer-a-staging
# Role: readonly
# Expires: 2026-02-22T15:00:00Z (30 minutes)
# Export with: export KUBECONFIG=/tmp/agent-kube-xxxx/config
```

**GitHub implementation notes**:
- Uses the GitHub App JWT authentication flow: sign a JWT with the App private key, exchange it for an installation token scoped to specific repos and permissions.
- GitHub App installation tokens expire in 1 hour max — no need for manual revocation in most cases.
- The GitHub REST API endpoint is `POST /app/installations/{installation_id}/access_tokens` with a body specifying `repositories` and `permissions`.
- A custom git credential helper returns the token only for matching HTTPS URLs, ensuring the token only works for in-scope repos. The token exists in the credential helper script on the container's filesystem — git consumes it transparently without the model needing to handle it.
- Git identity is configured as the App's bot account — commits are automatically verified by GitHub.
- TypeScript libraries: `jsonwebtoken` for JWT signing, `octokit` for GitHub API calls.

**Kubernetes implementation notes (via Teleport)**:

Teleport already provides complete credential isolation for Kubernetes. The agent never possesses credentials that work against the real K8s API server. The connection chain is:

```
kubectl → (Teleport cert, mTLS) → Teleport Proxy → (reverse tunnel) → Teleport K8s Service → (impersonation headers) → K8s API Server
```

- `tsh kube login` issues an x509 cert signed by **Teleport's User CA** — not a Kubernetes cert. The cert encodes Kubernetes groups/users in custom ASN.1 extensions (OID `1.3.9999.1.*`). This cert is useless against the K8s API server directly — it only works against the Teleport Proxy.
- The kubeconfig points to the **Teleport Proxy address**, not the K8s API server. The agent never learns the real API server address.
- The Teleport Kubernetes Service connects to the real K8s API server using **its own service account** and translates Teleport identity into **Kubernetes impersonation headers** (`Impersonate-User`, `Impersonate-Group`). Standard K8s RBAC applies from there.
- Certificate TTL is controlled per-role via `max_session_ttl` (default 12h, hard cap 30h). Teleport takes the minimum across all applicable roles.

**Machine ID (tbot) integration** — the programmatic path for Wardlock's [credential provider](03-credential-providers.md):

- `tbot` is a daemon for non-human identities. It authenticates via join tokens (supports AWS IAM, GCP SA, K8s SA — no interactive login) and continuously renews short-lived Teleport certs.
- The `kubernetes/v2` service type outputs a standard `kubeconfig.yaml` with embedded Teleport certs to a configured directory. This is the integration point: the Wardlock Teleport provider manages tbot lifecycle and injects the output kubeconfig as a `file` injection entry.
- On revocation, stop tbot and delete the kubeconfig. The Teleport certs in the kubeconfig are already short-lived and scoped to specific clusters/groups.

Because Teleport handles the credential isolation and proxying, Wardlock does not need to build its own TLS-intercepting proxy for Kubernetes when Teleport is available. Wardlock adds the request/approve/revoke lifecycle on top of Teleport's existing credential isolation.

For non-Teleport Kubernetes setups (direct API server access with bearer tokens or client certs), the [proxy injection type](04-credential-injection.md) described in open questions would be the alternative path.

**Teleport Application Access** — Teleport also provides an HTTPS reverse proxy for web applications and APIs. The architecture is the same pattern: client connects to the Teleport Proxy with Teleport certs, the Proxy forwards through a reverse tunnel to the Teleport Application Service, which rewrites requests (injects headers, JWTs) and forwards to the backend. Teleport has first-class credential injection for AWS (IAM role assumption, SigV4 signing), Azure, and GCP. For arbitrary SaaS APIs (like GitHub), Teleport can proxy the traffic and inject static headers, but lacks the dynamic credential management that Wardlock provides (issuing scoped short-lived tokens, rotation, revocation). The integration model for cloud APIs: Wardlock's Teleport provider leverages Teleport's native cloud support where available, and falls back to Wardlock's own credential providers (e.g., GitHub App tokens) with file injection for SaaS APIs that Teleport doesn't natively handle.

**Teleport as a credential provider** — In Wardlock's taxonomy, Teleport is a **[credential provider](03-credential-providers.md)**, not a new provider type. The Teleport provider's `issue()` call configures tbot to produce certs for the requested cluster or app and returns a `CredentialBundle` with `file` injection entries (kubeconfig, TLS certs). The fact that the kubeconfig points to the Teleport Proxy rather than the real K8s API server is an implementation detail of the credential — not something Wardlock needs to model separately. The proxy behavior is inherent to the credential: a Teleport cert naturally routes through the Teleport Proxy, the same way a GitHub credential helper naturally calls the GitHub API. Wardlock doesn't manage git's HTTPS connections — it just puts the credential helper in place. The provider taxonomy stays clean: credential providers produce bundles, isolation providers materialize them.

- TypeScript: `child_process.execFile` to wrap `tsh`/`tbot`, `js-yaml` for kubeconfig manipulation.

**Teleport role design** — scoped Teleport roles need to exist before the Kubernetes credential scoping works. Examples:

```yaml
# teleport role: kube-agent-readonly
kind: role
metadata:
  name: kube-agent-readonly
spec:
  allow:
    kubernetes_labels:
      '*': '*'
    kubernetes_resources:
      - kind: '*'
        namespace: '*'
        name: '*'
        verbs: ['get', 'list', 'watch']
    kubernetes_groups:
      - 'view'

# teleport role: kube-agent-namespace-admin
kind: role
metadata:
  name: kube-agent-namespace-admin
spec:
  allow:
    kubernetes_labels:
      '*': '*'
    kubernetes_resources:
      - kind: '*'
        namespace: '{{internal.kubernetes_namespace}}'
        name: '*'
        verbs: ['*']
    kubernetes_groups:
      - 'edit'
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
5. Kubernetes credential CLI
6. Integration test: run an actual agent session with both scoped credentials

---

## Phase 2: Minimal MCP Server

**Goal**: Wrap the Phase 1 credential logic as an MCP server, validating the MCP integration pattern and the metadata-only response model. This becomes the seed of the actual broker.

**What gets built**:
- A minimal MCP server exposing `request_access` and `list_active` as MCP tools.
- Two hardcoded "providers" (GitHub and Kubernetes) using the Phase 1 credential logic internally.
- Credential injection as a side effect of the MCP tool call — the MCP response contains metadata only, secrets are written to the filesystem.
- A **localhost [isolation provider](01-isolation.md)** that writes files directly to the operator's filesystem — no containers, no isolation, but it implements the full isolation provider interface and validates the injection/revocation lifecycle.
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
- Does the [MCP integration](02-credential-brokering.md#mcp-integration) work smoothly with Claude Code?
- Does the metadata-only response model hold up in practice — can the agent work effectively without secrets in the MCP response, even though they're accessible on the filesystem?
- What does the agent need to be told (via system prompt / context) to use `request_access` correctly?
- Is the `InjectionResult` metadata sufficient for the agent to understand its capabilities?
- Does the localhost isolation provider skeleton capture the right interface for future providers to implement?

**Order of work**:
1. Add MCP server infrastructure to the Phase 1 project (using the official TypeScript MCP SDK)
2. Implement the localhost isolation provider (file writes, revocation/cleanup)
3. Wrap GitHub credential logic as a `request_access` handler
4. Wrap Kubernetes credential logic as a `request_access` handler
5. Add `list_active` tool for the agent to check current credentials
6. Configure Claude Code to use the MCP server and test end-to-end

---

## Phase 3: Full MVP

**Goal**: Validate the complete framework loop — manifest → isolate → broker credentials → work → revoke — with the architecture described in this document.

**What gets built on top of Phase 2**:
- **Devcontainer isolation**: The framework generates a `devcontainer.json` from the [task manifest](07-task-manifest.md), including required CLI tool features. The `devcontainer` CLI handles container creation and lifecycle. No inherited host credentials.
- **Task manifest**: YAML file defining expected credentials. Operator reviews and approves it before the task starts. Manifest generates startup instructions for the agent.
- **[Tier 1 and 2 approval](06-tiered-approval.md)**: Auto-approve and rubber stamp. Skip adversarial review agents entirely.
- **[Provider plugin interface](03-credential-providers.md#provider-contract)**: Extract the hardcoded GitHub and Kubernetes logic into proper Go provider plugins speaking a defined protocol to the TypeScript broker.
- **mTLS**: Task-scoped client certificates for agent-to-broker authentication.
- **[Approval budgets](07-task-manifest.md#approval-budgets)**: Per-tier limits on out-of-manifest credential requests.
- **File-based [audit logging](09-audit-logging.md)**: Append-only JSON lines log of all credential requests, approvals, and revocations.

**What this validates**:
- Can the full credential broker model support real multi-repo infrastructure tasks end-to-end?
- Does the task manifest capture enough information to be useful without being burdensome to author?
- Is the provider plugin interface general enough to support GitHub and Kubernetes/Teleport without provider-specific leaks into the core?
- Does running in a devcontainer with brokered credentials feel meaningfully safer than the current manual approach?
- Is devcontainer startup time acceptable, or does feature installation overhead warrant caching/snapshotting?
- What's the approval UX friction like in practice — does the tier model reduce interruptions to a tolerable level?

## Questions to Resolve Before Phase 3

These must be answered before Phase 3 implementation can begin. Phases 1 and 2 can proceed without them.

1. **Provider plugin protocol** — provider plugins run on workers, not the control plane (see [Broker Decomposition](11-adoption-and-scaling.md#broker-decomposition-at-scale)). The protocol is between a worker process and the provider plugin it loads. Options: Go function interface compiled into the worker binary (simplest), or a subprocess protocol like JSON over stdin/stdout (if plugin isolation is desired). The [provider contract interface](03-credential-providers.md#provider-contract) is defined, but the loading mechanism needs to be decided.

2. **Devcontainer configuration** — the framework generates a `devcontainer.json` from the task manifest, mapping credential providers to devcontainer features (e.g., GitHub provider → `github-cli` feature, Kubernetes provider → `kubectl-helm-minikube` feature + `tsh`). For Phase 3, the mapping from providers to features can be hardcoded. Open questions: should the operator be able to add additional features beyond what the providers require (e.g., language runtimes, editors)? Should the framework support a base devcontainer.json that gets extended per task?

3. **mTLS certificate management** — the framework generates task-scoped client certificates for agent-to-broker auth. In solo mode, the framework acts as its own CA. What library/tooling handles cert generation? How are certs injected into the devcontainer alongside the MCP configuration?

---

## Open Questions

### Provider Language Migration (TypeScript → Go)

Phase 1-2 build GitHub and Kubernetes credential logic in TypeScript (hardcoded in the broker). Phase 3 specifies extracting these into Go provider plugins speaking a protocol to the broker. This implies rewriting the credential logic in Go.

Options:
- **Accept the rewrite** — Phase 1-2 TypeScript validates the logic and design. Phase 3 rewrites in Go for production. The TypeScript code is a prototype.
- **Keep providers in TypeScript** — drop the Go requirement. Providers are still separate processes speaking a protocol, just in TypeScript. Loses single-binary distribution and direct Teleport client library access.
- **Phase 2.5 extraction** — extract to Go plugins before Phase 3 adds isolation/manifests/tiers, isolating the language migration from the feature additions.

The strongest argument for Go is direct Teleport client library access (avoiding `tsh` CLI wrapping). If `tsh` wrapping proves sufficient through Phase 1-2, Go may not be necessary. Let real usage inform this decision.
