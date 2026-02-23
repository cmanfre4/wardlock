# Wardlock (`wlk`)

A framework for isolating agents and providing scoped, time-limited permissions to external resources so they can work autonomously with bounded blast radius.

Named for the ward — the internal mechanism of a lock that determines which key fits. Wardlock determines what level of access an agent can request, and gates the approval required to grant it.

## Problem Statement

When an AI agent operates across multiple git repositories and external systems (cloud infrastructure, Kubernetes clusters, CI/CD pipelines, secret stores), the blast radius of something going wrong scales dramatically. Unlike single-repo local work — where git history makes recovery straightforward — operations against live systems can be irreversible, affect customers, or cascade across system boundaries in ways that are difficult to roll back.

Today, the typical workaround is manual: the operator configures credentials, scopes kubectl contexts, exports AWS session tokens, and relies on context engineering to constrain the agent's behavior. This is fragile. The agent inherits the operator's full permissions, and the only guardrails are soft — prompts and conventions rather than hard access controls.

The goal of this framework is to formalize and automate that manual setup while making it significantly more scoped, auditable, and safe.

## Scope

This framework is primarily concerned with complex tasks that:

- **Span multiple git repositories** — where changes in one repo may have cascading effects on others (dependency bumps, shared configuration, API contracts), and rollback becomes a coordination problem across repo boundaries.
- **Require access to systems beyond git repos** — cloud infrastructure, Kubernetes clusters, CI/CD pipelines, databases, secret stores, and other systems where operations may not have clean "undo" semantics.

Single-repo local work is largely out of scope. Git history provides sufficient recovery, and existing agent tooling (worktrees, permission modes) handles this adequately.

## Core Architecture

The framework has three pillars: **Isolation**, **Credential Brokering**, and **Approval & Audit**.

The design follows the Terraform provider model: a core framework handles common concerns (isolation lifecycle, approval flows, audit logging, credential injection and revocation), while provider plugins implement a consistent interface for each backend system.

---

## Isolation

The agent runs in an isolated environment (container or VM) that only has access to what the current task requires, rather than inheriting the operator's full host credentials and filesystem.

### Isolation Providers

Isolation is modular, providing consistent semantics across varying container and VM technologies.

- **Localhost** — no actual isolation. The agent runs directly on the operator's machine. This is the starting point: it lets the credential brokering loop work end-to-end without container complexity, and serves as a skeleton/reference implementation for the isolation provider interface.
- **Docker / Devcontainers** — low overhead, widely understood, good CLI/IDE integration. The first real isolation provider, providing filesystem and credential isolation from the host.
- **Kubernetes (EKS)** — more relevant for multi-agent or cloud-native scenarios. Better suited as a target provider in the modular layer rather than the starting point.
- **Remote VMs** — for scenarios where local resources are insufficient or stronger isolation guarantees are needed.

### What Isolation Provides

- A shell/environment with only the CLI tools needed for the task (kubectl, tsh, vault, terraform, gh, aws, etc.)
- No direct access to the host filesystem beyond explicitly mounted project directories
- No inherited credentials from the host — all credentials are injected by the credential broker

### Communication

There are two distinct communication channels:

**Agent-to-broker** (resolved): The agent communicates with the credential broker via MCP. The broker exposes `request_access`, `revoke`, and `list_active` as MCP tools. This is detailed in the Credential Brokering and Agent-to-Broker Communication sections.

**Operator-facing** (open): The operator needs visibility into the agent's work and the ability to act on approval prompts. This channel must support:

- Real-time output streaming from the agent
- Approval prompts that the operator can act on (Tier 4 human approvals, out-of-budget escalations)
- Credential request notifications
- Task status and progress

The operator-facing channel is unresolved. Options include a local web UI, terminal integration, or Slack/webhook notifications. In solo mode, the operator is already at their terminal, so direct terminal integration may be sufficient. In organizational mode, asynchronous notification channels become more important.

---

## Credential Brokering

The central component of the framework. The credential broker manages the lifecycle of scoped, time-limited credentials across all backend systems.

### Interface

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

The `provider`, `resource`, `permission`, and `reason` fields are the same four fields used in task manifests, operator overrides, and audit logs. This vocabulary is consistent across the entire framework. The `resource` and `permission` values are opaque strings interpreted by the provider — the broker routes the request and manages the lifecycle, but the provider gives them meaning.

**Critical design principle: the credential management flow does not surface secrets into the model's context.** The broker interface returns metadata about issued credentials (status, expiry, what was configured) — not the actual secret material (tokens, certificates, passwords). Secrets exist in the container's filesystem where CLI tools consume them, and the agent *could* read them (e.g., `cat ~/.kube/config`), but the framework's MCP interface never puts them into the conversation. Credential injection is a side effect of the `request_access` call, handled by the broker and isolation provider deterministically — not by the model. The model needs the *capability* (e.g., "kubectl is now configured for cluster X"), not the *secret* (e.g., the certificate bytes). This ensures that:

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

### Credential Lifecycle

All credentials — both initial (declared in the manifest) and mid-task (discovered during work) — flow through the same `request_access` path. There is no separate pre-provisioning step.

1. **Request** — the agent calls `request_access` via MCP. At task start, the agent makes one call per credential declared in the manifest. Mid-task, the agent makes additional calls as it discovers new needs.
2. **Evaluate** — the framework determines the approval tier. Manifest-declared credentials are pre-approved (the operator approved them when reviewing the manifest) and pass through immediately. Out-of-manifest requests are evaluated against the tier classification and approval budget.
3. **Issue** — the backend credential provider creates a scoped, time-limited credential and returns a credential bundle describing what needs to be injected.
4. **Inject** — the broker hands the bundle to the isolation provider, which materializes the files in the target environment (localhost filesystem in early phases, container filesystem later). Secrets are placed where CLI tools expect them — they are not returned through the MCP response.
5. **Respond** — the MCP tool returns metadata to the agent: status, expiry, credential ID, and what was configured. The agent now has this information in its context and knows what capabilities are available.
6. **Monitor** — the framework tracks credential expiry.
7. **Revoke** — credentials are revoked when the agent's container exits (normal completion, crash, or operator cancellation), the TTL expires, or early revocation is triggered. Container exit is the primary trigger — the framework ties credential lifecycle to container lifecycle, so active credentials for a task are automatically revoked when the container stops. See Failure Modes in Open Questions for edge cases (broker crash, orphaned credentials).

### Time Limits

Credentials should be as short-lived as possible.

- If the backend supports short TTLs natively (e.g., Vault dynamic secrets, AWS STS assume-role with 15-minute duration, Teleport short-lived certificates), use them directly.
- If the backend's minimum TTL is longer than desired (e.g., GitHub fine-grained tokens with 1-day minimum), the framework must be able to revoke or rotate credentials early.
- Credential renewal should be possible for long-running tasks, subject to the same approval tier as the original request.

### Providers

Each backend system is implemented as a provider plugin with a consistent interface. Providers handle the specifics of how credentials are requested, scoped, issued, and revoked for their respective systems.

#### GitHub

GitHub is the hardest credential problem due to the breadth of access personal tokens provide. The GitHub provider replaces all personal credentials (admin tokens, SSH keys, signing keys) with a single GitHub App identity.

**Approach**: A GitHub App per organization, managed by the operator in solo mode or centrally by the platform team in organizational mode. The App provides:

