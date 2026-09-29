# ADR-005: Agent-generated content lives at its source and is read at runtime

**Status:** Accepted
**Date:** 2026-09-29

## Context

The `adr-index` agent instance summarizes the ADRs into structured JSON. The public site should show that index, reflecting documentation changes without a site deploy. Three questions cross repositories: where the agent runs, how its output reaches the site, and who reviews text written by a model before it is published.

Options considered:

- **Agent per page view** — a Static Web Apps function runs the agent on each request. Rejected: cost and 10–30 s latency per visit, an API key exposed to abuse, and it contradicts the agent architecture, where HTTP access is asynchronous and never request/response.
- **Generation in `www`** — a scheduled workflow regenerates the index and opens a pull request in `www`. Rejected: changes arrive only after a site deploy, and a credential for the model moves into the site repository.
- **Generation in `docs`, triggered by a docs pull request** — `GITHUB_TOKEN` can write only to its own repository, so the output stays next to its source and no cross-repository token is needed.

## Decision

- The `docs` pipeline runs `adr-index` on every pull request touching `adr/` and commits `index/adrs.json` to the same branch. The index is reviewed together with the ADR that changed it; merging is the human approval.
- `index/adrs.json` records the git tree hash of `adr/` it was generated from. A check on `main` fails when the hash no longer matches, so a stale index is detected without calling the model. Unchanged trees skip generation, at zero cost.
- The site fetches the index from `raw.githubusercontent.com` at page load. Model output is untrusted data: the site validates identifiers, statuses and paths, and inserts text as text, never as markup. A Content Security Policy restricts `connect-src` to that origin.
- The Anthropic API key is a repository secret in `docs`, the third static secret, bounded by a spend limit on its Anthropic workspace.

## Consequences

Positive:

- A merged ADR reaches the site on the next reload, within the raw content cache (about five minutes), with no deploy
- `www` still holds no credential beyond the Static Web Apps deployment token
- Every published summary passed a review, and every generation leaves a trace artifact with tokens and cost

Negative, accepted:

- The site depends at runtime on `raw.githubusercontent.com`, which has no SLA and is not meant as a CDN; acceptable for portfolio traffic, and the page degrades to its static content when the fetch fails
- The index section requires JavaScript
- Commits pushed by `GITHUB_TOKEN` do not trigger workflows, so the regenerated commit carries no checks of its own; the check on `main` covers it after merge
- A generated file lives in a repository of hand-written documentation
- A third static secret; it moves to Key Vault with managed identity when the agent runs in Azure
