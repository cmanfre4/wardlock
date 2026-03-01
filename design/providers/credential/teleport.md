# Teleport Credential Provider

Teleport provides credential isolation for multiple backend resource types — Kubernetes clusters, Vault, databases, and web applications. The agent never possesses credentials that work directly against the real backend; all access is proxied through Teleport.

## Approach

Teleport already issues short-lived certificates scoped to specific resources and roles. The Wardlock Teleport provider manages the tbot lifecycle and injects the output credentials as file injection entries. Wardlock adds the request/approve/revoke lifecycle on top of Teleport's existing credential isolation.

In Wardlock's taxonomy, Teleport is a **credential provider**, not a new provider type. The Teleport provider's `issue()` call configures tbot to produce certs for the requested resource and returns a `CredentialBundle` with `file` injection entries. The fact that credentials route through the Teleport Proxy rather than directly to backends is an implementation detail of the credential — not something Wardlock needs to model separately.

## Resource Types

The Teleport provider supports multiple resource types through a single provider interface:

| Resource Type | Description |
|---|---|
| `kubernetes` | Scoped certificates for Kubernetes clusters registered with Teleport |
| `vault` | Short-lived Vault tokens scoped to specific secret paths and capabilities |
| `database` | Database credentials (future) |
| `app` | Web application/API access (future) |

## Resource and Permission Vocabulary

- **`resource`**: A Teleport-registered resource identifier. For Kubernetes, this is the cluster name (e.g., `customer-a-staging`). For Vault, this is the Vault mount/path.
- **`permission`**: A Teleport role name that determines the level of access (e.g., `readonly`, `namespace-admin`). The Teleport role maps to backend-specific permissions (Kubernetes RBAC groups, Vault policies, etc.).

## Connection Architecture

The agent never connects directly to backend services. The connection chain for Kubernetes:

```
kubectl → (Teleport cert, mTLS) → Teleport Proxy → (reverse tunnel) → Teleport K8s Service → (impersonation headers) → K8s API Server
```

- `tsh kube login` issues an x509 cert signed by **Teleport's User CA** — not a Kubernetes cert. The cert encodes Kubernetes groups/users in custom ASN.1 extensions (OID `1.3.9999.1.*`). This cert is useless against the K8s API server directly — it only works against the Teleport Proxy.
- The kubeconfig points to the **Teleport Proxy address**, not the K8s API server. The agent never learns the real API server address.
- The Teleport Kubernetes Service connects to the real K8s API server using **its own service account** and translates Teleport identity into **Kubernetes impersonation headers** (`Impersonate-User`, `Impersonate-Group`). Standard K8s RBAC applies from there.
- Certificate TTL is controlled per-role via `max_session_ttl` (default 12h, hard cap 30h). Teleport takes the minimum across all applicable roles.

Because Teleport handles the credential isolation and proxying, Wardlock does not need to build its own TLS-intercepting proxy for Kubernetes when Teleport is available. The proxy injection type would be relevant for non-Teleport setups with direct API server access.

## Teleport Application Access

Teleport also provides an HTTPS reverse proxy for web applications and APIs. The architecture is the same pattern: client connects to the Teleport Proxy with Teleport certs, the Proxy forwards through a reverse tunnel to the Teleport Application Service, which rewrites requests (injects headers, JWTs) and forwards to the backend.

Teleport has first-class credential injection for AWS (IAM role assumption, SigV4 signing), Azure, and GCP. For arbitrary SaaS APIs (like GitHub), Teleport can proxy the traffic and inject static headers, but lacks the dynamic credential management that Wardlock provides (issuing scoped short-lived tokens, rotation, revocation).

The integration model for cloud APIs: the Teleport provider leverages Teleport's native cloud support where available, and falls back to other Wardlock credential providers (e.g., the GitHub provider for GitHub App tokens) for SaaS APIs that Teleport doesn't natively handle.

## Implementation

### Machine Identity (tbot)

`tbot` is Teleport's daemon for non-human identities. It is the programmatic integration point for the Wardlock Teleport provider:

- Authenticates via join tokens (supports AWS IAM, GCP SA, K8s SA — no interactive login) and continuously renews short-lived Teleport certs.
- The `kubernetes/v2` service type outputs a standard `kubeconfig.yaml` with embedded Teleport certs to a configured directory. The Wardlock Teleport provider manages tbot lifecycle and injects the output kubeconfig as a `file` injection entry.
- On revocation, stop tbot and delete the kubeconfig. The Teleport certs in the kubeconfig are already short-lived and scoped to specific clusters/groups.

