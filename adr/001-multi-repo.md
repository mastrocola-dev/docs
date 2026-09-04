# ADR-001: Multi-repo over monorepo

**Status:** Accepted
**Date:** 2026-09-04

## Context

The organization started as a single repository ("portfolio") holding infrastructure code and documentation together. It serves two goals that must not conflict: a public portfolio demonstrating architectural competence, and a foundation that can evolve into a real product company.

A repository structure is itself an architectural statement. The organization's stated specialty is decomposing monoliths into event-driven services; keeping everything in one repository would contradict that message.

## Decision

One repository per concern, under the `mastrocola-dev` organization:

- `.github` — organization profile and inherited templates
- `docs` — architecture documentation, ADRs, runbooks
- `infra` — infrastructure as code
- `service-*` — one per domain, created as they emerge

Cross-cutting documentation is centralized in `docs`; operational documentation lives with the code it operates.

## Consequences

Positive:

- Each repository is independently readable: focused README, own pipeline, own history
- Independent versioning and delivery per component
- Per-repository access control — individual repos can go private when real clients exist
- Organization-level secrets and templates are shared without duplication

Negative, accepted:

- CI and tooling conventions must be replicated across repos — mitigated by a future `template-service`
- Cross-repo changes require coordinated pull requests
- Repository transfer and renaming during migration invalidated the name-based OIDC federation subjects, requiring credential recreation (see `runbooks/github-azure-oidc.md`)

## Notes

The original repository history was preserved: the repo was transferred and renamed to `infra`, and documentation history was extracted into `docs` with `git filter-repo`.