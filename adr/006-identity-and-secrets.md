# ADR-006: One workload identity per repository and Key Vault as the only secret store

**Status:** Accepted
**Date:** 2026-09-30

## Context

Three static secrets live in GitHub: the Static Web Apps deployment token (`www`), the Cloudflare API token (`infra`) and the Anthropic API key (`docs`), accepted by ADR-002 and ADR-005 with the expectation that they move to Key Vault. The agent runtime adds at least two more: a separate Anthropic key and the Turnstile secret. A single Entra app registration federates `infra` and holds Contributor on `rg-portfolio-dev`.

Moving `www` and `docs` to OIDC under that same identity would hand a documentation pull request Contributor on the whole resource group — a larger blast radius than the secrets it replaces. Identity must be split before secrets can move.

Options considered for who grants Azure roles:

- **CI with constrained role-assignment delegation** — roles and principals limited by an ABAC condition. Rejected: one flaw in the condition lets CI grant itself read access to every secret.
- **Bootstrap, applied only by a human Owner** — the boundary already in place for every elevated change.

## Decision

- **One user-assigned managed identity per repository**, federated with GitHub OIDC (`azurerm_federated_identity_credential`, immutable subject format). No app registrations: identities are ARM resources and need no Entra directory role. The existing `infra` app registration migrates to this model and is deleted.
- **Least privilege per identity:**

| Identity | Grants |
|---|---|
| `id-infra` | Contributor on `rg-portfolio-dev`; read of the Cloudflare secret; Key Vault metadata read |
| `id-www` | Custom role limited to `listSecrets` on the Static Web App |
| `id-docs` | Read of the CI Anthropic secret |

- **Bootstrap owns identities, Key Vault and every role assignment**, including the runtime identities of later services; workload modules reference them as data sources.
- **Key Vault:** a single vault, RBAC authorization, role assignments scoped to individual secrets, soft delete 7 days, no purge protection, public network access (runners and serverless hosts have no fixed egress). Values are written with `az keyvault secret set` and never enter Terraform state.
- **The Static Web Apps token is no longer stored:** `www` fetches it at deploy time with `az staticwebapp secrets list`.
- **Identifiers are not secrets:** client, tenant and subscription IDs move to repository variables. GitHub ends with no secrets.
- **Anthropic workspaces per environment** — `dev` (local `.env`), `ci`, `runtime` — each with its own key and spend limit. No CI or runtime key exists on a workstation.
- **Rotation is manual, tracking is not:** every secret carries a 180-day expiry (Cloudflare tokens also expire at the source). A weekly `secret-expiry` workflow in `infra` reads metadata only and fails when a secret expires within 30 days, which notifies the owner. Rotation steps live in a runbook. Automating rotation was rejected: it requires a credential able to mint the others, more valuable than everything it rotates.

## Consequences

Positive:

- Zero secrets in GitHub; every pipeline authenticates through OIDC, as ADR-002 originally intended
- A compromised workflow reaches only its own repository's grants
- Revoking a pipeline is deleting one federated credential; rotating a secret touches only Key Vault
- The same identity model serves CI and the agent runtime

Negative, accepted:

- Key Vault becomes a dependency of every pipeline
- Every new permission requires a human bootstrap apply
- `id-www` can still read the deployment token, so its power is unchanged; the gain is that no long-lived copy exists outside Azure, and resetting the token needs no GitHub change
- Rotation remains a manual task, about ten times a year
- This ADR supersedes the secret placement in ADR-002 and ADR-005; their decisions otherwise stand
