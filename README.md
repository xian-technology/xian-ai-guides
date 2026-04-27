# xian-ai-guides

`xian-ai-guides` is a small, focused collection of context files that
help LLMs generate accurate Xian smart contracts, GraphQL queries
against the indexed BDS surface, and matching test cases. Upload these
files to an LLM (Claude, ChatGPT, LM Studio, …) as context, then ask
it to produce contracts, queries, or tests.

This repo is a content set, not an application. It contains no runtime
code — just the prompts and references LLMs need to stay aligned with
current Xian behavior.

## Quick Start

Upload the relevant file(s) to an LLM and reference them in your prompt.

```text
Using the contracting guide provided, create a staking contract
for the Xian blockchain that supports per-pool reward deposits and
emergency withdrawals.
```

```text
Using the BDS GraphQL schema provided, write a query that returns
the most recent 50 transfers for token contract `currency`,
including the block height, timestamp, sender, recipient, and amount.
```

## Principles

- **Context, not code.** The repo holds the *prompt-time* knowledge an
  LLM needs to produce correct Xian artifacts. Application logic lives
  in other repos.
- **Stays in sync with the runtime.** When `xian-contracting`,
  `xian-abci`, or the indexed surface in `xian-py` changes in a way
  that affects how to write contracts or query data, this repo is the
  destination for the prompt-side update.
- **One topic per file.** Each guide covers one surface so LLMs can be
  primed selectively without dumping unrelated context.
- **Plain-text and JSON only.** Files are formats every LLM consumes
  cleanly: Markdown for prose, JSON for schemas.

## Key Files

- `contracting-guide.md` — comprehensive Contracting reference for LLMs:
  language semantics, state types, decorators, standard library bridge,
  common patterns, and gotchas. Use this when prompting for contract
  generation, refactoring, or review.
- `bds_graphql_schema.json` — full BDS GraphQL schema dump. Use this
  when prompting for indexed queries (blocks, transactions, events,
  state history, balances, developer rewards, etc.).

## Typical Use Cases

- generate or refactor Contracting-style smart contracts
- generate Contracting-aware test cases that match the runtime
  semantics
- generate GraphQL queries against a BDS-enabled Xian node
- review contract code or queries for inconsistencies with the current
  runtime surface

## Related Repos

- [`../xian-contracting/README.md`](../xian-contracting/README.md) — contracting runtime that the guide describes
- [`../xian-py/README.md`](../xian-py/README.md) — Python SDK with the indexed BDS query helpers
- [`../xian-ai-skills/README.md`](../xian-ai-skills/README.md) — packaged Claude Code / Anthropic skills for Xian work
- [`../xian-mcp-server/README.md`](../xian-mcp-server/README.md) — MCP server that exposes Xian to AI assistants programmatically
