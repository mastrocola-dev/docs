# ADR-004: TypeScript without a build step and MCP server distribution

**Status:** Accepted
**Date:** 2026-09-29

## Context

ADR-003 chose TypeScript for the agent host and expected it to introduce a build step. Node 24 now strips type annotations natively, so `.ts` files run directly. The agent MVP was built on that basis in two repositories, `service-agent` and `mcp-docs`, which raised two cross-repository questions: which conventions every TypeScript repository shares, and how a host obtains the MCP servers it starts. Node refuses type stripping inside `node_modules`, so a server installed as a package would need to ship compiled JavaScript.

## Decision

**1. TypeScript runs without a build.** Node 24 executes `.ts` directly; `tsc` runs only as a CI gate (`noEmit`). `erasableSyntaxOnly`, `verbatimModuleSyntax` and `allowImportingTsExtensions` keep every file strippable in isolation, which forbids `enum`, `namespace`, parameter properties and decorators.

**2. MCP servers are not distributed as packages.** Locally, a host starts each server from a sibling checkout, declared per agent instance (`command` and `args`). In the cloud, each server runs as its own Container App over streamable HTTP, as planned in the agent architecture. Neither path needs a package.

**3. Shared conventions for TypeScript repositories.** Node 24 pinned in `.nvmrc`; Biome pinned to an exact version (no semicolons, single quotes, `lineWidth: 320`); EditorConfig as the single source of indentation; `node:test` with native V8 coverage and tests selected by explicit glob; CI gates `check`, `typecheck` and `test:coverage`. Conventions are copied into each repository; `template-service` is extracted when a third repository shows what is truly common.

## Consequences

Positive:
- No `dist/`: what runs is what is read, stack traces point at sources, and container images copy `src` as is
- One set of conventions across hosts and servers, enforced by CI rather than by discipline
- MCP servers stay independently versioned and deployable, as ADR-001 intends

Negative / trade-offs:
- Decorator-based frameworks, NestJS included, are excluded from these repositories
- Local development requires the sibling layout (`service-agent/`, `mcp-*/`, `docs/`)
- Publishing a server to a registry would require adding a build step to that server
- ADR-003's expected build step no longer applies; its decisions otherwise stand
