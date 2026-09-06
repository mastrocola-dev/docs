# Runbook: Static Web Apps — custom domains and async operations

Operational knowledge for managing Azure Static Web Apps through Terraform under this organization's least-privilege model. Everything here was learned the hard way; see the incident log.

## Domain validation patterns

The two validation types behave in opposite directions. Getting the direction wrong produces a failed apply, not a broken site — but it costs a cycle.

**Apex (`dns-txt-token`):** the validation token is born *from* the custom domain resource, so the TXT record comes *after* it — Terraform's implicit dependency (the record references `validation_token`) orders this correctly. The TXT record is **one-shot**: once the domain validates, Azure clears the token, and a later refresh will try to write `null` into the record. Remove the record from code after validation, same convention as consumed import blocks.

**Subdomain (`cname-delegation`):** validation happens *by* the CNAME record existing, so the record must come *first*. The CNAME references the app (not the custom domain), which means Terraform sees no dependency between them and parallelizes — an explicit `depends_on` on the custom domain resource is mandatory. The CNAME is permanent: it is the traffic itself.

Validation is asynchronous on Azure's side in both cases. Apex TXT validation can take from minutes up to hours; CNAME validation of a pre-existing record is near-instant.

## Async operation polling requires subscription-scope reads

Creating a custom domain (and other Static Web Apps mutations) returns an async operation whose status lives at **subscription scope**: `/providers/Microsoft.Web/locations/<region>/staticSitesOperationStatuses/<guid>`. A CI identity holding Contributor only on the workload resource group can start the operation but cannot poll it — the apply fails with 403 `AuthorizationFailed` while the operation itself usually **completes server-side**, leaving an orphan to import or delete.

Fix (in bootstrap): custom role `Web Async Operation Reader` assigned to the CI principal at subscription scope, with a single action:

```
Microsoft.Web/locations/*/read
```

The wildcard is not laziness — it is required. The literal action Azure demands (`staticSitesOperationStatuses/read`) is **absent from the `Microsoft.Web` operations catalog**, and custom role definitions reject actions the catalog does not list (`InvalidActionOrNotAction`). Wildcards are validated as patterns, not expanded against the catalog, so they match the phantom action at authorization time. Verify the gap: `az provider operation show --namespace Microsoft.Web` and search for `staticSitesOperationStatuses` — it is not there.

## Attributes owned by other systems

Two attributes on site resources are deliberately excluded from reconciliation via `lifecycle.ignore_changes`:

- `azurerm_static_web_app`: `repository_url`, `repository_branch` — written by the `static-web-apps-deploy` action on every content deploy from the `www` repo. Without the exclusion, infra applies erase them and content deploys restore them, forever.
- `azurerm_static_web_app_custom_domain`: `validation_type` — not returned by the Azure API, so an imported domain has it empty in state while config declares it. The attribute forces replacement, so a freshly imported domain gets destroyed and recreated on the next apply.

## Recovering an orphaned custom domain

Symptoms: apply fails with "a resource with the ID ... already exists", typically after a 403-interrupted create. Confirm with `az staticwebapp hostname list -n <app> -o table`.

Adopt via one-shot import block — the id composes from values already in state, so the subscription ID never enters the public repo:

```hcl
import {
  to = azurerm_static_web_app_custom_domain.<name>
  id = "${azurerm_static_web_app.<app>.id}/customDomains/<hostname>"
}
```

The `ignore_changes = [validation_type]` exclusion must be in place **before** this import lands, or the adoption immediately becomes a replacement. Remove the block once consumed.

## Incident log

**2026-09-04 — www custom domain creation race (400 `CNAME Record is invalid`).** First apply of the www subdomain created the CNAME and the custom domain in parallel; Azure checked before the record existed. Root cause: no implicit dependency (see patterns above). Fixed with explicit `depends_on`.

**2026-09-04 — 403 on async polling (`AuthorizationFailed` on `staticSitesOperationStatuses/read`).** Custom domain create succeeded server-side while the CI identity could not poll the operation, failing the apply and orphaning the resource. Root causes: least-privilege RBAC boundary vs subscription-scoped operation status endpoints; compounded by the action missing from the provider catalog, which rejected the first custom-role fix (`InvalidActionOrNotAction`). Fixed with the wildcard custom role.

**2026-09-05 — import-then-replace of adopted domain.** An orphaned www domain was imported and immediately destroyed/recreated by the same apply because `validation_type` came back empty from the API. The recreation then hit the (still unfixed) 403, re-orphaning it. Root cause chain: API gap + force-new attribute + merging a PR without reading the plan, which clearly showed `1 to import, 1 to add, 1 to destroy`. Fixed with `ignore_changes`; the plan-reading gate is the process fix.

**2026-09-05 — bootstrap local state lost in repo migration.** The bootstrap's intentionally-local tfstate did not survive a working-directory migration; recovered from the OS recycle bin. Permanent fix: bootstrap state migrated to the remote backend (`key = bootstrap.tfstate`) — the day-zero chicken-and-egg constraint no longer applied. The same episode surfaced weeks-old drift (unapplied tag renames) and a broken `outputs.tf` that no pipeline validates because bootstrap is outside CI — strengthening the case for a root-level fmt/validate job.