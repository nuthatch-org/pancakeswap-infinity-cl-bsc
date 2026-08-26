---
name: nuthatch
description: Query this self-hosted nuthatch nest on bsc - decoded events, balances, and read-only SQL. Use when asked about on-chain activity for these contracts.
---

# Querying the nuthatch nest

Contracts indexed on bsc:
- `pool_manager` = 0xa0ffb9c1ce1fe56963b0321b32e7a0302114058b
- `position_manager` = 0x55f4c8aba71a1e923edc303eb4feff14608cc226

Data is local - never call an external API for it.

## Preferred: MCP
If a `nuthatch` MCP server is configured, use its tools. Call `schema` first to learn the
data model, then `sql` / `entity` / `balance` / `top_balances`.

## Fallback: HTTP (a `nuthatch dev` must be running)
- Recent rows:  `curl localhost:8288/entities?limit=20`
- Read-only SQL: `curl -G localhost:8288/sql --data-urlencode 'q=SELECT count(*) FROM transfers'`

`sql` sees finalized data only; balances/entity cover the live tip.