- **Repo access** — installation tokens scoped to specific repositories with specific permissions (e.g., `contents:write`, `actions:read`), expiring in 1 hour.
- **Git authentication** — the provider configures a git credential helper in the container that serves the scoped token over HTTPS. The agent uses `https://` git URLs, not `ssh://`. The credential helper only responds to URLs matching in-scope repos — out-of-scope repos fail with a 401.
- **Commit signing** — commits attributed to the App's bot identity (`wardlock[bot]`) are automatically verified by GitHub. No GPG or SSH signing keys needed. This satisfies branch protection signing requirements.
- **Discrete service identity** — the App is a bot account that survives employee departures, is auditable separately from any individual, and makes agent-authored commits clearly identifiable in git history.

A single `request_access` call for a GitHub credential configures all of the above as side effects. The provider's injection is richer than a single credential file — it sets up the credential helper, git user identity, and committer email. The MCP response metadata tells the agent what was configured:

```json
{
  "status": "approved",
  "provider": "github",
  "resource": "org/eks-cluster-module",
  "permission": "contents:write",
  "expires": "2026-02-22T15:30:00Z",
  "credential_id": "cred-gh-abc123",
  "injected": {
    "credential_helper": "configured for https://github.com/org/eks-cluster-module",
    "commit_identity": "wardlock[bot]"
  },
  "note": "git configured for HTTPS access to org/eks-cluster-module with contents:write. Commits attributed to wardlock[bot] and automatically verified."
}
```

#### AWS

- AWS Identity Center (SSO) already supports short-lived credentials.
- The framework requests credentials scoped to specific accounts with specific permission sets.
- Duration can be as short as 15 minutes via STS assume-role.
- Different tasks can use different permission sets (readonly vs. specific write permissions) rather than the operator's full admin access.

#### Kubernetes (via Teleport)

- Teleport already issues short-lived certificates scoped to specific clusters and roles.
- The framework requests certificates for specific clusters with specific Kubernetes roles — not `system:masters` but task-appropriate roles (e.g., a role limited to specific namespaces, or a read-only cluster role).
- This is a direct replacement for the current manual `tsh` + kubectl context workflow.

#### Vault (via Teleport)

- Vault accessed via Teleport, with short-lived tokens scoped to specific secret paths and capabilities.
- Dynamic secrets (database credentials, cloud credentials) generated by Vault can be scoped and time-limited natively.

#### Extensibility

Providers follow the Terraform provider pattern:

- Independently versioned and distributed
- Consistent interface (request, revoke, audit)
- Schema that describes available resource types, permission levels, and scoping options
- Handle authentication and credential management internally

Third parties can implement providers for their own backends (GCP, GitLab, Azure, etc.) against the same interface.

#### Provider Contract

Each provider implements a consistent interface that the broker calls to manage credentials:

- `schema()` — returns the provider's supported resource types, permission levels, and additional match dimensions for overrides.
- `classify(resource, permission)` → tier — returns the provider's default tier classification for a given request.
- `issue(resource, permission, duration)` → credential bundle — creates a scoped, time-limited credential and returns a bundle describing what needs to be injected into the container and what metadata to show the agent.
- `revoke(credential_id)` → revocation bundle — returns the artifacts to remove and any backend-side cleanup (e.g., token invalidation).
- `validate(resource, permission)` → bool — checks whether the requested resource and permission are valid and the provider can fulfill them.

Credential providers are **isolation-agnostic**. They produce declarative bundles describing what needs to exist in the container — they do not write files or configure tools directly. The broker hands the bundle to the isolation provider, which knows how to materialize it in the container (see Credential Injection below).

##### Credential Bundle

The `issue()` method returns a credential bundle: a declarative description of what needs to exist in the agent's container, plus metadata for the agent's context. The bundle contains secret material (file contents, tokens). The broker passes the bundle to the isolation provider for injection and returns only the metadata portion to the agent via the MCP response. The secrets end up in the container's filesystem where CLI tools consume them — the agent could read these files directly, but the credential management flow does not put them into the conversation.

A bundle consists of typed **injection entries**, each declaring an injection type that the isolation provider must implement. This keeps credential providers isolation-agnostic — they declare what they need, and the isolation provider decides how to materialize each type for its container technology.

```typescript
// Each entry declares a type and type-specific parameters
type InjectionEntry =
  | { type: "file"; path: string; content: string; mode?: string }
  | { type: "env"; name: string; value: string };

interface CredentialBundle {
  // Unique identifier for this credential (used for revoke, monitoring)
  credential_id: string;

  // When the credential expires
  expires: string;

  // Typed injection entries — each describes something to materialize
  // in the container. Isolation providers implement the mechanics for
  // each type.
  injections: InjectionEntry[];

  // Metadata for the agent's context — provider-defined, describes what
  // was configured in human-readable terms. This is the ONLY part of the
  // bundle that reaches the model via the MCP response.
  metadata: {
    injected: Record<string, string>;
    note: string;
  };
}
```

**Injection types:**

| Type | Description | Phase |
|---|---|---|
| `file` | Write a file at `path` (relative to container home) with `content`. Optional `mode` (e.g., `"0755"` for executables). | Phase 1 |
| `env` | Set environment variable `name` to `value`. | Phase 3 |

New injection types can be added as needed (e.g., `socket`, `mount`). If an isolation provider encounters a type it doesn't support, it returns an error — the broker reports this as an injection failure. This makes the system extensible without requiring all isolation providers to support every type from day one.

The credential provider produces the bundle. The broker splits it: `injections` go to the isolation provider for materialization, `metadata` goes to the agent via the MCP response. The credential management flow does not surface secret material into the model's context — though the agent has filesystem access to the injected files.

Example bundles by provider:

**GitHub:**
```json
{
  "credential_id": "cred-gh-abc123",
  "expires": "2026-02-22T15:30:00Z",
  "injections": [
    {
      "type": "file",
      "path": ".config/git/wlk-credential-helper",
      "content": "#!/bin/sh\n# credential helper script content...",
      "mode": "0755"
    },
    {
      "type": "file",
      "path": ".gitconfig.d/wardlock.inc",
      "content": "[credential]\n\thelper = !~/.config/git/wlk-credential-helper\n[user]\n\tname = wardlock[bot]\n\temail = 12345+wardlock[bot]@users.noreply.github.com"
    }
  ],
  "metadata": {
    "injected": {
      "credential_helper": "configured for https://github.com/org/eks-cluster-module",
      "commit_identity": "wardlock[bot]",
      "committer_email": "12345+wardlock[bot]@users.noreply.github.com"
    },
    "note": "git configured for HTTPS access to org/eks-cluster-module with contents:write. Commits attributed to wardlock[bot] and automatically verified."
  }
}
```

**Kubernetes:**
```json
{
  "credential_id": "cred-k8s-def456",
  "expires": "2026-02-22T15:00:00Z",
  "injections": [
    {
      "type": "file",
      "path": ".kube/config",
      "content": "# kubeconfig YAML content..."
    }
  ],
  "metadata": {
    "injected": {
      "kubeconfig": "~/.kube/config",
      "context": "customer-a-staging"
    },
    "note": "kubectl configured for cluster customer-a-staging with readonly access."
  }
}
```

The `metadata.injected` field is intentionally a flat key-value map rather than a structured type — each provider defines what keys are meaningful for its domain. The broker doesn't interpret these; it passes them through to the agent.

##### Revocation Bundle

The `revoke()` method returns a revocation bundle describing what to clean up, using the same injection type system:

