# Open Questions

These are cross-cutting open questions that span multiple components. Component-specific open questions are co-located with their respective design documents:

- [Credential Brokering — MCP Integration Sub-questions](02-credential-brokering.md#mcp-integration-sub-questions)
- [Credential Providers — Provider Discovery, Provider Contract Sub-questions](03-credential-providers.md#open-questions)
- [Credential Injection — Injection Sub-questions, Proxy-based Injection](04-credential-injection.md#open-questions)
- [Broker Identity — Sub-questions](05-broker-identity.md#open-questions)
- [Tiered Approval — Operator Override Match Schema](06-tiered-approval.md#open-questions)
- [Task Manifest — Manifest Authoring UX, Wildcard Matching](07-task-manifest.md#open-questions)
- [Adversarial Review — Adversarial Agent Protocol](08-adversarial-review.md#open-questions)
- [Implementation Phases — Provider Language Migration](10-implementation-phases.md#open-questions)

---

## Multi-Agent Coordination

If multiple agents are working on related tasks concurrently, how are their credential scopes managed?

Concerns:
- Two agents with write credentials to the same repo could create conflicting changes.
- Two agents with write credentials to the same Kubernetes cluster could interfere with each other's deployments.
- Credential revocation for one task should not affect another task's active credentials.

Possible approaches:
- Each task gets its own isolated credentials, even if they access the same resources. No sharing.
- The framework tracks which resources have active write credentials and warns or blocks when a second task requests write access to the same resource.
- This may be out of scope for the MVP if the initial use case is a single operator running one task at a time.

## Credential Caching and Sharing

If two tasks (or two credential requests within the same task) need read access to the same resource, should they share a credential or each get their own?

Sharing reduces API calls and avoids rate limits, but complicates revocation (revoking one task's access revokes the other's). Separate credentials are simpler and safer but may hit provider rate limits or token caps.

## Failure Modes

What happens when things go wrong mid-task?

Scenarios to consider:
- A credential expires mid-operation (e.g., during a terraform apply that takes longer than the TTL).
- The broker process crashes or becomes unreachable while the agent has active credentials.
- A provider backend is unavailable when the agent requests a new credential.
- The agent's container crashes with active credentials — who revokes them?

For each scenario: does the agent pause and retry, fail the task, or continue with degraded access? How is the operator notified?

## Rollback Integration

Could the framework integrate with rollback mechanisms to provide automated recovery when operations go wrong?

Possible integrations:
- Git: automatic revert commits or branch deletion if a task fails partway through.
- Terraform: state rollback or targeted destroy of resources created during a failed task.
- Kubernetes: rollback to previous deployment revision.

This is likely out of scope for the MVP but worth designing for. The framework's [audit log](09-audit-logging.md) tracks what credentials were issued and when, and the backing systems' own audit logs (GitHub, CloudTrail, Kubernetes audit) track what was actually done with those credentials. Correlating across these systems to determine what to roll back is a significant undertaking and is not a near-term goal.

## State and Storage

Where does the framework store its runtime state?

State includes:
- Active credential records (ID, provider, resource, permission, TTL, task association).
- Audit log entries.
- Task manifest and budget state (how many out-of-manifest requests have been made per tier).
- Provider configuration and [broker identity](05-broker-identity.md) references.

Options range from flat files (simplest, fine for single-operator MVP) to a lightweight database (SQLite) to a proper backend (for multi-operator or multi-machine setups). See also the [pluggable infrastructure backends](11-adoption-and-scaling.md#broker-decomposition-at-scale) discussion for how state storage evolves with adoption phases.
