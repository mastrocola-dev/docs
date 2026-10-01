# GitHub → Azure OIDC federation

Pipelines authenticate to Azure without stored credentials: the workflow requests an OIDC token from GitHub, and Entra ID exchanges it for an access token of the repository's managed identity if the token's `sub` claim matches one of its federated credentials.

## Subject format

Since 2026-07-15, GitHub emits the **immutable** subject format for repositories that are created, renamed, or transferred — embedding numeric IDs that survive renames:

```
repo:mastrocola-dev@324597838/infra@1355062376:ref:refs/heads/main   (push to main)
repo:mastrocola-dev@324597838/infra@1355062376:pull_request          (pull requests)
```

Older repositories may still emit the legacy name-based format (`repo:owner/repo:...`). Never assume — verify what a repository actually emits:

```bash
gh api repos/mastrocola-dev/<repo>/actions/oidc/customization/sub
```

## Registering a new repository

Each repository authenticates as its own user-assigned managed identity in `rg-identity` ([ADR-006](../adr/006-identity-and-secrets.md)). Everything is declared in `infra/bootstrap/identity.tf` and applied by a human Owner:

1. Add the repository and its numeric id to `github_repository_ids` (`gh api repos/mastrocola-dev/<repo> --jq .id`)
2. Add one entry per subject the workflows present to `github_federations` — usually `ref:refs/heads/main` and `pull_request`
3. Grant the identity only what its workflows need, in bootstrap, scoped as narrowly as the resource allows
4. `terraform apply -parallelism=1` — Azure rejects concurrent federated credential writes on one identity (`409 Conflict`)
5. Set the repository variables `AZURE_CLIENT_ID` (output `ci_client_ids`), `AZURE_TENANT_ID` and, when the identity holds subscription-scoped roles, `AZURE_SUBSCRIPTION_ID`

Workflows request `id-token: write` and log in with `azure/login`; a job with no subscription-scoped role uses `allow-no-subscriptions: true`.

Constraints:

- Matching is exact and case-sensitive
- Limit of 20 federated credentials per identity
- GitHub *environments* change the subject again (`...:environment:<name>`) and need their own credential
- Pull requests from forks receive no OIDC token

## Diagnosing `AADSTS700213`

`No matching federated identity record found for presented assertion subject '<sub>'`

The error message contains the exact subject GitHub sent. Compare it character-by-character against registered credentials:

```bash
az identity federated-credential list --identity-name id-<repo> --resource-group rg-identity -o table
```

Common causes:

| Symptom | Cause |
|---|---|
| Subject contains `@<numbers>` but credential does not | Repo switched to immutable format (transfer, rename, or new repo) — re-register with the presented subject |
| Subject ends `:pull_request`, only `:ref:...` registered | Missing PR credential |
| Subject ends `:environment:<name>` | Workflow uses a GitHub environment — register that subject |
| Names differ in case | Case-sensitive match |

## Incident log

**2026-09-03** — `infra` pipeline failed with `AADSTS700213` after the repository was transferred from a personal account to `mastrocola-dev` and renamed. Transfer + rename triggered GitHub's automatic switch to immutable subjects; the name-based credentials never matched again. Fixed by registering both subjects in the immutable format (copied verbatim from the error message) and deleting the legacy credentials. Zero drift confirmed via `terraform plan`.

**2026-10-01** — Migration from a single app registration to per-repository managed identities (ADR-006). The first bootstrap apply created four of five federated credentials; the fifth failed with `409 Conflict: concurrent requests being made to the tenant` because two credentials on `id-infra` were written in parallel. Nothing was left half-created; a second apply added it. Credential additions now use `-parallelism=1`.
