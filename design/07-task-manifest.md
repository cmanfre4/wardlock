# Task Manifest and Approval Budgets

## Task Manifest

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

### Manifest as Startup Instructions

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

## Approval Budgets

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

## Open Questions

### Manifest Authoring UX

How do operators create task manifests efficiently? Writing YAML by hand for every task is a barrier to adoption.

Possible approaches:
- The framework could suggest a manifest based on past similar tasks or repository analysis.
- An agent could generate a draft manifest as part of task planning, which the operator then reviews and approves.
- Common task patterns could be captured as reusable manifest templates (e.g., "multi-repo terraform rollout" template that takes parameters).
- A specialized audit analysis agent (configured with the `audit_log` MCP tool, not the credential lifecycle tools) could review historical audit data and identify repeated out-of-manifest credential requests for certain task types, then suggest manifest template improvements. This closes the feedback loop: working agents generate audit data, analysis agents mine it, manifests get better over time.

### Manifest Wildcard Matching

When the manifest uses wildcards (e.g., `resource: "org/customer-*-infra"`) and the agent requests a specific resource (`org/customer-a-infra`), the broker needs to determine whether this is a pre-approved request. The matching algorithm is unspecified.

Options:
- **Glob matching** — simple, familiar, but matches against the pattern syntactically without validating that the resource exists. `org/customer-evil-infra` would also match.
- **Provider-validated matching** — the broker passes the manifest pattern and the actual request to the provider, which checks both the pattern match and whether the resource actually exists and is accessible. This is more correct because the provider knows its resource namespace (e.g., the GitHub provider can verify the repo exists and is covered by the App installation).
- **Exact match only** — safest but defeats the purpose of wildcards in manifests.

To decide before Phase 3 manifest implementation. Phase 1-2 have no manifests and are unaffected.
