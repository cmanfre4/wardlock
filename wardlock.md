# Wardlock (`wlk`)

A framework for isolating agents and providing scoped, time-limited permissions to external resources so they can work autonomously with bounded blast radius.

Named for the ward — the internal mechanism of a lock that determines which key fits. Wardlock determines what level of access an agent can request, and gates the approval required to grant it.

## Design Documents

| # | Component | Description |
|---|---|---|
| 00 | [Overview](design/00-overview.md) | Problem statement, scope, core architecture, example workflow, language choices |
| 01 | [Isolation](design/01-isolation.md) | Isolation providers, what isolation provides, communication channels |
| 02 | [Credential Brokering](design/02-credential-brokering.md) | Broker interface, credential lifecycle, time limits, MCP integration |
| 03 | [Credential Providers](design/03-credential-providers.md) | GitHub, AWS, Kubernetes/Teleport, Vault providers; provider contract; credential & revocation bundles |
| 04 | [Credential Injection](design/04-credential-injection.md) | Three-way collaboration (provider → broker → isolation), implementation matrix |
| 05 | [Broker Identity](design/05-broker-identity.md) | Backend auth, JWT/OIDC issuer, pluggable identity sources, agent-to-broker mTLS |
| 06 | [Tiered Approval](design/06-tiered-approval.md) | Tiers 1-4, classification system, provider defaults, operator overrides |
| 07 | [Task Manifest](design/07-task-manifest.md) | Manifest format, startup instructions, approval budgets |
| 08 | [Adversarial Review](design/08-adversarial-review.md) | Design principles, evaluation criteria, adversarial agent protocol |
| 09 | [Audit Logging](design/09-audit-logging.md) | Log contents, alerting, anomaly detection |
| 10 | [Implementation Phases](design/10-implementation-phases.md) | Phase 1 (CLI), Phase 2 (MCP server), Phase 3 (full MVP) |
| 11 | [Adoption & Scaling](design/11-adoption-and-scaling.md) | Adoption phases 1-4, broker decomposition, pluggable infrastructure |
| 12 | [Open Questions](design/12-open-questions.md) | Cross-cutting open questions; links to component-specific questions |

## Related Documents

- [Threat Model](threat_model.md)
