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

The framework has three pillars: **Isolation**, **[Credential Brokering](02-credential-brokering.md)**, and **Approval & Audit**.

The design follows the Terraform provider model: a core framework handles common concerns (isolation lifecycle, approval flows, audit logging, credential injection and revocation), while provider plugins implement a consistent interface for each backend system.

## Design Components

| Component | File |
|---|---|
| Isolation | [01-isolation.md](01-isolation.md) |
| Credential Brokering | [02-credential-brokering.md](02-credential-brokering.md) |
| Credential Providers | [03-credential-providers.md](03-credential-providers.md) |
| Credential Injection | [04-credential-injection.md](04-credential-injection.md) |
| Broker Identity | [05-broker-identity.md](05-broker-identity.md) |
| Tiered Approval | [06-tiered-approval.md](06-tiered-approval.md) |
| Task Manifest & Approval Budgets | [07-task-manifest.md](07-task-manifest.md) |
| Adversarial Review Agents | [08-adversarial-review.md](08-adversarial-review.md) |
| Audit Logging | [09-audit-logging.md](09-audit-logging.md) |
| Implementation Phases | [10-implementation-phases.md](10-implementation-phases.md) |
| Adoption & Scaling | [11-adoption-and-scaling.md](11-adoption-and-scaling.md) |
| Open Questions | [12-open-questions.md](12-open-questions.md) |

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
