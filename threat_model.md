# Wardlock Threat Model

> **Status: Work in progress.** This captures initial thinking and will need to be revisited as the design and implementation mature.

This document examines the trust boundaries and supply chain risks of the Wardlock framework. It is a companion to `wardlock.md` (the design document) and is intended to inform review criteria, testing strategies, and security decisions — not to duplicate the architecture.

## Trust Boundaries

Wardlock's trust boundary is the credential management flow: brokering, injection, and revocation. The framework is responsible for ensuring that credentials are scoped, time-limited, and delivered to the right container. It is **not** responsible for:

- **What runs inside the container** — the agent CLI, CLI tools (`kubectl`, `tsh`, `git`), language runtimes, and any other software the operator installs. These are the operator's choice and responsibility.
- **What the agent does with credentials** — once a credential is in the container's filesystem, the agent (and any tool it invokes) can use it within the credential's scope. Wardlock bounds access via scoping and TTL, not by hiding files.
- **Network security of the container** — network policy is a function of the isolation environment, not something Wardlock manages.

## Supply Chain

### What Wardlock Owns

1. **Broker + core** — TypeScript, distributed via npm. Dependencies include the MCP SDK, crypto/cert libraries for mTLS.
2. **First-party credential providers** (GitHub, Kubernetes) — TypeScript, npm dependencies (`jsonwebtoken`, `octokit`, `tsh` CLI wrapping).
3. **First-party isolation providers** (devcontainer) — TypeScript, wraps the `devcontainer` CLI.

These are under the project's direct control. The primary exposure is the **npm dependency tree** — a compromised transitive dependency in any of these could intercept secrets as they flow through broker or provider code.

### What Wardlock Does Not Own

4. **Third-party credential providers** (future) — written by others, distributed independently. Their own dependency trees, outside the project's control.
5. **Container contents** — agent CLIs, CLI tools, devcontainer features. Upstream projects with their own security posture.
6. **The agent itself** — Claude Code, Codex, or whatever runs inside the container.

Items 5 and 6 are explicitly the operator's responsibility. Item 4 will eventually need a trust model (registry, signing, community vetting) similar to Terraform's provider registry, but this is a far-future concern — all providers are first-party for the foreseeable future.

### Risk Concentration

The most likely attack vector is a **compromised npm dependency** in the broker or first-party providers. This is the classic supply chain risk (e.g., `event-stream` incident). A malicious transitive dependency could intercept credential material as it passes through the broker from credential provider to isolation provider.

A **malicious third-party provider** is a future risk. Both credential providers and isolation providers must handle secret material to do their jobs — a credential provider creates secrets, an isolation provider materializes them in containers. This trust is inherent and cannot be architecturally eliminated. It is the same trust model as Terraform providers.

## Mitigations

### For items 1-3 (what we own)

- Minimize dependencies — fewer transitive deps means less exposure.
- Pin dependency versions and use lockfiles.
- Regular `npm audit` and dependency review.
- The broker treats credential bundle contents as opaque bytes and does not log them — limits accidental exposure through logging.

### For item 4 (third-party providers, future)

- To be defined. Likely involves a provider registry, signing, and community vetting. Not a near-term concern.

## Open Areas

- What specific npm dependencies will be in the tree, and what is their security track record?
- Should the broker enforce any additional isolation between itself and providers (e.g., providers as separate processes with limited IPC)?
- How do we handle the case where a provider's dependency is compromised after initial vetting?
- Should there be a mechanism for operators to vendor or pin provider versions independently of the core framework?
