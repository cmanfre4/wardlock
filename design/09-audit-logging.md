# Audit Logging

All credential requests, approvals, denials, and revocations are logged.

## Log Contents

- Timestamp
- Requesting agent / task ID
- Resource and permission level requested
- Reason provided
- Approval tier and outcome (auto-approved, rubber-stamped, adversarial-reviewed, human-approved, denied)
- Adversarial agent assessments (if applicable)
- Credential ID, TTL, and revocation time

## Alerting and Anomaly Detection

An audit log is only valuable if something is reviewing it. The framework should support:

- Alerting on denied requests or adversarial agent disagreements
- Anomaly detection on credential request patterns (e.g., an agent requesting significantly more access than its [task manifest](07-task-manifest.md) declared)
- Post-task review summaries showing what access was requested vs. what was declared in the manifest

## Storage Progression

Audit log storage evolves with [adoption phases](11-adoption-and-scaling.md):

- **Single-operator (Adoption Phase 1-2)**: File-based. Append-only JSON lines log on the broker's filesystem (or in the [StateStore](11-adoption-and-scaling.md#broker-decomposition-at-scale) alongside credential metadata).
- **Multi-practitioner (Adoption Phase 3+)**: Centralized. Audit logs move to an organizational logging system as the broker becomes a shared service. Every credential request is tied to an authenticated OIDC identity, not just "the operator."

At scale, audit logging is one of the [control plane's responsibilities](11-adoption-and-scaling.md#broker-decomposition-at-scale) — it stays in the stateful core rather than being delegated to workers.
