# Remote VM Isolation Provider

For scenarios where local resources are insufficient or stronger isolation guarantees are needed than containers provide.

## What It Provides

- Full VM-level isolation
- Stronger security boundary than containers (separate kernel)
- Suitable for workloads that require elevated privileges or specialized hardware

## Supported Injection Types

| Injection Type | Implementation |
|---|---|
| `file` | scp to host |
| `env` | Export in shell profile |

## Backend Identity

| Adoption Phase | Identity Source |
|---|---|
| Solo/Organizational | Cloud provider identity (AWS, GCP, Azure SDK auth) |

## Adoption Phase Relevance

A future isolation provider for scenarios where container isolation is insufficient. Not a near-term priority.
