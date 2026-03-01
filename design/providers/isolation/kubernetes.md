# Kubernetes Isolation Provider

Runs the agent's container as a Kubernetes pod. More relevant for multi-agent or cloud-native scenarios.

## What It Provides

- Full container isolation in a managed cluster
- Native integration with Kubernetes secrets for credential injection
- Scalable — multiple agent pods can run concurrently
- Suitable for organizational deployment where agents are a platform service

## Supported Injection Types

| Injection Type | Implementation |
|---|---|
| `file` | Create Secret, mount into pod |
| `env` | Pod env spec or ConfigMap |

## Backend Identity

| Adoption Phase | Identity Source |
|---|---|
| Organizational | Dedicated ServiceAccount for pod management, or broker-signed JWTs if the K8s cluster trusts the broker as an OIDC issuer |

## Communication

The broker's MCP gateway is deployed as a service in the cluster. Agent pods connect via in-cluster networking or a Unix socket mounted from a shared volume.

## Adoption Phase Relevance

Used in Adoption Phase 3-4 (multi-practitioner and organizational orchestration). Isolation workers authenticate to the control plane via mTLS and receive scoped JWTs per job.

## Note on Naming

This is the Kubernetes **isolation** provider — it runs agent containers as K8s pods. This is distinct from the Teleport credential provider, which issues scoped credentials *for* Kubernetes clusters. Different roles, same backend technology.