```typescript
interface RevocationBundle {
  // Typed entries describing what to undo in the container
  revocations: RevocationEntry[];

  // Whether the provider also invalidated the credential backend-side
  backend_revoked: boolean;
}

type RevocationEntry =
  | { type: "file"; path: string }         // delete this file
  | { type: "env"; name: string };         // unset this env var
```

The broker passes the revocation bundle to the isolation provider, which removes the files and unsets env vars. The provider handles backend-side revocation (e.g., invalidating a GitHub token via API) as part of the `revoke()` call itself.

Partial cleanup is acceptable if some artifacts can't be undone (e.g., a token that can only be revoked by TTL expiry). The `backend_revoked` field indicates whether the provider was able to invalidate the credential at the source.

### Credential Injection

Credential injection is a **side effect of the broker's `request_access` call**, not a separate step the model performs. The MCP response contains only metadata — secrets are materialized in the container's filesystem for CLI tools to consume.

Injection is a collaboration between three components:

1. **Credential provider** — produces a credential bundle declaring what needs to exist in the container (files to write, env vars to set) and what metadata to show the agent. The provider is isolation-agnostic — it has no knowledge of how containers work.
2. **Broker** — orchestrates the flow. Calls the credential provider to get the bundle, hands the entries to the isolation provider for materialization, and returns only the metadata portion to the agent via the MCP response.
3. **Isolation provider** — materializes the bundle in the container. It knows how to write files and set env vars for the specific container technology being used.

This separation keeps credential providers and isolation providers fully decoupled. A GitHub credential provider doesn't need to know whether it's running on localhost or in a remote Kubernetes pod — it produces the same bundle either way. Each isolation provider implements the injection types it supports:

| Injection Type | Localhost | Devcontainer (future) | Kubernetes pod (future) | Remote VM (future) |
|---|---|---|---|---|
| `file` | Write directly to filesystem | Write to bind-mounted shared volume | Create Secret, mount into pod | scp to host |
| `env` | Export in shell profile | Container env config or wrapper script | Pod env spec or ConfigMap | Export in shell profile |

If an isolation provider encounters an injection type it doesn't support, it returns an error. This lets the system add new injection types incrementally without requiring all isolation providers to update simultaneously.

Secrets pass through the broker in memory during this handoff, but this is trusted infrastructure code. The broker treats bundle contents as opaque bytes and does not log or inspect them. The secrets ultimately live on the filesystem where the agent runs — accessible to the agent if it reads the files, but not surfaced through the MCP interface.

The broker completes injection before returning the `request_access` MCP response, so by the time the model sees the "kubectl is now configured" metadata, the kubeconfig is already in place.

### MCP Integration

The broker exposes `request_access`, `revoke`, and `list_active` as MCP tools. The agent CLI discovers them like any other MCP server. This is the most natural integration for AI agent CLIs because:

- The agent already knows how to discover and call MCP tools — no custom integration needed.
- MCP tool descriptions serve as built-in documentation for the agent on how to request credentials.
- The MCP tool response is the natural place to return credential metadata (status, expiry, what was configured) without surfacing secret material. The broker orchestrates injection before responding and returns only the bundle's metadata to the agent.
- MCP servers can be configured per-environment via the agent's MCP config, making the broker automatically available whether running on localhost or inside a container.

**Important constraint:** The broker's MCP surface must be **tools only**. MCP also supports *resources* (read-only data the model can access). Credentials must not be exposed as MCP resources, as that would surface secret material into the model's context as part of the framework's own interface.

**Transport:** The MCP server runs on the host (or as a sidecar). The transport between container and host is a Unix socket bind-mounted into the container (MCP supports stdio and streamable HTTP transports).

**Fallback options** for non-MCP-compatible agents:
- **HTTP API on a Unix socket** — the broker listens on a socket that's bind-mounted into the container. Works with any agent that can make HTTP calls. The same "inject as side effect, return metadata" pattern applies.
- **CLI wrapper** — a `broker` CLI inside the container communicates with the host broker via a socket or pipe. The agent runs `broker request-access --provider github --resource org/repo ...` as a shell command and parses the JSON metadata response.

Both fallback options implement the same broker interface and the same metadata-only response pattern. The MCP approach is preferred because it requires no additional tooling inside the container and integrates naturally with how AI agents discover capabilities.

### Bootstrap Credentials

The broker itself needs credentials to create scoped credentials on behalf of the agent. The bootstrap credential model supports two modes that represent a natural progression from individual adoption to organizational rollout.

#### Solo Practitioner Mode

The operator already has broad access to the systems the agent needs to reach. The broker's job is to **derive constrained credentials from the operator's existing sessions** rather than requiring dedicated service identities.

- The GitHub provider uses a GitHub App that the operator installs on their own orgs. The App's private key lives on the operator's machine. The operator is the App's sole administrator.
- The Kubernetes/Teleport provider uses the operator's existing `tsh` session (from their normal Okta SSO login) to request scoped cluster certificates for the agent. The operator already has access to the clusters — the framework just requests narrower roles than the operator would use directly.
- The Vault provider uses the operator's existing Teleport-brokered Vault session to generate scoped tokens.

This mode prioritizes low friction. The operator shouldn't need to set up new infrastructure to get started — they should be able to point the framework at their existing sessions and be running within minutes. The bootstrap credentials are inherently tied to the operator's identity and permissions, which is acceptable for solo use.

#### Organizational Mode

When the framework is adopted by a team or organization, bootstrap credentials move from personal sessions to dedicated service identities managed by the platform/infrastructure team.

- The GitHub provider uses a centrally managed GitHub App installed across the organization's repos, with the private key stored in a secret manager.
- The Kubernetes/Teleport provider uses a Teleport machine identity (bot) with roles specifically designed for credential delegation — capable of requesting scoped certificates for agents, but not capable of direct cluster access itself.
- The Vault provider uses a dedicated Vault AppRole or machine identity with policies that allow generating scoped tokens but not direct secret access.
- Operator override policies are defined centrally and apply to all operators in the organization.

#### Transition Path

The transition from solo to organizational mode should not require changes to task manifests, provider configurations, or the tier system. Two things change: the bootstrap credential source and the broker's deployment model.

**Bootstrap credentials** move from personal sessions to organizational service identities (described above).

**The broker / MCP server** moves from the operator's laptop to a shared service:

1. **Solo mode** — the broker runs as a local process on the operator's machine. The MCP server is exposed to the agent's devcontainer via a bind-mounted Unix socket or local network. The operator's machine is the trust boundary — bootstrap credentials live there, approval prompts appear there.
2. **Organizational mode** — the broker runs as a service, potentially deployed into a Kubernetes cluster. The MCP server is accessible to agent containers over the network (authenticated and encrypted). Bootstrap credentials come from the cluster's secret management (Vault, Kubernetes secrets). Approval prompts route to operators via a web UI, Slack integration, or similar. Audit logs go to a centralized logging system rather than local files. Classification policies and operator overrides are managed centrally and apply to all operators.

The transition path:

1. Solo operator starts using the framework — broker runs locally, MCP server on a Unix socket, bootstrap credentials from local sessions.
2. They find it valuable, want to share it with the team.
3. The broker is deployed as a service in a Kubernetes cluster. The MCP server endpoint moves from a local socket to a network address — but the MCP interface is identical. Agent containers are configured to point at the service endpoint instead of a local socket.
4. Bootstrap credentials are migrated from local sessions to organizational service identities.
5. Centralized operator override policies are added to enforce org-wide standards.
6. Everything else stays the same — same providers, same manifest format, same tier system, same `request_access` MCP calls from the agent's perspective.

