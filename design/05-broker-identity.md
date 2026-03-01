# Broker Identity

The broker needs its own identity, permissions, and authentication mechanism for every backend system it interacts with — both credential provider backends (to issue and revoke scoped credentials) and isolation provider backends (to materialize files, manage containers, start/stop environments).

**Credential provider backends:**

| Backend | Identity | Permissions | Auth mechanism |
|---|---|---|---|
| GitHub API | GitHub App | Create installation access tokens scoped to repos/permissions | JWT signed with App private key |
| Teleport Auth Service | Operator's SSO session (solo) or tbot machine identity (org) | Request certificates with specific Teleport roles | Teleport cert/session |
| Vault | Operator's session (solo) or dedicated AppRole (org) | Generate scoped tokens with specific policies | Vault token or AppRole |

**Isolation provider backends:**

| Backend | Identity | Permissions | Auth mechanism |
|---|---|---|---|
| Localhost | The operator's OS user | Write files to filesystem | Implicit (same process) |
| Docker daemon | The operator's Docker socket (solo) or service account (org) | Create/exec/remove containers, bind mount volumes | Docker socket or Docker API TLS |
| Kubernetes API | Service account or kubeconfig | Create pods, mount secrets, exec into containers | ServiceAccount token or kubeconfig |
| Remote VM provider | Cloud provider identity | Create/manage VMs, SSH access | Cloud provider SDK auth |

The broker's identity for each backend should follow least privilege — it should be able to *delegate* scoped credentials but not necessarily have direct access itself. A Teleport bot that can request `kube-agent-readonly` certificates but can't itself kubectl into the cluster.

**How isolation backend auth evolves with adoption phases:**

- **Phase 1 (local)**: Localhost only — implicit, same process, no auth. The broker writes files as the operator's OS user.
- **Phase 2 (remote)**: The isolation provider moves to Docker or a VM. The broker authenticates to the Docker daemon via Docker API TLS (client cert) or to the VM provider via cloud SDK credentials. These are stored wherever the broker runs (secrets manager, encrypted config).
- **Phase 3-4 (org)**: The isolation provider is Kubernetes. The broker uses a dedicated ServiceAccount (for pod management, not workload access) or broker-signed JWTs if the K8s cluster trusts the broker as an OIDC issuer. Isolation workers authenticate to the control plane via mTLS and receive scoped JWTs per job.

## Broker as Identity Issuer (JWT/OIDC)

For backends that support it, the broker can authenticate by issuing its own short-lived, scoped JWTs signed with a trusted private key. Backends verify these JWTs against the broker's JWKS endpoint. The broker becomes its own identity issuer — its own IdP, essentially.

**Backends with native JWT/OIDC trust:**

| Backend | How it works |
|---|---|
| Teleport | Register the broker as a trusted OIDC issuer. Teleport accepts broker-signed JWTs to issue scoped certificates. Replaces the tbot join token model. |
| Kubernetes | Native OIDC token authentication (`--oidc-issuer-url`). The broker issues JWTs that K8s clusters accept directly. |
| Vault | JWT/OIDC auth method. The broker signs a JWT, Vault verifies against the broker's JWKS, issues a scoped Vault token. |
| AWS | OIDC federation via IAM Identity Provider. The broker issues JWTs that assume IAM roles via `sts:AssumeRoleWithWebIdentity`. |

**Backends that don't support JWKS verification** (like the GitHub API, which has its own App JWT flow) fall back to the existing per-backend auth mechanisms.

This scales naturally with adoption:

- **Phase 1**: Broker signs JWTs with a local key. JWKS endpoint on localhost. Backends configured to trust it locally.
- **Phase 2**: Same key, but the JWKS endpoint is now reachable over the network. Backends trust the broker's remote JWKS URL.
- **Phase 3-4**: Signing key managed properly (HSM, KMS, rotated). JWKS endpoint is HA. Backends across the org trust it.

