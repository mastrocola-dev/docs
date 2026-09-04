# GitHub → Azure OIDC federation

Pipelines authenticate to Azure without stored credentials: the workflow requests an OIDC token from GitHub, and Entra ID exchanges it for Azure access if the token's `sub` claim matches a registered federated credential.

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

## Registering credentials for a new repository

One credential per subject the workflows will present. A repo with plan-on-PR and apply-on-main needs two:

```bash
APP_ID=$(az ad app list --display-name "<app-name>" --query "[0].id" -o tsv)

az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "<org>-<repo>-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "<subject for refs/heads/main>",
  "audiences": ["api://AzureADTokenExchange"]
}'

az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "<org>-<repo>-pr",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "<subject for pull_request>",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

Constraints:

- Matching is exact and case-sensitive
- Limit of 20 federated credentials per application — delete dead ones
- GitHub *environments* change the subject again (`...:environment:<name>`) and need their own credential

## Diagnosing `AADSTS700213`

`No matching federated identity record found for presented assertion subject '<sub>'`

The error message contains the exact subject GitHub sent. Compare it character-by-character against registered credentials:

```bash
az ad app federated-credential list --id $APP_ID -o table
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