The agent doesn't know or care whether the MCP server is a local process on the operator's laptop or a service in a Kubernetes cluster. It makes the same `request_access` calls and gets the same metadata responses. This is analogous to how Terraform works: you start with personal AWS credentials and a local state file, then eventually move to a shared state backend and assume-role chains — but the `.tf` files don't change.

#### Design Implications

**Provider interface:** The provider interface must not assume either mode. A provider receives a bootstrap credential configuration (which could be "use this local session" or "use this service identity endpoint") and uses it to issue scoped credentials. The provider doesn't know or care whether the bootstrap credential came from a personal `tsh` login or a Teleport bot.

**Agent-to-broker authentication:** The framework uses mTLS to authenticate agent containers to the broker. When launching a devcontainer, the framework generates a short-lived client certificate scoped to the task (task ID encoded in the subject/SAN) and injects it into the container alongside the MCP server configuration. The broker validates this certificate on every request, binding each `request_access` call to a specific approved manifest. This is consistent across both modes — in solo mode the framework acts as its own CA locally, in organizational mode the CA could be Teleport or another internal PKI. The client certificate is not a credential the model sees or manages — it is part of the container's infrastructure, like the MCP socket itself.

The provider configuration supports pluggable bootstrap credential sources:

```yaml
providers:
  kubernetes:
    bootstrap:
      # Solo mode: use operator's existing tsh session
      source: local_tsh_session

  github:
    bootstrap:
      # Solo mode: App key on local filesystem
      source: file
      path: ~/.config/wardlock/github-app-key.pem
      app_id: 12345
```

```yaml
providers:
  kubernetes:
    bootstrap:
      # Org mode: dedicated Teleport bot identity
      source: teleport_bot
      token_path: /var/lib/teleport/bot/token

  github:
    bootstrap:
      # Org mode: App key from Vault
      source: vault
      vault_path: secret/data/github-app/private-key
      app_id: 12345
```

---

## Tiered Approval

Not all credential requests carry the same risk. The framework uses a tiered approval model that concentrates human attention where it matters most.

### Tier 1: Auto-approve

**Read/list access to non-sensitive resources.**

- Auto-approved immediately
- Logged for audit
- Examples: reading terraform state, listing repos in an org, reading Kubernetes pod status, viewing CI workflow logs

### Tier 2: Rubber Stamp

**Low blast radius writes, or reads on sensitive resources.**

- Auto-approved but with additional safeguards
- Agent must provide explicit reasoning for the request
- Logged with enhanced detail
- May trigger a brief "pause and reflect" step where the agent re-evaluates whether the access is truly necessary
- Examples: writing to a non-production branch, reading secrets from a dev Vault path, modifying a staging-only resource

### Tier 3: Adversarial Review

**High blast radius operations.**

- Reviewed by one or more independent adversarial agents before execution
- Adversarial agents receive the task description and the permission request, but NOT the full working context of the requesting agent (context isolation prevents shared poisoning)
- If adversarial agents unanimously approve, the request proceeds
- If any adversarial agent denies or agents disagree, the request escalates to Tier 4
- Disagreement itself is treated as a signal worth escalating — not resolved by majority vote
- Logged with full review chain
- Examples: writing to a production Kubernetes cluster, modifying IAM policies, changing Vault access policies, terraform apply against production

### Tier 4: Human Approval

**Highest blast radius, or escalations from Tier 3.**

- Requires explicit human approval
- Presented with full context: what's being requested, why, what the adversarial agents said (if applicable)
- Logged with approval decision and approver
- Examples: operations that could cause customer impact, broad permission requests, anything the adversarial agents flagged as concerning

### Tier Classification

Tier classification determines which approval tier a given credential request falls into. This is a layered system: providers supply sensible defaults based on their domain knowledge, and operators can override or refine those defaults based on their own risk tolerance, organizational policies, and environment-specific context.

#### How Classification Works

When a credential request comes in, the framework evaluates it against classification rules in priority order:

1. **Operator overrides** (highest priority) — explicit rules defined by the operator that target specific resources, accounts, clusters, or paths. These override provider defaults based on the operator's knowledge of their own environment.
2. **Provider defaults** (lowest priority) — the baseline classification the provider ships with, based on resource type and permission level.

The first matching rule wins. This means a provider can ship with reasonable out-of-the-box behavior, but the operator always has final say over how risk is categorized in their environment.

#### Provider Default Classifications

Each provider declares a default tier mapping as part of its schema. Because the framework operates at the **credential level** (issuing scoped credentials, not intercepting individual API calls), provider defaults classify based on the credential's scope and capability — not individual operations the agent might perform with that credential.

A Kubernetes provider might ship with:

| Credential Scope | Default Tier |
|---|---|
| cluster-wide read-only role | Tier 1 |
| namespace-scoped read-write role | Tier 2 |
| cluster-wide read-write role | Tier 3 |
| cluster-admin / system:masters | Tier 4 |

A GitHub provider might ship with:

| Credential Scope | Default Tier |
|---|---|
| single repo, read-only permissions (contents:read, actions:read) | Tier 1 |
| single repo, write permissions (contents:write, pull_requests:write) | Tier 2 |
| multiple repos or wildcard, write permissions | Tier 3 |
| organization-level permissions (members:write, admin:org) | Tier 4 |

An AWS provider might ship with:

| Credential Scope | Default Tier |
|---|---|
| single account, read-only permission set | Tier 1 |
| single account, scoped write permission set (e.g., S3-only, Lambda-only) | Tier 2 |
| single account, broad write permission set (e.g., PowerUserAccess) | Tier 3 |
| single account, AdministratorAccess or IAM-write permission set | Tier 4 |

Note that providers classify based on the breadth and capability of the credential being issued — they have no concept of environments, account purposes, or organizational structure. The distinction between a sandbox account and a production account is operator context, expressed through operator overrides (see below).

These defaults encode the provider author's domain expertise about what's risky in their system. They should be conservative — it's easier for an operator to downgrade a tier than to recover from a miscategorized high-risk operation.

#### Operator Overrides

Operators define overrides in a classification policy file. Overrides can promote or demote the tier for specific resources based on the operator's knowledge of their own environment.

```yaml
classification_overrides:

  # Our dev AWS account is a sandbox — relax write restrictions
  - provider: aws
    match:
      resource: "123456789012"  # dev-sandbox
    permission: "*"
    override_tier: 2
    reason: "Dev sandbox account, no customer data, freely replaceable"

  # Customer production clusters are always Tier 4, even for reads on secrets
  - provider: kubernetes
    match:
      resource: "customer-*-production"
      resource_type: "secrets"
    permission: read
    override_tier: 4
    reason: "Production secrets may contain customer credentials"

  # The shared-infra repo's main branch is critical — always require human approval
  - provider: github
    match:
      resource: "org/shared-infra"
      ref: "main"
    permission: "contents:write"
    override_tier: 4
    reason: "Shared infrastructure repo, changes affect all customers"

  # Staging clusters can be treated as lower risk for pod operations
  - provider: kubernetes
    match:
      resource: "*-staging"
      resource_type: "pods,deployments,services"
    permission: write
    override_tier: 2
    reason: "Staging clusters are non-customer-facing and recoverable"

  # Platform-configuration repos control CI permissions — always adversarial review
  - provider: github
    match:
      resource: "*/platform-configuration"
    permission: "*:write"
    override_tier: 3
    reason: "Controls GitHub Actions permissions via Vault, broad blast radius"
```

