# Adoption Phases and Scaling

The [implementation phases](10-implementation-phases.md) describe what gets built. The adoption phases below describe how the deployment model evolves as usage scales from a single practitioner to an organization. Each adoption phase introduces architectural requirements that the implementation must eventually support.

---

## Adoption Phase 1: Solo Practitioner, Local Broker

The operator runs the broker on their laptop. Agents run locally (localhost isolation provider). The CLI communicates with the broker over HTTP on localhost — no auth, no TLS, but the API contract is real from day one. This means the operator-facing API and the agent-facing MCP interface are separate surfaces from the start: agents talk MCP via Unix socket, the CLI talks HTTP to the broker's API.

**Architecture:** Single process. Broker + MCP server + credential providers all in one. The trust boundary is the operator's machine. Approval prompts appear in the operator's terminal.

**[Broker identity](05-broker-identity.md)** is implicit — derived from the operator's existing sessions. No new infrastructure required. The operator should be able to point the framework at what they already have and be running within minutes:

- The GitHub provider uses a GitHub App the operator installs on their own orgs. The App's private key lives on the operator's machine.
- The Kubernetes/Teleport provider uses the operator's existing `tsh` session (from their normal SSO login) to request scoped cluster certificates. The operator already has access — the framework just requests narrower roles.
- The Vault provider uses the operator's existing Teleport-brokered Vault session to generate scoped tokens.
- The localhost isolation provider runs as the operator's OS user — no auth needed.

The broker's identity is inherently tied to the operator's own identity and permissions, which is acceptable for solo use.

**Why the API matters early:** The CLI-to-broker API exists and is tested before auth, TLS, durable state, or any hosted infrastructure enters the picture. Same rationale as the localhost isolation provider — get the interface working before the real implementation.

**Self-documenting API:** The broker API should be self-documenting from day one. An OpenAPI spec generated from the actual route definitions (not hand-maintained separately) ensures docs and implementation can't drift. The broker serves interactive API documentation (Swagger UI or equivalent) at a well-known path — hit `/docs` on a running broker and you see the full API with try-it-out capability. This is important for adoption at every phase: solo practitioners get immediate discoverability, and platform engineering teams integrating in Adoption Phase 4 can discover the API from the running broker rather than reading external documentation that may be stale.

---

## Adoption Phase 2: Solo Practitioner, Remote Broker

The operator moves the broker to cloud infrastructure (a VM, a container, a small K8s deployment). Agents run in remote isolation providers alongside it. The operator's laptop is no longer in the critical path — tasks survive laptop sleep/shutdown.

**This is the first real architectural inflection.** Everything that was trivially local now needs to work over the network:

- **Auth between CLI and broker.** In solo mode, this could be a simple pre-shared token or mTLS cert. Nothing fancy — it's one person.
- **Broker identity moves out of the laptop.** The GitHub App key, Teleport join token, etc. now live wherever the broker runs — a secrets manager or encrypted config. The isolation provider backend also needs explicit auth (Docker API TLS instead of a local socket). Same operator credentials, just stored remotely instead of locally.
- **Approval flow changes.** The operator can't get a terminal prompt on a remote broker. Approvals need to happen through the CLI (polling or push), a web UI, or notifications.
- **Durable state.** The broker needs to survive restarts. Active credentials, audit log, approval queue need persistent storage. In Adoption Phase 1 these can be in-memory.
- **TLS.** The CLI-to-broker API is now over the internet.

The key advantage of having the API from Phase 1: you're adding auth and infrastructure to an API that already works, not inventing the API at the same time.

---

## Adoption Phase 3: Multi-Practitioner, OIDC Auth

Multiple people in the org use the CLI to launch agents and request credentials. The broker integrates with organizational OIDC (Okta, Azure AD, Google Workspace, etc.).

**Architecture changes:**

