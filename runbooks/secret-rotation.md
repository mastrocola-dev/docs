# Secret rotation

Every secret lives in Key Vault `kv-mastrocola-dev` with an expiry, 180 days unless the source says otherwise ([ADR-006](../adr/006-identity-and-secrets.md)). Nothing rotates automatically: automating it would need a credential able to mint every other one. Tracking is automatic instead — the weekly [`secret-expiry`](https://github.com/mastrocola-dev/infra/blob/main/.github/workflows/secret-expiry.yml) workflow in `infra` fails when a secret or certificate has no expiry or expires within 30 days, and that failure notification is the reminder.

## Inventory

| Secret | Source | Read by | Scope at the source |
|---|---|---|---|
| `cloudflare-api-token` | Cloudflare → My Profile → API Tokens | `id-infra` (`site` and `agent` workflows) | Zone → DNS, Zone Settings and Zone WAF → Edit, zone `mastrocola.dev` only, TTL set |
| `anthropic-api-key-ci` | Anthropic Console, workspace `ci` | `id-docs` | Workspace spend limit |
| `anthropic-api-key-runtime` | Anthropic Console, workspace `runtime` | `id-run-worker` | Workspace spend limit: the hard ceiling of the public runtime |
| `turnstile-secret-key` | Cloudflare → Turnstile → the site's widget | `id-run-api` | One widget, hostnames `mastrocola.dev` and `www.mastrocola.dev` |

| Certificate | Source | Read by | Validity |
|---|---|---|---|
| `origin-api` | Cloudflare → SSL/TLS → Origin Server, signed from a key generated locally | App Service resource provider, to serve `api.mastrocola.dev` | 1 year |

The certificate has its own procedure, since a private key is involved: [Origin certificate](https://github.com/mastrocola-dev/infra#origin-certificate) in `infra`. The steps below are for secrets.

The Static Web Apps deployment token is not stored; reset it with `az staticwebapp secrets reset-api-key --name stapp-portfolio-www --resource-group rg-portfolio-dev` whenever it may have leaked.

## Rotating

1. Create the new credential at the source with a 180-day expiry; when the source caps it lower, accept the cap
2. Store it with the **same** expiry the source shows — never later, or `secret-expiry` stays green while the credential dies at the source:

```bash
read -rs VALUE && az keyvault secret set --vault-name kv-mastrocola-dev --name <secret> --value "$VALUE" \
  --expires "$(date -u -d '<expiry at the source, or +180 days>' +%Y-%m-%dT%H:%M:%SZ)" --query attributes.expires -o tsv
```

3. Confirm the consumer works with the new value:

| Secret | Check |
|---|---|
| `cloudflare-api-token` | `site` and `agent` in `infra`, via dispatch, both green |
| `anthropic-api-key-ci` | any pull request touching `adr/` |
| `anthropic-api-key-runtime`, `turnstile-secret-key` | ask a question on [mastrocola.dev](https://mastrocola.dev) and get an answer |

4. Revoke the previous credential at the source
5. Run `secret-expiry` via dispatch and confirm it is green

Consumers always read the latest version, so no pipeline or repository changes: workflows read the vault in the step that uses the secret, and the runtime reads it on every request. Writing the value never involves Terraform: bootstrap declares each secret with a write-only placeholder and ignores `expiration_date`.

## Adding a secret

Add it to `secret_readers` in `infra/bootstrap/secrets.tf` with the identity that reads it, apply bootstrap, then store the real value as above and add a row to the inventory.

## Incident log

**2026-10-04** — After importing the real `origin-api` certificate over the placeholder, the check "issuer is no longer `Self`" kept failing although the import had worked: the issuer shown belongs to the certificate policy, which an import does not rewrite. The check now compares thumbprints.

**2026-10-01** — The first `anthropic-api-key-ci` was created in the Anthropic Console with a 30-day expiry and stored in Key Vault with 180 days. `secret-expiry` would have stayed green while the key expired at the source, failing the next ADR pull request. Caught by review before any failure; the key was reissued and the rule "Key Vault expiry equals the source expiry" added above.
