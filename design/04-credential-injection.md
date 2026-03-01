# Credential Injection

Credential injection is a **side effect of the broker's `request_access` call**, not a separate step the model performs. The MCP response contains only metadata — secrets are materialized in the container's filesystem for CLI tools to consume.

Injection is a collaboration between three components:

1. **Credential provider** — produces a credential bundle declaring what needs to exist in the container (files to write, env vars to set) and what metadata to show the agent. The provider is isolation-agnostic — it has no knowledge of how containers work.
2. **Broker** — orchestrates the flow. Routes the request to a credential provider, then routes the resulting bundle to an isolation provider for materialization, and returns only the metadata portion to the agent via the MCP response. In the single-process deployment, this is direct function calls. As the broker decomposes at scale, the control plane routes jobs to credential workers and isolation workers via a work queue.
3. **Isolation provider** — materializes the bundle in the container. It knows how to write files and set env vars for the specific container technology being used.

This separation keeps credential providers and isolation providers fully decoupled. A GitHub credential provider doesn't need to know whether it's running on localhost or in a remote Kubernetes pod — it produces the same bundle either way. Each isolation provider implements the injection types it supports:

| Injection Type | Localhost | Devcontainer (future) | Kubernetes pod (future) | Remote VM (future) |
|---|---|---|---|---|
| `file` | Write directly to filesystem | Write to bind-mounted shared volume | Create Secret, mount into pod | scp to host |
| `env` | Export in shell profile | Container env config or wrapper script | Pod env spec or ConfigMap | Export in shell profile |

If an isolation provider encounters an injection type it doesn't support, it returns an error. This lets the system add new injection types incrementally without requiring all isolation providers to update simultaneously.

In the single-process deployment, secrets pass through the broker in memory during this handoff. As the broker decomposes at scale, credential workers encrypt bundles before enqueuing them, and isolation workers decrypt after dequeuing — the control plane never touches plaintext bundle contents. In both cases, the broker treats bundle contents as opaque bytes and does not log or inspect them. The secrets ultimately live on the filesystem where the agent runs — accessible to the agent if it reads the files, but not surfaced through the MCP interface.

The broker completes injection before returning the `request_access` MCP response, so by the time the model sees the "kubectl is now configured" metadata, the kubeconfig is already in place.

---

## Open Questions

### Credential Injection Sub-questions

- If multiple credential bundles write to the same file path (e.g., two Kubernetes clusters both targeting `.kube/config`), how are merges handled? Kubeconfig supports multiple contexts natively, but other tools may not. Should the isolation provider handle merging, or should credential providers use distinct file paths?
- For the `env` injection type (Phase 3): how does the isolation provider handle rotation for env vars that can't be updated after container start? Options include wrapper scripts, a sidecar that re-exports, or requiring the container to be restarted.
- **Proxy-based injection type (future):** The framework already operates a CA for mTLS between agent and broker. Could this CA infrastructure be extended to support a TLS-intercepting proxy as an injection type? The proxy would sit between the agent and the network, terminating TLS on the agent side (using dynamically generated certs signed by Wardlock's CA) and establishing new TLS connections to real destinations with real credentials. Two modes of credential injection are possible at the proxy layer:
  - **HTTP-layer injection**: The proxy terminates TLS, injects HTTP headers (e.g., `Authorization: Bearer ...`), and re-encrypts to the destination. Covers GitHub API tokens, SaaS API keys, and similar HTTP-based credentials.
  - **TLS-layer injection (mTLS)**: The proxy holds client certificates and presents them during the TLS handshake with the destination. The agent never sees the client cert or private key. This works for any protocol over TLS — the proxy doesn't need to understand the application layer, it just handles the TLS handshake differently on each side and passes application bytes through unmodified.
  - The proxy needs a routing table: which client cert or HTTP credential to use for which destination, matched on SNI hostname.
  - Revocation is stronger than file injection — removing the proxy rule immediately stops credential injection even if the agent cached the token in memory.
  - Explicit proxy (`HTTPS_PROXY` env var) is likely more pragmatic than transparent proxy (iptables redirect) — simpler, and the tools that don't respect `HTTPS_PROXY` are mostly the ones that need file-based credentials anyway.
  - **What the proxy cannot cover**: non-TLS protocols (SSH key-based auth on port 22) and tools with fully custom connection logic that bypass standard TLS/HTTP stacks.
  - **CA trust injection** adds complexity: the Wardlock CA cert must be added to the container's system trust store, plus language-specific stores (Node.js `NODE_EXTRA_CA_CERTS`, Python `REQUESTS_CA_BUNDLE`, Java keystore) and tool-specific config (`git http.sslCAInfo`). The isolation provider would handle this at container setup.
  - **Open design question**: should the mTLS CA (agent-to-broker auth) and the interception CA (proxy-generated destination certs) be the same CA or separate CAs? Separate CAs limit blast radius — compromising the interception CA only allows MITM of agent traffic, not impersonation of agents to the broker.
  - **Kubernetes API server analysis**: kubectl's connection to the API server supports both client certificate auth (real mTLS at the TLS handshake layer) and bearer token auth (HTTP `Authorization` header inside the TLS tunnel). Both are proxyable. Bearer token injection is the cleaner path — most managed Kubernetes (EKS, GKE, AKS) uses token-based auth, and even kubeadm clusters can issue ServiceAccount tokens. Client cert mTLS proxying also works (the proxy holds the private key and presents the cert to the API server) but identity is baked into the TLS session (CN/O fields), making per-request identity switching impossible. kubectl still needs a kubeconfig on disk to know where to connect, but that kubeconfig is just a non-secret pointer to the proxy address — no credential material in it. Additional proxy complexity for Kubernetes: WebSocket/SPDY upgrades (`kubectl exec`, `kubectl port-forward`) and long-lived watch streams (`kubectl get pods -w`) require the proxy to handle HTTP upgrades and chunked streaming. **Note**: For Teleport-managed clusters, the proxy injection type is unnecessary — Teleport already provides credential isolation via its own proxy and impersonation model (see Kubernetes implementation notes). The Wardlock proxy would be relevant for non-Teleport K8s setups with direct API server access.
  - This would be an additional injection type alongside `file` and `env`, not a replacement. A credential provider's bundle could mix types — e.g., `proxy` for the API token and `file` for a non-secret kubeconfig pointing to the proxy. Worth evaluating once file injection is proven and the mTLS CA infrastructure is mature.