Override match rules use the same `resource` and `permission` vocabulary as the broker interface and task manifest. The `resource_type` field is an optional secondary match dimension available when a provider exposes sub-resource classification (e.g., Kubernetes resource kinds within a cluster, Vault secret engine paths within a mount). The `ref` field in the GitHub example is also a provider-specific match refinement — providers declare which additional match dimensions they support in their schema, and the override system passes them through.

#### Classification Dimensions

Tier classification can take into account multiple dimensions, not just resource type and permission level:

- **Environment** — production vs. staging vs. dev. The same operation (e.g., kubectl delete pod) might be Tier 2 in staging and Tier 4 in production.
- **Resource sensitivity** — whether the resource contains customer data, secrets, or access control configurations.
- **Blast radius** — how many systems, customers, or services could be affected. Deleting a single pod is different from deleting a namespace.
- **Reversibility** — whether the operation can be cleanly undone. `terraform plan` is always safe. `terraform apply` on a resource that can be recreated is less risky than `terraform apply` on a stateful resource like a database.
- **Scope breadth** — a request for write access to one specific repo is less risky than write access to `org/*`.

#### Credential-Level vs. Operation-Level Gating

The tier classification system described above operates at the **credential level** — it determines the approval required before a credential is issued. Once issued, the agent can perform any operation that credential permits.

This is an important boundary. The framework does not intercept individual commands (`kubectl delete`, `terraform apply`, `vault write`). That would require deep integration with every CLI tool and introduce significant complexity.

However, there are cases where the risk of a credential can't be fully assessed until the agent reveals how it intends to use it. For example, a `ReadOnlyAccess` AWS credential is low risk regardless of usage, but a write credential to a Kubernetes cluster could be used for a benign config update or a destructive namespace deletion.

Two mechanisms address this gap:

1. **Prefer narrower credentials** — rather than issuing a broad credential and hoping the agent uses it responsibly, encourage the agent to request the narrowest credential possible. An agent that needs to update a deployment in a specific namespace should request a namespace-scoped role, not cluster-admin. The tier system naturally incentivizes this: narrower credentials get lower tiers and faster approval.

2. **Operation-level gating (future extension)** — a separate, optional layer that could intercept specific high-risk commands before execution. This is architecturally distinct from credential brokering and significantly more complex to implement. It is noted here as a potential extension, not a core requirement. If implemented, it would function as a command allowlist/denylist within the isolated environment, independent of the credential tier system.

---

## Task Manifest and Approval Budgets

### Task Manifest

A task manifest declares the expected resource requirements for a task upfront. The operator reviews and approves the manifest before the task begins — this review acts as the approval for all declared credentials, regardless of what tier they would normally fall into. The agent only hits an approval gate when it requests access to something not declared in the manifest.

```yaml
task:
  name: "Update EKS module to v1.5.0 and roll out to staging"
  description: "Update the eks-cluster child module, tag a new release, and update customer cluster repos to use the new version."

credentials:
  - provider: github
    resource: "org/eks-cluster-module"
    permission: contents:write
    reason: "Update module code and create release tag"

  - provider: github
    resource: "org/customer-*-infra"
    permission: contents:write
    reason: "Update module version reference in customer repos"

  - provider: github
    resource: "org/customer-*-infra"
    permission: actions:read
    reason: "Monitor terraform plan workflows"

  - provider: kubernetes
    resource: "customer-a-staging"
    permission: readonly
    reason: "Verify rollout health after terraform apply"
```

Every credential entry uses the same four required fields: `provider`, `resource`, `permission`, and `reason`. An optional `duration` field can override the default TTL. The values of `resource` and `permission` are provider-interpreted — a GitHub provider understands that `resource` is a repository identifier and `permission` maps to GitHub App permission scopes, while a Kubernetes provider understands that `resource` is a cluster identifier and `permission` maps to a Teleport/Kubernetes role.

This turns the approval UX from "approve every action" into "approve the plan, then only intervene on deviations" — which maps to how you'd supervise a human doing the same work.

#### Manifest as Startup Instructions

The manifest serves a dual purpose: it is the operator's approval document **and** the agent's startup instructions. When the framework launches the agent, it generates initial context (system prompt or injected instructions) telling the agent to call `request_access` for each credential in the manifest before beginning work:

```
You have been approved for the following credentials for this task. Before
beginning work, request each one using the request_access MCP tool:

1. provider: github, resource: org/eks-cluster-module, permission: contents:write
   reason: "Update module code and create release tag"
2. provider: github, resource: org/customer-*-infra, permission: contents:write
   reason: "Update module version reference in customer repos"
3. provider: github, resource: org/customer-*-infra, permission: actions:read
   reason: "Monitor terraform plan workflows"
4. provider: kubernetes, resource: customer-a-staging, permission: readonly
   reason: "Verify rollout health after terraform apply"

After requesting credentials, you will receive metadata about what was
configured (file paths, kubectl contexts, expiry times). Use this information
to guide your work. If you need access to additional resources not listed
above, you may request them — they will be subject to approval.
```

This approach has several advantages:

- **One codepath** — initial and mid-task credentials both flow through the same `request_access` MCP call. No separate pre-provisioning logic in the broker.
- **Agent has full awareness** — the credential metadata responses are in the agent's context from the start. It knows its access scope, expiry times, and which tools are configured where.
- **Self-documenting sessions** — the beginning of every agent session shows exactly what credentials were requested and granted. This is visible in conversation logs and useful for debugging.
- **Broker stays stateless between runs** — it doesn't need to pre-stage anything before the agent starts. The manifest is the source of truth, and the agent drives the credential acquisition.

### Approval Budgets

For larger tasks or projects, the manifest can include an approval budget that limits how many out-of-manifest credential requests the agent can make before all subsequent requests escalate to human approval, regardless of their tier.

Budgets are specified per tier:

```yaml
budget:
  tier_1: unlimited    # auto-approved reads are low risk, no cap needed
  tier_2: 5            # allow up to 5 additional rubber-stamped requests
  tier_3: 1            # allow 1 additional adversarial-reviewed request
```

- A tight budget means the agent needs a clear plan upfront and the manifest should cover most of what's needed.
- A loose budget gives more flexibility for discovery during the task.
- Budget exhaustion doesn't block the agent — it escalates all subsequent out-of-manifest requests to Tier 4 (human approval).
- Tier 4 requests are never budgeted — they always require human approval.

This helps calibrate the safety/autonomy tradeoff per task rather than applying a single global policy.

---

## Adversarial Review Agents

Adversarial review agents serve as an automated security review layer for high-blast-radius operations. They are conceptually similar to separation of duties in traditional security: the agent performing the work should not be the same one approving elevated access.

### Design Principles

- **Context isolation** — adversarial agents receive the task description and the specific permission request, but NOT the full working context of the requesting agent. This prevents a shared context poisoning attack from compromising both the worker and the reviewer.
- **Independence** — multiple adversarial agents evaluate the request independently, without seeing each other's assessments.
- **Conservatism** — adversarial agents are tuned to be skeptical. Their job is to ask "does this request make sense given the stated task?" and flag anything that seems out of scope.
- **Statistical safety** — the requesting agent may have been led astray by poisoned context or accumulated errors. Because adversarial agents evaluate the request with isolated context and independent reasoning, it is unlikely that they would be fooled by the same failure mode. If any adversarial agent flags a concern, the request escalates to a human rather than proceeding unchecked.

### What Adversarial Agents Evaluate