- **CLI authenticates via OIDC.** Device flow or browser redirect. The CLI gets an ID token, presents it to the broker. The broker validates against the org's IdP.
- **Broker identity moves from personal sessions to organizational service identities.** The GitHub provider uses a centrally managed GitHub App installed across the organization's repos, with the private key in a secret manager. The Kubernetes/Teleport provider uses a Teleport machine identity (bot) with roles designed for credential delegation — capable of requesting scoped certificates for agents, but not capable of direct cluster access itself. The Vault provider uses a dedicated AppRole or machine identity. Isolation provider backends use dedicated service accounts (K8s service account for pod management, etc.).
- **Authorization becomes real.** Who can launch what tasks? Who can approve what tier of credential? The broker needs a role/policy model mapping OIDC claims (groups, roles) to Wardlock permissions. Operator override policies are defined centrally and apply to all operators.
- **Audit attribution matters.** Every credential request is tied to an authenticated identity. The audit log goes from "the operator did X" to "alice@company.com's agent requested X."
- **Credential scoping may be identity-dependent.** Alice can request GitHub tokens for repos she has access to. Bob can't request tokens for Alice's repos. The broker may need to map OIDC identity to allowed credential scopes.
- **The broker becomes a shared service.** Needs proper uptime, monitoring, possibly HA. Audit logs go to a centralized logging system rather than local files.

---

## Adoption Phase 4: Organizational Orchestration

The org's own automation systems (CI/CD pipelines, internal platforms, custom orchestrators) integrate with the broker directly. The broker is clustered for availability and throughput.

**Architecture changes:**

- **The broker API becomes a first-class integration surface.** Service accounts authenticate via OIDC client credentials flow or mTLS. API stability and versioning matter.
- **Clustering.** Multiple broker instances sharing state. Leader election or shared database for credential lifecycle, approval queues, audit log.
- **Delegated approval.** Orchestration systems pre-approve certain credential patterns via policy rather than requiring human approval. The [tier system](06-tiered-approval.md) becomes policy-driven: "CI pipelines from this service account auto-approve Tier 1-2, escalate Tier 3+."
- **Multi-tenancy.** Different teams, different credential scopes, different policies. The broker needs namespace isolation or team-scoped configuration.
- **Provider scaling.** Many concurrent credential operations. Providers may need connection pooling, rate limiting, circuit breakers for backend system API limits.

---

## Broker Decomposition at Scale

As load increases, the single broker process decomposes into component parts. The decomposition follows the principle: **the control plane makes decisions, workers execute them.**

**The broker's full responsibilities:**

1. MCP server (agent-facing)
2. Operator API (CLI-facing)
3. JWKS endpoint (backend-facing)
4. Isolation job routing (launch, inject, teardown — executed by isolation workers)
5. Credential job routing (issue, revoke — executed by credential workers)
6. Approval workflow (tier evaluation, policy, human approval queue)
7. Credential lifecycle (TTL tracking, expiry, revocation)
8. Certificate authority (mTLS certs for agents)
9. [Audit logging](09-audit-logging.md)
10. API documentation serving

**Natural service boundaries:**

- **Control plane** — the decision-making core. API, approval workflow, policy evaluation, audit, CA, JWKS endpoint. Stateful, owns the database. Holds the JWT signing key. **Provider-agnostic** — the control plane does not load or run provider plugins, does not need provider dependencies, and does not need network access to backend systems. It knows "this job needs the `github` credential provider" and routes accordingly. This is the trust anchor.
- **MCP gateway** — stateless proxy between agents and the control plane. Routes `request_access` calls, streams responses. Scales horizontally.
- **Credential workers** — load and run **[credential provider](03-credential-providers.md) plugins** (the Go binaries that know how to talk to GitHub, Teleport, Vault). Stateless, pull jobs from a queue, return results. Scale independently per provider. Workers can be general (all provider plugins installed) or specialized (dedicated worker pools per provider).
- **Isolation workers** — load and run **[isolation provider](01-isolation.md) plugins** (the binaries that know how to write files into containers, manage Docker, manage K8s pods). Also stateless and queue-driven. Scale independently per isolation backend.

**Worker credential flow — workers never hold long-lived secrets:**

