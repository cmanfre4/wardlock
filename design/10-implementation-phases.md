# Implementation Phases

The implementation is structured as a series of phases, each building on the last. Every phase produces something immediately usable for safer agent operation, and every line of code moves toward the full framework.

---

## Phase 1: GitHub Credential Provider

**Goal**: Build the first credential provider as a standalone Go package and validate the GitHub App credential flow end-to-end. This establishes the concrete types (credential bundles, injection entries) and patterns that all later providers and the broker build on. A thin CLI harness exercises the provider — the provider package is the deliverable, the harness is scaffolding.

**What gets built**:
- A Go module with a `github` provider package implementing the `CredentialProvider` interface: JWT signing, GitHub API token exchange, credential bundle construction, git credential helper generation, commit identity setup. See GitHub provider spec for implementation details.
- A CLI harness (`cmd/wlk`) that calls the provider through the `CredentialProvider` interface and prints the bundle metadata to stdout. No file writes, no injection, no isolation provider — just validating that the provider produces a correct bundle.
- No isolation, no tiers, no manifests.

**Prerequisites (manual, one-time)**:
- Create and install a GitHub App on target organization(s). See GitHub prerequisites.

**Usage**:

```bash
wlk github \
  --app-id 12345 \
  --app-key ~/.config/wardlock/github-app-key.pem \
  --org my-org \
  --repos "eks-cluster-module,customer-a-infra,customer-b-infra" \
  --permissions "contents:write,actions:read"

# Output: bundle metadata printed to stdout, no secrets displayed
# Provider: github
# Scoped to: my-org/eks-cluster-module, my-org/customer-a-infra, my-org/customer-b-infra
# Permissions: contents:write, actions:read
# Commit identity: wardlock[bot]
# Expires: 2026-02-22T15:30:00Z (1 hour)
# Injection entries: 2 files
```

**What this validates**:
- Does the GitHub App token flow work end-to-end (JWT → installation token → credential bundle)?
- Does the `CredentialProvider` interface feel right with one real implementation behind it?
- Do the `CredentialBundle` and `InjectionEntry` types carry the right information for the GitHub provider's needs?
- Does the generated credential helper script correctly scope access to only in-scope repos?
- What patterns emerge from a real provider implementation that should inform the second provider?

**Order of work**:
1. GitHub App setup (manual, one-time)
2. Go module setup, shared types (`CredentialBundle`, `InjectionEntry`, `CredentialProvider` interface)
3. GitHub provider package: JWT signing, token exchange, credential bundle construction
4. Git credential helper generation (scoped to in-scope repos only)
5. CLI harness: calls provider through `CredentialProvider` interface, prints bundle metadata to stdout

---

## Phase 2: Localhost Isolation Provider

**Goal**: Build the first isolation provider and validate the credential bundle handoff — the localhost provider consumes the bundles that the GitHub provider produces, materializing them on disk. This is where the two halves of the injection model meet for the first time. No broker yet — the CLI harness wires them together directly.

**What gets built**:
- The `IsolationProvider` interface and `EnvironmentHandle` type.
- A `localhost` isolation provider package implementing `IsolationProvider`: takes a credential bundle and writes the injection entries to the operator's filesystem. No containers, no actual isolation — it implements the materialization side of the injection lifecycle.
- The CLI harness is updated to call `Setup` → `Inject` → `Teardown` through the `IsolationProvider` interface. The harness becomes: GitHub provider produces a bundle → localhost provider materializes it → harness prints the metadata.

**What changes from Phase 1**:
- In Phase 1, the harness only prints bundle metadata. Now the localhost provider writes credential files to disk — this is the first time injection actually happens.
- The credential bundle becomes a real integration boundary between two independent packages rather than output that gets printed and discarded.

**What this validates**:
- Does the `CredentialBundle` type work as a clean handoff between a credential provider and an isolation provider, or does the isolation provider need information the bundle doesn't carry?
- Does the `InjectionEntry` type system (`file`, `env`) cover what the GitHub provider actually needs, or are there gaps?
- What does error handling look like at the boundary — what happens when a file write fails, when a path is invalid, when permissions are wrong?
- Do the two packages compose cleanly through the bundle type alone, or is there implicit coupling?
- What would a second isolation provider (devcontainer) need from this same bundle that localhost doesn't care about?

**Order of work**:
1. `IsolationProvider` interface and `EnvironmentHandle` type
2. Localhost isolation provider package: iterate injection entries, write files, set modes, export env vars
3. Update CLI harness to wire `CredentialProvider` → `IsolationProvider` through the bundle
4. Integration test: GitHub provider → localhost provider → git push with scoped credentials
5. Evaluate the bundle types — do they need adjustment now that both sides of the handoff exist?
