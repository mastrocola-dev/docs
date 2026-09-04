# ADR-002: Public site on Azure Static Web Apps with DNS as code

**Status:** Accepted
**Date:** 2026-09-04

## Context

The organization needs a public landing page at `mastrocola.dev`. The domain is registered at GoDaddy and its DNS zone is managed by Cloudflare. The site is itself part of the portfolio: how it is hosted and deployed should demonstrate the same practices as the rest of the platform.

Options considered:

- **GitHub Pages** — simplest, but places the organization's storefront outside the Azure foundation it exists to showcase.
- **Azure Storage static website** — requires a CDN in front for HTTPS on a custom domain, adding cost and moving parts incompatible with the budget guardrails.
- **Azure Static Web Apps (Free plan)** — custom domain with managed TLS, global edge, preview environments per pull request, at zero cost. `.dev` is on the HSTS preload list, so mandatory HTTPS comes for free with the managed certificate.

DNS records could be managed manually in the Cloudflare dashboard or declared in Terraform alongside the resources they point to.

## Decision

- Host the public site on **Azure Static Web Apps, Free plan**, declared in a new `site` root module in `infra`, with its own state key and pipeline.
- Manage the `mastrocola.dev` DNS records with the **Cloudflare Terraform provider** in the same module. Domain validation uses a TXT token, so the domain is validated before traffic moves — cutover from the registrar's parked page is atomic.
- Cloudflare proxying stays **off** for site records: Static Web Apps has its own global edge, and proxying interferes with domain validation.
- Site content lives in a separate `www` repository, deployed by its own pipeline using the Static Web Apps deployment token.

## Consequences

Positive:

- Zero hosting cost within Free plan limits (2 custom domains, 100 GB/month bandwidth, 250 MB per environment) — far above the needs of a static landing page
- The entire path from DNS record to served page is versioned and auditable
- Preview environments per pull request without any additional setup
- The `www` repository needs no Azure federated credential, preserving the 20-credential limit

Negative, accepted:

- Two static secrets enter the project, breaking the OIDC-only pattern: the Static Web Apps **deployment token** (can only publish static content) and the Cloudflare **API token** (scoped to DNS edit on the single zone). Both have a small, contained blast radius and no path to Azure control-plane access.
- Static Web Apps is not available in `brazilsouth`; the resource metadata lives in `eastus2`. Content is served from the global edge, so user-facing latency is unaffected.
- Free plan has no SLA — acceptable for a portfolio site; upgrading to Standard is a plan change, not a migration.