1. Control plane approves a credential request and enqueues a **credential job** (job ID + metadata, no secrets) for a credential worker
2. Credential worker picks up the job, authenticates to the control plane (workers have their own mTLS cert or long-lived identity)
3. Credential worker requests credentials for the job: "I picked up job X, give me what I need"
4. Control plane verifies the worker's identity, verifies the job is legitimately assigned, and issues a short-lived scoped JWT for the specific backend operation
5. Credential worker presents the JWT to the backend (for backends supporting JWKS verification) or receives a short-lived token the control plane obtained from the backend (for backends like GitHub that don't support JWKS)
6. Credential worker receives the credential bundle from the backend
7. Credential worker **encrypts the bundle** using the EncryptionProvider and enqueues an **injection job** with the encrypted bundle attached
8. Isolation worker picks up the injection job, **decrypts the bundle** using the EncryptionProvider, and gets its own scoped JWT for the isolation backend
9. Isolation worker injects the bundle into the target environment, reports success

**Encrypted bundles in transit.** The credential bundle contains secret material (tokens, keys, certs). Plaintext bundles are never stored in queues, tables, or the control plane's state store. The credential worker encrypts the bundle immediately after receiving it from the backend, and the isolation worker decrypts it immediately before injection. The encrypted ciphertext is opaque — useless without access to the decryption key. This means the encrypted bundle can safely sit in a queue or table between the two workers.

**Separation of duties.** The credential worker can encrypt but not decrypt. The isolation worker can decrypt but not encrypt. This is enforced by the encryption backend's access policies (IAM policies for KMS, Vault policies for Transit). The control plane orchestrates the flow but never touches plaintext bundle contents. Every encrypt and decrypt operation is audited by the encryption backend (CloudTrail for KMS, Vault audit log for Transit).

**Revocation bundles** are stored separately in the control plane's state store. These contain only cleanup instructions (file paths to delete, env var names to unset) — no secret material — and are needed later when credentials expire or are revoked.

Spinning up more workers requires no secret distribution — they authenticate to the control plane and get scoped JWTs per job.

**Pluggable infrastructure backends:**

The control plane depends on three infrastructure interfaces:

- **StateStore**: `get`, `put`, `query`, `delete` — holds credential metadata, approval queue, audit log, task state, revocation bundles. No secret material.
- **WorkQueue**: `enqueue`, `dequeue`, `ack`, `nack` — distributes jobs to workers. Encrypted bundles may be attached to injection jobs.
- **EncryptionProvider**: `encrypt(plaintext) → ciphertext`, `decrypt(ciphertext) → plaintext` — protects credential bundles in transit between workers.

| Adoption Phase | StateStore | WorkQueue | EncryptionProvider |
|---|---|---|---|
| Phase 1 (local) | `InMemoryStateStore` (or SQLite) | `InMemoryWorkQueue` (async function calls) | `LocalKeyEncryptionProvider` (RSA key pair on disk) |
| Phase 2+ (AWS) | `DynamoDBStateStore` | `SQSWorkQueue` | `KMSEncryptionProvider` |
| Phase 2+ (non-AWS) | Equivalent managed DB | Equivalent managed queue | `VaultTransitEncryptionProvider` |

The in-memory implementations should behave like the real ones — async returns (promises, not synchronous), serialize/deserialize data (catch accidental object reference sharing). This prevents surprises when swapping to real backends.

In Phase 1, the "workers" are just async functions in the same process. The queue is an event emitter. The state store is a Map. The encryption provider uses a local RSA key pair. The decomposition exists at the interface level, not the deployment level — same pattern as the localhost isolation provider.

**Implications for provider plugins:** Provider plugins run on workers, not the control plane. This means provider updates don't touch the control plane — ship a new version of the GitHub provider binary and only the credential workers need to be updated. It also clarifies the provider plugin protocol: the protocol is between a worker process and the provider plugin it loads, not between the control plane and a remote plugin over gRPC. This could be as simple as a Go function interface compiled into the worker binary, or a subprocess protocol if plugin isolation is desired.

---

## Key Transitions

The biggest architectural jumps:

1. **Phase 1 → 2**: Local to remote. Forces real auth, durable state, TLS, and rethinks the approval UX. Hardest jump because it touches everything.
2. **Phase 3 → 4**: Multi-user to org orchestration forces clustering, API stability, and multi-tenancy.

Phases 1-2 are solo practitioner — buildable and validatable without organizational buy-in. Phase 3 is the "convince your team" phase. Phase 4 is the "convince platform engineering" phase.

**What stays the same across all phases:** The agent doesn't know or care whether the broker is a local process or a clustered service. It makes the same `request_access` MCP calls and gets the same metadata responses. [Task manifests](07-task-manifest.md), [provider configurations](03-credential-providers.md), and the [tier system](06-tiered-approval.md) are stable across transitions — what changes is the broker's identity sources, deployment model, and auth layer.