The broker's JWT signing key and its mTLS CA are two facets of the same identity infrastructure — the broker is a trust anchor for both inbound connections (agent-to-broker mTLS) and outbound connections (broker-to-backend JWT).

### Key and Identity Rotation

The broker's signing key and per-backend identities each have a rotation path tied to the adoption phase:

- **JWT signing key**: In Phase 1-2, this is a local RSA key pair — rotation is manual (generate new key, update JWKS, re-trust on backends). In Phase 3-4, the signing key lives in an HSM or KMS, and rotation uses the key management system's native rotation (AWS KMS automatic key rotation, Vault Transit key versioning). The JWKS endpoint naturally serves the current and previous key versions during rotation windows.
- **mTLS CA key**: Same progression. Local CA in Phase 1 → managed CA (Teleport CA, internal PKI) in Phase 3-4. Agent client certs are already short-lived and task-scoped, so CA rotation only requires that new certs are signed by the new CA — existing short-lived certs expire naturally.
- **Per-backend identities**: These rotate through their native mechanisms. The GitHub App private key can be stored in Vault (`source: vault`) and rotated via Vault's secret rotation. Teleport bot identities (`tbot`) continuously renew their own certs. AWS roles assumed via OIDC federation have no static secret to rotate — the trust relationship is the JWKS URL, not a key the broker holds.

The key insight: as the broker moves from local keys to managed key infrastructure (KMS, HSM, Vault), rotation becomes the key management system's responsibility rather than the broker's. The broker's job is to use the key, not to manage its lifecycle.

## Pluggable Identity Sources

How the broker authenticates and whose identity it uses changes as adoption scales (see Adoption Phases). The provider interface must not assume any particular mode — a provider receives an identity configuration (which could be "use this local session", "use this service identity endpoint", or "sign a JWT with this key") and uses it. The provider doesn't know or care whether the identity came from a personal `tsh` login, a Teleport bot, or a broker-signed JWT.

The provider configuration supports pluggable identity sources:

```yaml
credential_providers:
  kubernetes:
    identity:
      # Solo: use operator's existing tsh session
      source: local_tsh_session

  github:
    identity:
      # Solo: App key on local filesystem
      source: file
      path: ~/.config/wardlock/github-app-key.pem
      app_id: 12345

isolation_providers:
  localhost:
    identity:
      # Implicit — same process, no auth needed
      source: local
```

```yaml
credential_providers:
  kubernetes:
    identity:
      # Org: broker-signed JWT verified against JWKS
      source: broker_jwt
      audience: https://teleport.example.com

  github:
    identity:
      # GitHub doesn't support JWKS — use App key from Vault
      source: vault
      vault_path: secret/data/github-app/private-key
      app_id: 12345

isolation_providers:
  kubernetes:
    identity:
      # Org: broker-signed JWT for K8s OIDC auth
      source: broker_jwt
      audience: https://kubernetes.default.svc
```

**Agent-to-broker authentication:** The framework uses mTLS to authenticate agents to the broker. When launching an isolated environment, the framework generates a short-lived client certificate scoped to the task (task ID encoded in the subject/SAN) and injects it alongside the MCP server configuration. The broker validates this certificate on every request, binding each `request_access` call to a specific approved manifest. In solo mode the framework acts as its own CA locally; in organizational mode the CA could be Teleport or another internal PKI. The client certificate is not a credential the model sees or manages — it is part of the environment's infrastructure, like the MCP socket itself.

---

## Open Questions

### Broker Identity Sub-questions

- In solo mode, what happens when the operator's session expires (e.g., Okta SSO session timeout)? Does the broker detect this and prompt the operator to re-authenticate?
- In organizational mode, how are approval prompts routed to the right operator? A web UI? Slack notifications? Integration with existing on-call or approval workflows?
- How does the initial setup work for each mode — a setup wizard, manual configuration file, or integration with existing tooling (`tsh status`, `gh auth status`)?