TypeScript: `child_process.execFile` to wrap `tsh`/`tbot`, `js-yaml` for kubeconfig manipulation.

### Solo vs. Organizational Identity

| Adoption Phase | Identity Source |
|---|---|
| Solo | Operator's existing `tsh` session (from their normal SSO login) — `source: local_tsh_session` |
| Organizational | tbot machine identity or broker-signed JWT verified against JWKS — `source: broker_jwt` |

In solo mode, the provider uses the operator's existing `tsh` session to request scoped cluster certificates. The operator already has access — the framework just requests narrower roles.

In organizational mode, the broker can register as a trusted OIDC issuer with Teleport. Teleport accepts broker-signed JWTs to issue scoped certificates, replacing the tbot join token model.

## Injection

### Credential Bundle (Kubernetes)

```json
{
  "credential_id": "cred-k8s-def456",
  "expires": "2026-02-22T15:00:00Z",
  "injections": [
    {
      "type": "file",
      "path": ".kube/config",
      "content": "YXBpVmVyc2lvbjogdjEKa2luZDogQ29uZmlnCmNsdXN0ZXJzOi..."
    }
  ],
  "metadata": {
    "injected": {
      "kubeconfig": "~/.kube/config",
      "context": "customer-a-staging"
    },
    "note": "kubectl configured for cluster customer-a-staging with readonly access."
  }
}
```

Decoded injection contents:

`.kube/config`:
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <Teleport Proxy CA>
    server: https://teleport.example.com:443
    tls-server-name: kube-teleport-proxy-alpn.teleport.example.com
  name: customer-a-staging
contexts:
- context:
    cluster: customer-a-staging
    user: customer-a-staging
  name: customer-a-staging
current-context: customer-a-staging
users:
- name: customer-a-staging
  user:
    client-certificate-data: <Teleport-issued short-lived cert>
    client-key-data: <private key>
```

### MCP Response (what the model sees)

```json
{
  "status": "approved",
  "provider": "teleport",
  "resource": "customer-a-staging",
  "permission": "readonly",
  "expires": "2026-02-22T15:00:00Z",
  "credential_id": "cred-k8s-def456",
  "injected": {
    "kubeconfig": "~/.kube/config",
    "context": "customer-a-staging"
  },
  "note": "kubectl configured for cluster customer-a-staging with readonly access."
}
```

## Default Tier Classifications

### Kubernetes Resources

| Credential Scope | Default Tier |
|---|---|
| cluster-wide read-only role | Tier 1 |
| namespace-scoped read-write role | Tier 2 |
| cluster-wide read-write role | Tier 3 |
| cluster-admin / system:masters | Tier 4 |

### Vault Resources

| Credential Scope | Default Tier |
|---|---|
| read-only access to non-secret paths | Tier 1 |
| read access to secret paths | Tier 2 |
| write access to secret paths | Tier 3 |
| policy or auth method management | Tier 4 |

## Teleport Role Design

Scoped Teleport roles need to exist before credential scoping works. These are prerequisites, defined once per environment.

```yaml
# teleport role: kube-agent-readonly
kind: role
metadata:
  name: kube-agent-readonly
spec:
  allow:
    kubernetes_labels:
      '*': '*'
    kubernetes_resources:
      - kind: '*'
        namespace: '*'
        name: '*'
        verbs: ['get', 'list', 'watch']
    kubernetes_groups:
      - 'view'

# teleport role: kube-agent-namespace-admin
kind: role
metadata:
  name: kube-agent-namespace-admin
spec:
  allow:
    kubernetes_labels:
      '*': '*'
    kubernetes_resources:
      - kind: '*'
        namespace: '{{internal.kubernetes_namespace}}'
        name: '*'
        verbs: ['*']
    kubernetes_groups:
      - 'edit'
```

## Prerequisites

- Teleport cluster with Kubernetes Service configured for target clusters.
- Scoped Teleport roles defined for agent use (see role design above).
- Solo: operator has an active `tsh` session via SSO.
- Organizational: tbot configured with appropriate join tokens, or broker registered as a trusted OIDC issuer with Teleport.