- Does the requested resource and permission level align with the stated task?
- Is the scope of the request as narrow as it could be?
- Is there a less risky way to accomplish what the agent is trying to do?
- Does the request represent an escalation beyond what the task manifest anticipated?

### Cost Considerations

Each adversarial review is a few API calls. For high-blast-radius operations (production deployments, IAM changes, infrastructure modifications), this cost is trivial relative to the risk being mitigated. The tiered model ensures adversarial review only fires on operations that warrant it.

---

## Audit Logging

All credential requests, approvals, denials, and revocations are logged.

### Log Contents

- Timestamp
- Requesting agent / task ID
- Resource and permission level requested
- Reason provided
- Approval tier and outcome (auto-approved, rubber-stamped, adversarial-reviewed, human-approved, denied)
- Adversarial agent assessments (if applicable)
- Credential ID, TTL, and revocation time

### Alerting and Anomaly Detection

An audit log is only valuable if something is reviewing it. The framework should support:

- Alerting on denied requests or adversarial agent disagreements
- Anomaly detection on credential request patterns (e.g., an agent requesting significantly more access than its task manifest declared)
- Post-task review summaries showing what access was requested vs. what was declared in the manifest

---

## Example Workflow

A typical multi-repo infrastructure task:

1. **Operator defines task manifest** — declares the repos, clusters, and other systems the task will touch, with expected permission levels.
2. **Operator reviews and approves the manifest** — this approval covers all declared credentials, regardless of tier.
3. **Framework launches devcontainer** — generates a `devcontainer.json` from the manifest (including required CLI tool features), starts the container with the broker's MCP server available, and injects startup instructions derived from the manifest.
4. **Agent requests initial credentials** — following the startup instructions, the agent calls `request_access` via MCP for each credential declared in the manifest. The broker recognizes these as pre-approved and issues them immediately. The agent receives metadata about what was configured (kubectl contexts, git credential helpers, expiry times) and has full awareness of its access scope.
5. **Agent begins work** — operates within the approved scope, using the tools and contexts configured by the credential responses.
6. **Agent discovers additional need** — requests access to a resource not in the manifest via the same `request_access` MCP call. The framework evaluates the tier and routes to the appropriate approval path. If within the approval budget, lower-tier requests proceed with logging. If budget is exhausted or the request is high-tier, it escalates.
7. **High-risk operation** — agent needs to run terraform apply against production. The request goes to adversarial review. Adversarial agents confirm the request aligns with the task. The operation proceeds (or escalates to the operator if there's disagreement).
8. **Task completes** — all credentials are revoked. An audit summary is generated showing what access was requested, approved, and when. Actual usage auditing is left to the backing systems (GitHub audit log, Kubernetes audit log, CloudTrail, etc.).

---

## Language Choices

Different components use the language best suited to their role. The boundaries between components are well-defined protocols (MCP JSON-RPC, provider plugin protocol), not shared code, so mixed languages work cleanly.

- **Broker / MCP server**: TypeScript — the official MCP SDK is TypeScript-first, and the broker is fundamentally an MCP server. This is the most important integration to get right.
- **Credential providers**: Go — natural fit for the plugin model (single binary distribution, Terraform-style go-plugin patterns). Providers are independent processes that speak a protocol to the broker. If deep integration with Teleport client libraries is needed, Go is the only option.
- **Pre-MVP tools**: TypeScript — shares the language with the broker, so credential scoping code (GitHub JWT flow, tsh wrapping) translates directly into the MVP. The pre-MVP can evolve into a minimal MCP server prototype, becoming the seed of the actual broker rather than throwaway work.

---

## Implementation Phases

The implementation is structured as a series of phases, each building on the last. Every phase produces something immediately usable for safer agent operation, and every line of code moves toward the full framework.

---

### Phase 1: Credential Scoping CLI

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

The App installation is the bootstrap credential. Each `request_access` call derives a narrower token from it.

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

**Machine ID (tbot) integration** — the programmatic path for Wardlock's credential provider:

- `tbot` is a daemon for non-human identities. It authenticates via join tokens (supports AWS IAM, GCP SA, K8s SA — no interactive login) and continuously renews short-lived Teleport certs.
- The `kubernetes/v2` service type outputs a standard `kubeconfig.yaml` with embedded Teleport certs to a configured directory. This is the integration point: the Wardlock Teleport provider manages tbot lifecycle and injects the output kubeconfig as a `file` injection entry.
- On revocation, stop tbot and delete the kubeconfig. The Teleport certs in the kubeconfig are already short-lived and scoped to specific clusters/groups.

Because Teleport handles the credential isolation and proxying, Wardlock does not need to build its own TLS-intercepting proxy for Kubernetes when Teleport is available. Wardlock adds the request/approve/revoke lifecycle on top of Teleport's existing credential isolation.

For non-Teleport Kubernetes setups (direct API server access with bearer tokens or client certs), the proxy injection type described in open questions would be the alternative path.

**Teleport Application Access** — Teleport also provides an HTTPS reverse proxy for web applications and APIs. The architecture is the same pattern: client connects to the Teleport Proxy with Teleport certs, the Proxy forwards through a reverse tunnel to the Teleport Application Service, which rewrites requests (injects headers, JWTs) and forwards to the backend. Teleport has first-class credential injection for AWS (IAM role assumption, SigV4 signing), Azure, and GCP. For arbitrary SaaS APIs (like GitHub), Teleport can proxy the traffic and inject static headers, but lacks the dynamic credential management that Wardlock provides (issuing scoped short-lived tokens, rotation, revocation). The integration model for cloud APIs: Wardlock's Teleport provider leverages Teleport's native cloud support where available, and falls back to Wardlock's own credential providers (e.g., GitHub App tokens) with file injection for SaaS APIs that Teleport doesn't natively handle.

**Teleport as a credential provider** — In Wardlock's taxonomy, Teleport is a **credential provider**, not a new provider type. The Teleport provider's `issue()` call configures tbot to produce certs for the requested cluster or app and returns a `CredentialBundle` with `file` injection entries (kubeconfig, TLS certs). The fact that the kubeconfig points to the Teleport Proxy rather than the real K8s API server is an implementation detail of the credential — not something Wardlock needs to model separately. The proxy behavior is inherent to the credential: a Teleport cert naturally routes through the Teleport Proxy, the same way a GitHub credential helper naturally calls the GitHub API. Wardlock doesn't manage git's HTTPS connections — it just puts the credential helper in place. The provider taxonomy stays clean: credential providers produce bundles, isolation providers materialize them.

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

### Phase 2: Minimal MCP Server

**Goal**: Wrap the Phase 1 credential logic as an MCP server, validating the MCP integration pattern and the metadata-only response model. This becomes the seed of the actual broker.

**What gets built**:
- A minimal MCP server exposing `request_access` and `list_active` as MCP tools.
- Two hardcoded "providers" (GitHub and Kubernetes) using the Phase 1 credential logic internally.
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
4. Wrap Kubernetes credential logic as a `request_access` handler
5. Add `list_active` tool for the agent to check current credentials
6. Configure Claude Code to use the MCP server and test end-to-end

---

### Phase 3: Full MVP

**Goal**: Validate the complete framework loop — manifest → isolate → broker credentials → work → revoke — with the architecture described in this document.

**What gets built on top of Phase 2**:
- **Devcontainer isolation**: The framework generates a `devcontainer.json` from the task manifest, including required CLI tool features. The `devcontainer` CLI handles container creation and lifecycle. No inherited host credentials.
- **Task manifest**: YAML file defining expected credentials. Operator reviews and approves it before the task starts. Manifest generates startup instructions for the agent.
- **Tier 1 and 2 approval**: Auto-approve and rubber stamp. Skip adversarial review agents entirely.
- **Provider plugin interface**: Extract the hardcoded GitHub and Kubernetes logic into proper Go provider plugins speaking a defined protocol to the TypeScript broker.
- **mTLS**: Task-scoped client certificates for agent-to-broker authentication.
- **Approval budgets**: Per-tier limits on out-of-manifest credential requests.
- **File-based audit logging**: Append-only JSON lines log of all credential requests, approvals, and revocations.

**What this validates**:
- Can the full credential broker model support real multi-repo infrastructure tasks end-to-end?
- Does the task manifest capture enough information to be useful without being burdensome to author?
- Is the provider plugin interface general enough to support GitHub and Kubernetes/Teleport without provider-specific leaks into the core?
- Does running in a devcontainer with brokered credentials feel meaningfully safer than the current manual approach?
- Is devcontainer startup time acceptable, or does feature installation overhead warrant caching/snapshotting?
- What's the approval UX friction like in practice — does the tier model reduce interruptions to a tolerable level?

### Questions to Resolve Before Phase 3

These must be answered before Phase 3 implementation can begin. Phases 1 and 2 can proceed without them.

1. **Provider plugin protocol** — what is the exact protocol between the TypeScript broker and Go provider plugins? gRPC (like Terraform's go-plugin)? JSON over stdin/stdout? The provider contract interface is defined (see Provider Contract under Credential Brokering), but the transport needs to be decided.

2. **Devcontainer configuration** — the framework generates a `devcontainer.json` from the task manifest, mapping credential providers to devcontainer features (e.g., GitHub provider → `github-cli` feature, Kubernetes provider → `kubectl-helm-minikube` feature + `tsh`). For Phase 3, the mapping from providers to features can be hardcoded. Open questions: should the operator be able to add additional features beyond what the providers require (e.g., language runtimes, editors)? Should the framework support a base devcontainer.json that gets extended per task?

3. **mTLS certificate management** — the framework generates task-scoped client certificates for agent-to-broker auth. In solo mode, the framework acts as its own CA. What library/tooling handles cert generation? How are certs injected into the devcontainer alongside the MCP configuration?

---

## Open Questions

### Manifest Authoring UX

How do operators create task manifests efficiently? Writing YAML by hand for every task is a barrier to adoption.

Possible approaches:
- The framework could suggest a manifest based on past similar tasks or repository analysis.
- An agent could generate a draft manifest as part of task planning, which the operator then reviews and approves.
- Common task patterns could be captured as reusable manifest templates (e.g., "multi-repo terraform rollout" template that takes parameters).
- A specialized audit analysis agent (configured with the `audit_log` MCP tool, not the credential lifecycle tools) could review historical audit data and identify repeated out-of-manifest credential requests for certain task types, then suggest manifest template improvements. This closes the feedback loop: working agents generate audit data, analysis agents mine it, manifests get better over time.

### Provider Discovery and Configuration

How does the framework discover which providers are available and how they're configured?

Options:
- A configuration file similar to Terraform's provider blocks, declaring which providers are available and their connection details.
- A plugin directory where provider binaries are placed, with each provider self-describing its schema on startup.
- A registry for community providers, with local overrides for internal providers.

Related: how is provider configuration versioned and shared across an organization if the framework is used by multiple people?

### Provider Contract Sub-questions

The provider contract interface is defined in the main doc (see Provider Contract under Credential Brokering). Remaining open questions:

- How does the provider report errors (invalid resource, insufficient bootstrap permissions, rate limits)?
- Should the provider expose a `renew(credential_id, duration)` method, or should renewal go through the full `issue` path?
- Should the `metadata.injected` map support structured values (e.g., a list of configured repos) or stay strictly flat?

### Bootstrap Credential Sub-questions

The bootstrap credential model is defined in the main doc (see Bootstrap Credentials under Credential Brokering). Remaining open questions:

- In solo mode, what happens when the operator's session expires (e.g., Okta SSO session timeout)? Does the broker detect this and prompt the operator to re-authenticate?
- In organizational mode, how are bootstrap credentials rotated? Should the framework integrate with secret manager rotation mechanisms?
- In organizational mode, how are approval prompts routed to the right operator? A web UI? Slack notifications? Integration with existing on-call or approval workflows?
- How does the initial setup work for each mode — a setup wizard, manual configuration file, or integration with existing tooling (`tsh status`, `gh auth status`)?

### MCP Integration Sub-questions

The MCP integration design is defined in the main doc (see MCP Integration under Credential Brokering). Remaining open questions:

- How is the MCP server configured in the devcontainer? Can the framework inject the MCP server config into the devcontainer's agent CLI configuration automatically?
- For the Unix socket transport, what are the security implications of a bind-mounted socket? Can other processes in the container abuse it?
- Should the MCP server support notifications (MCP has a notification mechanism) to proactively inform the agent about expiring credentials?

### Credential Injection Sub-questions

The credential injection mechanism is defined in the main doc (see Credential Injection under Credential Brokering). Remaining open questions:

- On revocation, the isolation provider removes files and runs cleanup commands from the revocation bundle. But does the agent need to be notified that a credential was revoked, or is it sufficient for the next CLI command to simply fail?
- If multiple credential bundles write to the same file path (e.g., two Kubernetes clusters both targeting `.kube/config`), how are merges handled? Kubeconfig supports multiple contexts natively, but other tools may not. Should the isolation provider handle merging, or should credential providers use distinct file paths?
- For the `env` injection type (Phase 3): how does the isolation provider handle rotation for env vars that can't be updated after container start? Options include wrapper scripts, a sidecar that re-exports, or requiring the container to be restarted.
- **Proxy-based injection type (future):** The framework already operates a CA for mTLS between agent and broker. Could this CA infrastructure be extended to support a TLS-intercepting proxy as an injection type? The proxy would sit between the agent and the network, terminating TLS on the agent side (using dynamically generated certs signed by Wardlock's CA) and establishing new TLS connections to real destinations with real credentials. Two modes of credential injection are possible at the proxy layer:
  - **HTTP-layer injection**: The proxy terminates TLS, injects HTTP headers (e.g., `Authorization: Bearer ...`), and re-encrypts to the destination. Covers GitHub API tokens, SaaS API keys, and similar HTTP-based credentials.
  - **TLS-layer injection (mTLS)**: The proxy holds client certificates and presents them during the TLS handshake with the destination. The agent never sees the client cert or private key. This works for any protocol over TLS — the proxy doesn't need to understand the application layer, it just handles the TLS handshake differently on each side and passes application bytes through unmodified.
  - The proxy needs a routing table: which client cert or HTTP credential to use for which destination, matched on SNI hostname.
  - Revocation is stronger than file injection — removing the proxy rule immediately stops credential injection even if the agent cached the token in memory.
  - Explicit proxy (`HTTPS_PROXY` env var) is likely more pragmatic than transparent proxy (iptables redirect) — simpler, and the tools that don't respect `HTTPS_PROXY` are mostly the ones that need file-based credentials anyway.
  - **What the proxy cannot cover**: non-TLS protocols (SSH key-based auth on port 22) and tools with fully custom connection logic that bypass standard TLS/HTTP stacks.
  - **CA trust injection** adds complexity: the Wardlock CA cert must be added to the container's system trust store, plus language-specific stores (Node.js `NODE_EXTRA_CA_CERTS`, Python `REQUESTS_CA_BUNDLE`, Java keystore) and tool-specific config (`git http.sslCAInfo`). The isolation provider would handle this at container setup.
  - **Open design question**: should the mTLS CA (agent-to-broker auth) and the interception CA (proxy-generated destination certs) be the same CA or separate CAs? Separate CAs limit blast radius — compromising the interception CA only allows MITM of agent traffic, not impersonation of agents to the broker.
  - **Kubernetes API server analysis**: kubectl's connection to the API server supports both client certificate auth (real mTLS at the TLS handshake layer) and bearer token auth (HTTP `Authorization` header inside the TLS tunnel). Both are proxyable. Bearer token injection is the cleaner path — most managed Kubernetes (EKS, GKE, AKS) uses token-based auth, and even kubeadm clusters can issue ServiceAccount tokens. Client cert mTLS proxying also works (the proxy holds the private key and presents the cert to the API server) but identity is baked into the TLS session (CN/O fields), making per-request identity switching impossible. kubectl still needs a kubeconfig on disk to know where to connect, but that kubeconfig is just a non-secret pointer to the proxy address — no credential material in it. Additional proxy complexity for Kubernetes: WebSocket/SPDY upgrades (`kubectl exec`, `kubectl port-forward`) and long-lived watch streams (`kubectl get pods -w`) require the proxy to handle HTTP upgrades and chunked streaming. **Note**: For Teleport-managed clusters, the proxy injection type is unnecessary — Teleport already provides credential isolation via its own proxy and impersonation model (see Kubernetes implementation notes under Pre-MVP). The Wardlock proxy would be relevant for non-Teleport K8s setups with direct API server access.
  - This would be an additional injection type alongside `file` and `env`, not a replacement. A credential provider's bundle could mix types — e.g., `proxy` for the API token and `file` for a non-secret kubeconfig pointing to the proxy. Worth evaluating once file injection is proven and the mTLS CA infrastructure is mature.

### Multi-Agent Coordination

If multiple agents are working on related tasks concurrently, how are their credential scopes managed?

Concerns:
- Two agents with write credentials to the same repo could create conflicting changes.
- Two agents with write credentials to the same Kubernetes cluster could interfere with each other's deployments.
- Credential revocation for one task should not affect another task's active credentials.

Possible approaches:
- Each task gets its own isolated credentials, even if they access the same resources. No sharing.
- The framework tracks which resources have active write credentials and warns or blocks when a second task requests write access to the same resource.
- This may be out of scope for the MVP if the initial use case is a single operator running one task at a time.

### Credential Caching and Sharing

If two tasks (or two credential requests within the same task) need read access to the same resource, should they share a credential or each get their own?

Sharing reduces API calls and avoids rate limits, but complicates revocation (revoking one task's access revokes the other's). Separate credentials are simpler and safer but may hit provider rate limits or token caps.

### Failure Modes

What happens when things go wrong mid-task?

Scenarios to consider:
- A credential expires mid-operation (e.g., during a terraform apply that takes longer than the TTL).
- The broker process crashes or becomes unreachable while the agent has active credentials.
- A provider backend is unavailable when the agent requests a new credential.
- The agent's container crashes with active credentials — who revokes them?

For each scenario: does the agent pause and retry, fail the task, or continue with degraded access? How is the operator notified?

### Rollback Integration

Could the framework integrate with rollback mechanisms to provide automated recovery when operations go wrong?

Possible integrations:
- Git: automatic revert commits or branch deletion if a task fails partway through.
- Terraform: state rollback or targeted destroy of resources created during a failed task.
- Kubernetes: rollback to previous deployment revision.

This is likely out of scope for the MVP but worth designing for. The framework's audit log tracks what credentials were issued and when, and the backing systems' own audit logs (GitHub, CloudTrail, Kubernetes audit) track what was actually done with those credentials. Correlating across these systems to determine what to roll back is a significant undertaking and is not a near-term goal.

### State and Storage

Where does the framework store its runtime state?

State includes:
- Active credential records (ID, provider, resource, permission, TTL, task association).
- Audit log entries.
- Task manifest and budget state (how many out-of-manifest requests have been made per tier).
- Provider configuration and bootstrap credential references.

Options range from flat files (simplest, fine for single-operator MVP) to a lightweight database (SQLite) to a proper backend (for multi-operator or multi-machine setups).

### Adversarial Agent Protocol

What exactly do adversarial review agents receive, return, and how is consensus evaluated?

To define before implementing Tier 3:
- **Input**: the task description, the specific credential request (provider, resource, permission, reason), and the task manifest. NOT the requesting agent's full conversation context.
- **Output**: an approve/deny decision with a written justification.
- **Consensus**: how many adversarial agents are consulted? Is it configurable? What constitutes agreement vs. disagreement?
- **Tuning**: how are adversarial agents prompted to be appropriately skeptical without being obstructive?
- **Failure**: what happens if an adversarial agent times out or errors? Is it treated as a denial, or is a replacement consulted?

### Operator Override Match Schema

The operator override YAML currently uses provider-specific match fields (`resource_type` for Kubernetes sub-resource kinds, `ref` for GitHub branches) alongside the universal `resource` field inside the `match` block. The question is whether these provider-specific fields should be well-known names or a fully generic match map where the provider declares which additional dimensions it supports via its `schema()`.

A generic approach would mean the framework only validates `resource` as a universal match key, and passes all other keys in `match` through to the provider for validation. This keeps the override schema clean and avoids baking in field names that only make sense for specific providers. To decide when Phase 3 override implementation begins — real usage of overrides in Phase 2 testing may inform the right design.

### Manifest Wildcard Matching

When the manifest uses wildcards (e.g., `resource: "org/customer-*-infra"`) and the agent requests a specific resource (`org/customer-a-infra`), the broker needs to determine whether this is a pre-approved request. The matching algorithm is unspecified.

Options:
- **Glob matching** — simple, familiar, but matches against the pattern syntactically without validating that the resource exists. `org/customer-evil-infra` would also match.
- **Provider-validated matching** — the broker passes the manifest pattern and the actual request to the provider, which checks both the pattern match and whether the resource actually exists and is accessible. This is more correct because the provider knows its resource namespace (e.g., the GitHub provider can verify the repo exists and is covered by the App installation).
- **Exact match only** — safest but defeats the purpose of wildcards in manifests.

To decide before Phase 3 manifest implementation. Phase 1-2 have no manifests and are unaffected.

### Provider Language Migration (TypeScript → Go)

Phase 1-2 build GitHub and Kubernetes credential logic in TypeScript (hardcoded in the broker). Phase 3 specifies extracting these into Go provider plugins speaking a protocol to the broker. This implies rewriting the credential logic in Go.

Options:
- **Accept the rewrite** — Phase 1-2 TypeScript validates the logic and design. Phase 3 rewrites in Go for production. The TypeScript code is a prototype.
- **Keep providers in TypeScript** — drop the Go requirement. Providers are still separate processes speaking a protocol, just in TypeScript. Loses single-binary distribution and direct Teleport client library access.
- **Phase 2.5 extraction** — extract to Go plugins before Phase 3 adds isolation/manifests/tiers, isolating the language migration from the feature additions.

The strongest argument for Go is direct Teleport client library access (avoiding `tsh` CLI wrapping). If `tsh` wrapping proves sufficient through Phase 1-2, Go may not be necessary. Let real usage inform this decision.
