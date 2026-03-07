# Credential Providers

Each backend system is implemented as a provider plugin with a consistent interface. Providers handle the specifics of how credentials are requested, scoped, issued, and revoked for their respective systems.

## First-Party Providers

| Provider | Description | Spec |
|---|---|---|
| GitHub | GitHub App-based scoped tokens, git credential helper, commit signing | github.md |
| Teleport | Short-lived certs for Kubernetes, Vault, and other Teleport-brokered resources | teleport.md |
| AWS | Short-lived STS/Identity Center credentials scoped to accounts and permission sets | aws.md |

## Extensibility

Providers follow the Terraform provider pattern:

- Independently versioned and distributed
- Consistent interface (request, revoke, audit)
- Schema that describes available resource types, permission levels, and scoping options
- Handle authentication and credential management internally

Third parties can implement providers for their own backends (GCP, GitLab, Azure, etc.) against the same interface.

## Provider Discovery

How the framework discovers and loads providers evolves with adoption phases:

- **Early adoption**: Providers are hardcoded in the broker process. GitHub and Teleport credential logic is built directly into the broker binary — no plugin loading, no discovery. The broker knows what providers exist because they're compiled in.
- **At scale**: Providers are extracted into separate plugin binaries that run on credential workers, not the control plane. Workers load the provider plugins they're configured to run. The control plane routes jobs to the right worker pool by provider name — it doesn't need the provider binary itself.

Provider configuration follows the Terraform model: a configuration file declares which providers are available and their connection details (identity sources, backend endpoints). In early adoption this is the broker's own config. At scale the control plane owns provider configuration centrally, and worker pools are deployed with the corresponding plugin binaries. Configuration is versioned alongside the broker's deployment — in organizational mode, it lives in the control plane's config store and applies uniformly across all operators.

## Provider Contract

Each provider implements a consistent interface that the broker calls to manage credentials:

```go
type CredentialProvider interface {
    // Schema returns the provider's supported resource types, permission
    // levels, and additional match dimensions for overrides.
    Schema() ProviderSchema

    // Classify returns the provider's default tier classification for a
    // given resource and permission combination.
    Classify(resource, permission string) Tier

    // Issue creates a scoped, time-limited credential and returns a bundle
    // describing what needs to be injected into the container and what
    // metadata to show the agent.
    Issue(ctx context.Context, resource, permission string, duration time.Duration) (*CredentialBundle, error)

    // Revoke invalidates the credential at the backend if possible (e.g.,
    // revokes a GitHub token via API). Returns whether backend-side
    // revocation succeeded. Some backends don't support explicit revocation
    // and rely on TTL expiry instead.
    Revoke(ctx context.Context, credentialID string) (bool, error)

    // Validate checks whether the requested resource and permission are
    // valid and the provider can fulfill them.
    Validate(resource, permission string) error
}
```

There is no `Renew()` method. Credential renewal flows through the same `request_access` → `Issue()` path as the original request, subject to the same approval tier evaluation. The broker revokes the expiring credential and issues a fresh one. This keeps the provider interface simple and ensures renewal doesn't bypass approval controls.

Credential providers are **isolation-agnostic**. They produce declarative bundles describing what needs to exist in the container — they do not write files or configure tools directly. The broker hands the bundle to the isolation provider, which knows how to materialize it in the container (see Credential Injection).

### Credential Bundle

The `issue()` method returns a credential bundle: a declarative description of what needs to exist in the agent's container, plus metadata for the agent's context. The bundle contains secret material (file contents, tokens). The broker passes the bundle to the isolation provider for injection and returns only the metadata portion to the agent via the MCP response. The secrets end up in the container's filesystem where CLI tools consume them — the agent could read these files directly, but the credential management flow does not put them into the conversation.

A bundle consists of typed **injection entries**, each declaring an injection type that the isolation provider must implement. This keeps credential providers isolation-agnostic — they declare what they need, and the isolation provider decides how to materialize each type for its container technology.

```go
// InjectionEntry describes a single item to materialize in the container.
// Each entry declares a type and type-specific parameters.
type FileInjection struct {
    Path    string // relative to container home
    Content []byte // raw content (base64 encoding is a serialization concern)
    Mode    string // e.g. "0755", optional
}

type EnvInjection struct {
    Name  string
    Value string
}

type InjectionEntry struct {
    Type string // "file" or "env"
    File *FileInjection
    Env  *EnvInjection
}

// CredentialBundle is the output of a credential provider's Issue call.
type CredentialBundle struct {
    // Unique identifier for this credential (used for revoke, monitoring)
    CredentialID string

    // When the credential expires
    Expires time.Time

    // Typed injection entries — each describes something to materialize
    // in the container. Isolation providers implement the mechanics for
    // each type.
    Injections []InjectionEntry

    // Metadata for the agent's context — provider-defined, describes what
    // was configured in human-readable terms. This is the ONLY part of the
    // bundle that reaches the model via the MCP response.
    Metadata BundleMetadata
}

type BundleMetadata struct {
    Injected map[string]string
    Note     string
}
```

**Injection types:**

| Type | Description |
|---|---|
| `file` | Write a file at `path` (relative to container home) with base64-encoded `content`. Optional `mode` (e.g., `"0755"` for executables). |
| `env` | Set environment variable `name` to `value`. |

New injection types can be added as needed (e.g., `socket`, `mount`). If an isolation provider encounters a type it doesn't support, it returns an error — the broker reports this as an injection failure. This makes the system extensible without requiring all isolation providers to support every type from day one.

The credential provider produces the bundle. The broker splits it: `injections` go to the isolation provider for materialization, `metadata` goes to the agent via the MCP response. The credential management flow does not surface secret material into the model's context — though the agent has filesystem access to the injected files.

The `metadata.injected` field is intentionally a flat key-value map rather than a structured type — each provider defines what keys are meaningful for its domain. The broker doesn't interpret these; it passes them through to the agent.

For example bundles, see the individual provider specs: GitHub, Teleport.

Credential artifacts (files, env vars) injected into the container are not cleaned up on revocation or expiry. The credential is invalidated at the backend — the files on disk become stale and harmless. When the container exits at task end, everything is destroyed. If the agent hits an expired credential mid-task, it calls `request_access` again for a fresh one.

---

## Open Questions

### Provider Contract Sub-questions

- How does the provider report errors (invalid resource, insufficient broker permissions, rate limits)?
- Should the `metadata.injected` map support structured values (e.g., a list of configured repos) or stay strictly flat?
