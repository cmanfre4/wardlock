# GitHub Credential Provider

GitHub is the hardest credential problem due to the breadth of access personal tokens provide. The GitHub provider replaces all personal credentials (admin tokens, SSH keys, signing keys) with a single GitHub App identity.

## Approach

A GitHub App per organization, managed by the operator in solo mode or centrally by the platform team in organizational mode. The App provides:

- **Repo access** — installation tokens scoped to specific repositories with specific permissions (e.g., `contents:write`, `actions:read`), expiring in 1 hour.
- **Git authentication** — the provider configures a git credential helper in the container that serves the scoped token over HTTPS. The agent uses `https://` git URLs, not `ssh://`. The credential helper only responds to URLs matching in-scope repos — out-of-scope repos fail with a 401.
- **Commit signing** — commits attributed to the App's bot identity (`wardlock[bot]`) are automatically verified by GitHub. No GPG or SSH signing keys needed. This satisfies branch protection signing requirements.
- **Discrete service identity** — the App is a bot account that survives employee departures, is auditable separately from any individual, and makes agent-authored commits clearly identifiable in git history.

## Resource and Permission Vocabulary

- **`resource`**: A repository identifier in `org/repo` format. Supports wildcards (e.g., `org/customer-*-infra`).
- **`permission`**: GitHub App permission scopes (e.g., `contents:write`, `actions:read`, `pull_requests:write`). Multiple permissions can be requested in a single call.

## GitHub App Scoping Model

One App is sufficient for all tasks across an org. The App has three layers of scoping:

1. **App manifest** (set once at creation) — declares the maximum set of permissions the App *could ever* request (e.g., `contents: write`, `actions: read`, `pull_requests: write`). This is the ceiling. Set it to the broadest set of permissions any agent task might need.
2. **Installation** (set once per org) — when the App is installed on an org, the org admin approves the declared permissions and chooses which repos the App can access (all repos, or a specific subset). This is also a one-time step.
3. **Token request** (per `request_access` call) — each call to `POST /app/installations/{installation_id}/access_tokens` can narrow to a subset of repos and a subset of permissions from what the installation allows. Each token is independently scoped and independently expires in 1 hour.

This means from a single App installed once on an org, you can generate many concurrent tokens with different scopes:
- Token A: `contents:write` on `org/repo-a` only
- Token B: `actions:read` on `org/repo-b` and `org/repo-c` only
- Token C: `contents:write` + `pull_requests:write` on `org/repo-a` through `org/repo-d`

The App installation is the broker's identity with GitHub.

## Implementation

Uses the GitHub App JWT authentication flow: sign a JWT with the App private key, exchange it for an installation token scoped to specific repos and permissions.

- GitHub App installation tokens expire in 1 hour max — no need for manual revocation in most cases.
- The GitHub REST API endpoint is `POST /app/installations/{installation_id}/access_tokens` with a body specifying `repositories` and `permissions`.
- A custom git credential helper returns the token only for matching HTTPS URLs, ensuring the token only works for in-scope repos. The token exists in the credential helper script on the container's filesystem — git consumes it transparently without the model needing to handle it.
- Git identity is configured as the App's bot account — commits are automatically verified by GitHub.
- Go: `golang-jwt/jwt` for JWT signing, `google/go-github` for GitHub API calls.

## Injection

A single `request_access` call for a GitHub credential configures all of the above as side effects. The provider's injection is richer than a single credential file — it sets up the credential helper, git user identity, and committer email.

### Credential Bundle

```json
{
  "credential_id": "cred-gh-abc123",
  "expires": "2026-02-22T15:30:00Z",
  "injections": [
    {
      "type": "file",
      "path": ".config/git/wlk-credential-helper",
      "content": "IyEvYmluL3NoCiMgY3JlZGVudGlhbCBoZWxwZXIgc2NyaXB0IGNvbnRlbnQu...",
      "mode": "0755"
    },
    {
      "type": "file",
      "path": ".gitconfig.d/wardlock.inc",
      "content": "W2NyZWRlbnRpYWxdCgloZWxwZXIgPSAhfi8uY29uZmlnL2dpdC93bGstY3Jl..."
    }
  ],
  "metadata": {
    "injected": {
      "credential_helper": "configured for https://github.com/org/eks-cluster-module",
      "commit_identity": "wardlock[bot]",
      "committer_email": "12345+wardlock[bot]@users.noreply.github.com"
    },
    "note": "git configured for HTTPS access to org/eks-cluster-module with contents:write. Commits attributed to wardlock[bot] and automatically verified."
  }
}
```

Decoded injection contents:

`.config/git/wlk-credential-helper`:
```bash
#!/bin/sh
# Wardlock git credential helper
# Returns the scoped GitHub App installation token for matching HTTPS URLs
case "$1" in
  get)
    echo "protocol=https"
    echo "host=github.com"
    echo "username=x-access-token"
    echo "password=ghs_xxxxxxxxxxxx"
    ;;
esac
```

`.gitconfig.d/wardlock.inc`:
```ini
[credential]
	helper = !~/.config/git/wlk-credential-helper
[user]
	name = wardlock[bot]
	email = 12345+wardlock[bot]@users.noreply.github.com
```

### MCP Response (what the model sees)

```json
{
  "status": "approved",
  "provider": "github",
  "resource": "org/eks-cluster-module",
  "permission": "contents:write",
  "expires": "2026-02-22T15:30:00Z",
  "credential_id": "cred-gh-abc123",
  "injected": {
    "credential_helper": "configured for https://github.com/org/eks-cluster-module",
    "commit_identity": "wardlock[bot]"
  },
  "note": "git configured for HTTPS access to org/eks-cluster-module with contents:write. Commits attributed to wardlock[bot] and automatically verified."
}
```

## Default Tier Classifications

| Credential Scope | Default Tier |
|---|---|
| single repo, read-only permissions (contents:read, actions:read) | Tier 1 |
| single repo, write permissions (contents:write, pull_requests:write) | Tier 2 |
| multiple repos or wildcard, write permissions | Tier 3 |
| organization-level permissions (members:write, admin:org) | Tier 4 |

## Backend Identity

GitHub doesn't support JWKS verification — the broker authenticates using the App's private key directly (JWT signed with the key, exchanged for installation tokens).

| Adoption Phase | Identity Source |
|---|---|
| Solo | App private key on local filesystem (`source: file`) |
| Organizational | App private key in Vault (`source: vault`) |

## Prerequisites

- Create and install a GitHub App on target organization(s).
- Store the private key according to the adoption phase (local file or Vault).
- Configure the App manifest with the maximum permission set any agent task might need.
- Choose repo access scope during installation (all repos or specific subset).
