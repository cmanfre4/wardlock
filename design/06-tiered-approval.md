# Tiered Approval

Not all credential requests carry the same risk. The framework uses a tiered approval model that concentrates human attention where it matters most.

## Tier 1: Auto-approve

**Read/list access to non-sensitive resources.**

- Auto-approved immediately
- Logged for audit
- Examples: reading terraform state, listing repos in an org, reading Kubernetes pod status, viewing CI workflow logs

## Tier 2: Rubber Stamp

**Low blast radius writes, or reads on sensitive resources.**

- Auto-approved but with additional safeguards
- Agent must provide explicit reasoning for the request
- Logged with enhanced detail
- May trigger a brief "pause and reflect" step where the agent re-evaluates whether the access is truly necessary
- Examples: writing to a non-production branch, reading secrets from a dev Vault path, modifying a staging-only resource

## Tier 3: Adversarial Review

**High blast radius operations.**

- Reviewed by one or more independent [adversarial agents](08-adversarial-review.md) before execution
- Adversarial agents receive the task description and the permission request, but NOT the full working context of the requesting agent (context isolation prevents shared poisoning)
- If adversarial agents unanimously approve, the request proceeds
- If any adversarial agent denies or agents disagree, the request escalates to Tier 4
- Disagreement itself is treated as a signal worth escalating — not resolved by majority vote
- Logged with full review chain
- Examples: writing to a production Kubernetes cluster, modifying IAM policies, changing Vault access policies, terraform apply against production

## Tier 4: Human Approval

**Highest blast radius, or escalations from Tier 3.**

- Requires explicit human approval
- Presented with full context: what's being requested, why, what the adversarial agents said (if applicable)
- Logged with approval decision and approver
- Examples: operations that could cause customer impact, broad permission requests, anything the adversarial agents flagged as concerning

## Tier Classification

Tier classification determines which approval tier a given credential request falls into. This is a layered system: providers supply sensible defaults based on their domain knowledge, and operators can override or refine those defaults based on their own risk tolerance, organizational policies, and environment-specific context.

### How Classification Works

When a credential request comes in, the framework evaluates it against classification rules in priority order:

1. **Operator overrides** (highest priority) — explicit rules defined by the operator that target specific resources, accounts, clusters, or paths. These override provider defaults based on the operator's knowledge of their own environment.
2. **Provider defaults** (lowest priority) — the baseline classification the provider ships with, based on resource type and permission level.

The first matching rule wins. This means a provider can ship with reasonable out-of-the-box behavior, but the operator always has final say over how risk is categorized in their environment.

### Provider Default Classifications

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

### Operator Overrides

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

Override match rules use the same `resource` and `permission` vocabulary as the [broker interface](02-credential-brokering.md) and [task manifest](07-task-manifest.md). The `resource_type` field is an optional secondary match dimension available when a provider exposes sub-resource classification (e.g., Kubernetes resource kinds within a cluster, Vault secret engine paths within a mount). The `ref` field in the GitHub example is also a provider-specific match refinement — providers declare which additional match dimensions they support in their schema, and the override system passes them through.

### Classification Dimensions

Tier classification can take into account multiple dimensions, not just resource type and permission level:

- **Environment** — production vs. staging vs. dev. The same operation (e.g., kubectl delete pod) might be Tier 2 in staging and Tier 4 in production.
- **Resource sensitivity** — whether the resource contains customer data, secrets, or access control configurations.
- **Blast radius** — how many systems, customers, or services could be affected. Deleting a single pod is different from deleting a namespace.
- **Reversibility** — whether the operation can be cleanly undone. `terraform plan` is always safe. `terraform apply` on a resource that can be recreated is less risky than `terraform apply` on a stateful resource like a database.
- **Scope breadth** — a request for write access to one specific repo is less risky than write access to `org/*`.

### Credential-Level vs. Operation-Level Gating

The tier classification system described above operates at the **credential level** — it determines the approval required before a credential is issued. Once issued, the agent can perform any operation that credential permits.

This is an important boundary. The framework does not intercept individual commands (`kubectl delete`, `terraform apply`, `vault write`). That would require deep integration with every CLI tool and introduce significant complexity.

However, there are cases where the risk of a credential can't be fully assessed until the agent reveals how it intends to use it. For example, a `ReadOnlyAccess` AWS credential is low risk regardless of usage, but a write credential to a Kubernetes cluster could be used for a benign config update or a destructive namespace deletion.

Two mechanisms address this gap:

1. **Prefer narrower credentials** — rather than issuing a broad credential and hoping the agent uses it responsibly, encourage the agent to request the narrowest credential possible. An agent that needs to update a deployment in a specific namespace should request a namespace-scoped role, not cluster-admin. The tier system naturally incentivizes this: narrower credentials get lower tiers and faster approval.

2. **Operation-level gating (future extension)** — a separate, optional layer that could intercept specific high-risk commands before execution. This is architecturally distinct from credential brokering and significantly more complex to implement. It is noted here as a potential extension, not a core requirement. If implemented, it would function as a command allowlist/denylist within the isolated environment, independent of the credential tier system.

---

## Open Questions

### Operator Override Match Schema

The operator override YAML currently uses provider-specific match fields (`resource_type` for Kubernetes sub-resource kinds, `ref` for GitHub branches) alongside the universal `resource` field inside the `match` block. The question is whether these provider-specific fields should be well-known names or a fully generic match map where the provider declares which additional dimensions it supports via its `schema()`.

A generic approach would mean the framework only validates `resource` as a universal match key, and passes all other keys in `match` through to the provider for validation. This keeps the override schema clean and avoids baking in field names that only make sense for specific providers. To decide when Phase 3 override implementation begins — real usage of overrides in Phase 2 testing may inform the right design.
