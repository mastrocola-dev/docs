# docs

Architecture documentation for [mastrocola.dev](https://github.com/mastrocola-dev). If a decision matters, it is written here.

## Map

```
adr/            Architecture Decision Records — why things are the way they are
architecture/   C4 diagrams and context maps — what things are
runbooks/       Operational procedures — how to act when something needs doing
```

## Decision records

Numbered, immutable once accepted, superseded rather than edited.

| ADR | Decision | Status |
|---|---|---|
| ADR-001 | Multi-repo over monorepo | Draft |

Format: context, decision, consequences. One page maximum.

## Runbooks

| Runbook | Covers |
|---|---|
| `github-azure-oidc.md` | Federated credential setup, immutable subject format, diagnosing `AADSTS700213` | 

## Conventions

- Documentation that explains *architecture* lives here; documentation that explains *operation of a specific repo* lives in that repo's README
- Diagram sources (`.mermaid`, `.drawio`) are committed alongside their exports
- English for all published content