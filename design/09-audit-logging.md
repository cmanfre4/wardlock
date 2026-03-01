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
