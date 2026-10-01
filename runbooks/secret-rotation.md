# Secret rotation

Every secret lives in Key Vault `kv-mastrocola-dev` with a 180-day expiry ([ADR-006](../adr/006-identity-and-secrets.md)). Nothing rotates automatically: automating it would need a credential able to mint every other one. Tracking is automatic instead — the weekly [`secret-expiry`](https://github.com/mastrocola-dev/infra/blob/main/.github/workflows/secret-expiry.yml) workflow in `infra` fails when a secret has no expiry or expires within 30 days, and that failure notification is the reminder.

## Inventory

| Secret | Source | Read by | Scope at the source |
|---|---|---|---|
| `cloudflare-api-token` | Cloudflare → My Profile → API Tokens | `id-infra` | Zone → DNS → Edit, zone `mastrocola.dev` only, TTL set |
| `anthropic-api-key-ci` | Anthropic Console, workspace `ci` | `id-docs` | Workspace spend limit |

The Static Web Apps deployment token is not stored; reset it with `az staticwebapp secrets reset-api-key --name stapp-portfolio-www --resource-group rg-portfolio-dev` whenever it may have leaked.

## Rotating

1. Create the new credential at the source, with an expiry when the source supports one
2. Store it:

```bash
read -rs VALUE && az keyvault secret set --vault-name kv-mastrocola-dev --name <secret> --value "$VALUE" \
  --expires "$(date -u -d '+180 days' +%Y-%m-%dT%H:%M:%SZ)" --query attributes.expires -o tsv
```

3. Run the consuming workflow once and confirm it is green (`site` in `infra` via dispatch for Cloudflare; any pull request touching `adr/` for Anthropic)
4. Revoke the previous credential at the source
5. Run `secret-expiry` via dispatch and confirm it is green

Consumers always read the latest version, so no pipeline or repository changes. Writing the value never involves Terraform: bootstrap declares each secret with a write-only placeholder and ignores `expiration_date`.

## Adding a secret

Add it to `secret_readers` in `infra/bootstrap/secrets.tf` with the identity that reads it, apply bootstrap, then store the real value as above and add a row to the inventory.

## Incident log

None yet.
