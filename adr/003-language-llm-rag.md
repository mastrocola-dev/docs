# ADR-003: Agent language, LLM integration, and RAG decoupling

**Status:** Accepted
**Date:** 2026-09-23

## Context

The agent (docs/architecture, Arquitetura_agente.md) starts as a single agent, with tools exposed via MCP from the outset. Two open items were blocking the start of implementation: the host language and how to access the LLM. There was also the question of how to accommodate RAG later without reopening this decision.

## Decision

**1. Host language (`service-agent`): TypeScript.**
The official TypeScript MCP SDK is the protocol's reference implementation. Strict typing via the compiler gives a guarantee equivalent to what Python only reaches with extra tooling (mypy/pyright). Aligns with the backend stack already used elsewhere (Node/Nest).

**2. LLM integration: direct Anthropic API, official SDK.**
No Microsoft Foundry in the MVP. The API key lives in Key Vault, as a small-blast-radius static secret — same pattern as the two secrets already accepted by ADR-002. Rationale: access to all models and new features without waiting on Foundry availability, and no Marketplace subscription requirement. Migrating to Foundry with Managed Identity (removes this secret) stays open for when the agent leaves the local CLI and moves to Container Apps — at that point, re-evaluate against Foundry's model availability at the time.

**3. RAG: deferred, and implemented in Python when it exists.**
MCP decouples the host from tools by protocol, not by imported library. This lets a future `mcp-rag` run in Python — a more mature ecosystem for embeddings and vector search — without the TypeScript host needing to change. The host's language does not lock in the MCP servers' language.

## Repository structure (confirmed)

- `service-agent`: agent host, TypeScript
- `mcp-*`: one repository per MCP server; language decided per server (e.g. `mcp-rag` in Python)

## Consequences

Positive:
- Multi-language across the org is the expected pattern, not a case-by-case exception to document
- Strong typing in the host with no extra configuration overhead
- RAG can be added without touching the host

Negative / trade-offs:
- A TypeScript host introduces a build step, which contrasts with the "no build" minimalism adopted for the site (`www`) — accepted because the agent is a long-running process, not a static artifact
- Direct Anthropic API keeps one more static secret in Key Vault until the eventual migration to Foundry/Managed Identity