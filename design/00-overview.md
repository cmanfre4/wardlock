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

## Key Design Principles

**Secrets never enter the model's context through the framework.** The credential management flow returns metadata about issued credentials (status, expiry, what was configured) — not actual secret material. Secrets are injected into the agent's filesystem as a side effect of the `request_access` call. The model needs the *capability* ("kubectl is now configured for cluster X"), not the *secret* (the certificate bytes). This ensures the default flow doesn't leak secrets through model outputs, conversation logs, or prompt injection attacks.

**Credential providers and isolation providers are fully decoupled.** Credential providers produce declarative bundles describing what needs to exist in the container. Isolation providers materialize those bundles. A GitHub credential provider doesn't know whether it's running on localhost or in a Kubernetes pod — it produces the same bundle either way. This follows the Terraform provider model: a core framework handles common concerns while provider plugins implement a consistent interface for each backend system.

**The agent doesn't know or care about the broker's deployment model.** It makes the same `request_access` MCP calls and gets the same metadata responses whether the broker is a local process on the operator's laptop or a clustered service behind a load balancer. Task manifests, provider configurations, and the tier system are stable across all deployment modes — what changes is the broker's identity sources, deployment model, and auth layer.

## Core Architecture

The framework has two API surfaces and two provider types, connected by a credential broker.

**API surfaces:**

- **MCP server** (agent-facing) — exposes `request_access`, `revoke`, and `list_active` as MCP tools. The agent discovers and calls them like any other MCP server. This is the primary integration point.
- **HTTP API** (operator-facing) — the CLI and other tooling communicate with the broker over HTTP. This is a separate surface from the agent's MCP interface, present from day one. Self-documenting via OpenAPI spec.

**Provider types:**

- **Credential providers** — know how to issue and revoke scoped, time-limited credentials for a specific backend (GitHub, Teleport, AWS). They produce credential bundles against a common provider contract.
- **Isolation providers** — know how to run an agent in an isolated environment and how to materialize credential bundles in that environment. They implement injection and cleanup for each injection type (`file`, `env`) using the mechanics of their container or VM technology.

**The broker** sits between these, orchestrating the credential lifecycle: routing requests to credential providers, evaluating approval tiers, handing bundles to isolation providers for injection, tracking expiry, and logging everything. It never interprets or logs secret material — bundles pass through as opaque bytes.

**Pluggable infrastructure:** The broker's internal state (credential records, audit log, approval queue) is persisted through three pluggable interfaces — StateStore, WorkQueue, and EncryptionProvider. In the simplest deployment these are in-memory data structures. At scale they become DynamoDB, SQS, KMS or equivalent. The decomposition exists at the interface level from day one.

**Broker decomposition at scale:** The single broker process naturally decomposes into a control plane (decisions, policy, audit, CA), an MCP gateway (stateless proxy), credential workers (run credential provider plugins), and isolation workers (run isolation provider plugins). The control plane never touches secret material. Workers authenticate per-job and hold no long-lived secrets.

## Adoption Modes

The framework is designed to support four progressive adoption modes. Each mode introduces architectural requirements, but the agent-facing interface stays the same across all of them:

1. **Solo practitioner, local broker** — single process on the operator's laptop. Broker identity derived from the operator's existing sessions (GitHub App key on disk, existing `tsh` login). No auth, no TLS — the trust boundary is the operator's machine.

2. **Solo practitioner, remote broker** — broker moves to cloud infrastructure. Forces real auth, durable state, TLS, and a rethink of the approval UX. The hardest architectural jump because it touches everything.

3. **Multi-practitioner, OIDC auth** — multiple operators share a broker. CLI authenticates via organizational OIDC. Broker identity moves from personal sessions to organizational service identities. Authorization, audit attribution, and identity-scoped credential access become real.

4. **Organizational orchestration** — CI/CD pipelines and internal platforms integrate with the broker directly. Clustering, API stability, multi-tenancy, and delegated approval via policy.

The adoption phases document describes these modes in full detail. The implementation phases document describes the build order to get there.

## Tiered Approval

Not all credential requests carry the same risk. The framework uses a four-tier approval model:

- **Tier 1 (auto-approve)** — read access to non-sensitive resources. Logged, no friction.
- **Tier 2 (rubber stamp)** — low-risk writes. Auto-approved with enhanced logging and agent self-justification.
- **Tier 3 (adversarial review)** — high blast radius operations. Reviewed by independent adversarial agents with isolated context before proceeding.
- **Tier 4 (human approval)** — highest risk, or escalations from Tier 3. Requires explicit operator approval.

Providers supply default tier classifications based on credential scope. Operators override these based on their own environment knowledge (a dev sandbox account might be Tier 2 everywhere; a production cluster might be Tier 4 even for reads on secrets).

Task manifests let operators pre-approve a set of credentials before the task begins, turning the approval UX from "approve every action" into "approve the plan, then only intervene on deviations."

## Example Workflow

In the simplest mode (solo practitioner, local broker, no isolation):

1. Operator starts the broker locally and launches an agent session with the broker's MCP server configured.
2. Agent calls `request_access` for the credentials it needs. The broker issues scoped, short-lived credentials and injects them into the filesystem. The agent receives metadata about what was configured.
3. Agent works using the configured tools (git, kubectl, etc.) within the credential scope.
4. Credentials expire or are revoked when the session ends. Everything is logged.

In the full-featured mode (manifest, devcontainer isolation, tiered approval):

1. Operator defines a task manifest declaring expected credentials and reviews it.
2. Framework launches a devcontainer from the manifest with the broker's MCP server available.
3. Agent requests each credential declared in the manifest — these are pre-approved and issued immediately.
4. Agent works within approved scope. Mid-task credential requests are evaluated against the tier system and approval budget.
5. High-risk requests go through adversarial review or escalate to the operator.
6. Task completes — all credentials revoked, audit summary generated.

The framework is useful at every point along this spectrum. The simplest mode replaces the manual credential setup workflow. Each additional capability (manifests, isolation, tiers, adversarial review) adds safety without changing the core interaction pattern.

