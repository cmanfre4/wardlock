# AWS Credential Provider

The AWS provider issues short-lived, scoped credentials via AWS Identity Center (SSO) and STS.

## Approach

- AWS Identity Center already supports short-lived credentials.
- The framework requests credentials scoped to specific accounts with specific permission sets.
- Duration can be as short as 15 minutes via STS assume-role.
- Different tasks can use different permission sets (readonly vs. specific write permissions) rather than the operator's full admin access.

## Resource and Permission Vocabulary

- **`resource`**: An AWS account identifier (account ID or alias).
- **`permission`**: An IAM permission set name (e.g., `ReadOnlyAccess`, `S3FullAccess`, `AdministratorAccess`).

## Default Tier Classifications

| Credential Scope | Default Tier |
|---|---|
| single account, read-only permission set | Tier 1 |
| single account, scoped write permission set (e.g., S3-only, Lambda-only) | Tier 2 |
| single account, broad write permission set (e.g., PowerUserAccess) | Tier 3 |
| single account, AdministratorAccess or IAM-write permission set | Tier 4 |

Note that the provider classifies based on the breadth and capability of the credential being issued — it has no concept of environments, account purposes, or organizational structure. The distinction between a sandbox account and a production account is operator context, expressed through operator overrides.

## Backend Identity

| Adoption Phase | Identity Source |
|---|---|
| Solo | Operator's existing AWS SSO session or environment credentials |
| Organizational | OIDC federation via IAM Identity Provider — broker issues JWTs that assume IAM roles via `sts:AssumeRoleWithWebIdentity` |

## Prerequisites

- AWS Identity Center configured with appropriate permission sets.
- Solo: operator has an active AWS SSO session.
- Organizational: IAM Identity Provider configured to trust the broker's JWKS endpoint.
