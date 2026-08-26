# pancakeswap-infinity-cl-bsc

A [nuthatch](https://github.com/nightswatchhq/nuthatch) nest: **PancakeSwap Infinity CL on BNB Smart Chain**.

Swaps, liquidity changes, pool initialisations and position lifecycle from the Infinity singleton
PoolManager and its PositionManager.

One binary, one config file, no graph-node, no gateway, no query fees.

## Why this exists

The published subgraph for this protocol (`8jFYxwKP8tNGSDisucpHRK1ojUchZd7ELd8zh2ugHGDN`, deployment
`QmVjXU7yQNyyLphPGqoz8iBzqu5YXphooFJn15JqZ6ZMFz`) has been unqueryable through the gateway since
2026-08-02. It hit one deterministic `nonFatalError` at BSC block 113,581,822:

```
Mapping aborted at src/mappings/swap.ts:39: unexpected null  (handler: handleSwap)
```

Line 39 is `const pool = Pool.load(poolId)!`. The `token0`/`token1` loads two lines below it are
guarded with `if (token0 && token1)`; the pool load is not. A `Swap` on a pool with no `Pool` entity
aborts the handler, and although the deployment keeps indexing to chain head, `health: unhealthy`
makes every response unattestable. The data underneath is current and unreachable. A republish under
the same subgraph ID is the only fix and it needs the publisher.

This nest was scaffolded from that same deployment CID in one command:

```sh
nuthatch init --from-subgraph QmVjXU7yQNyyLphPGqoz8iBzqu5YXphooFJn15JqZ6ZMFz
```

## What it indexes

**Chain:** `bsc`. **2 contracts**, **17 tables**, no factory.

| alias | address | start block |
|---|---|---|
| `pool_manager` | `0xa0ffb9c1ce1fe56963b0321b32e7a0302114058b` | 47,214,308 |
| `position_manager` | `0x55f4c8aba71a1e923edc303eb4feff14608cc226` | 47,215,015 |

Infinity is a singleton architecture, so there are no per-pool contracts to discover and no factory
rules to write. Two static addresses cover the whole protocol.

## Verified

Indexed blocks **118,167,017 to 118,170,309** and stored **21,305 rows** across 7 tables:
20,456 swaps, 514 pool liquidity modifications, 220 position liquidity modifications, 63 position
transfers, 45 position mints and 7 new pool initialisations. Every table below is generated from the
vendored ABIs, and the run above is what this nest actually decoded, not an estimate.

## Read this before trusting it

- **This is not a port of the subgraph's entities.** You get decoded event tables. The subgraph's
  derived surface - `totalValueLockedUSD`, `volumeUSD`, `PoolDayData`, `PoolHourData`, `Bundle` and
  the whole USD pricing graph - is computed inside the mapping WASM, not declared in the manifest, so
  nothing can read it out. `token0Price`/`token1Price` are recoverable from `sqrtPriceX96` with plain
  arithmetic; TVL and USD denomination are not, without pricing work this nest does not do.
- **It indexes more than the subgraph did.** The manifest allowlisted three events per contract. The
  vendored ABIs define more, and the `eth_getLogs` sweep is address-filtered, so widening the topic
  set costs no extra requests. `Donate`, `DynamicLPFeeUpdated`, `ProtocolFeeUpdated`, `MintPosition`
  and `ModifyLiquidity` on the PositionManager are real protocol state the subgraph never recorded.
- **Backfilling from deployment needs your own endpoint.** BSC's keyless public RPC refuses archive
  `eth_getLogs` outright (`"Archive requests require a personal token"`), and every free alternative
  tested on 2026-08-26 refuses too: three `bsc-dataseed` hosts return `limit exceeded`, `bsc.drpc.org`
  free times out, `1rpc.io/bnb` caps `getLogs` at 50 blocks. A recent window works on the default; a
  backfill from 47,214,308 - about 71 million blocks - does not. Pass `--rpc <your archive endpoint>`.
- The `Initialize` and `Swap` signatures carry the Infinity pool `id` as an indexed `bytes32`, not a
  pool address. Joins are on that id, not on a contract address.

## Run it

```sh
nuthatch init --from https://github.com/nightswatchhq/pancakeswap-infinity-cl-bsc
cd pancakeswap-infinity-cl-bsc
nuthatch dev --dir . --backfill 3000 --seal-direct
nuthatch sql --dir . "SELECT count(*) FROM pool_manager__swap"
```

The endpoint in `nuthatch.toml` is keyless and public, so this file is publishable: a `nuthatch.toml`
is pinned into the nest's content address and must never carry a credential. It is enough to follow
the tip. A **backfill** wants archive depth it does not have - pass your own with `--rpc`, and check
it first with `nuthatch doctor --rpc <url>`.

## Tables

```
pool_manager__donate
pool_manager__dynamic_l_p_fee_updated
pool_manager__initialize
pool_manager__modify_liquidity
pool_manager__ownership_transferred
pool_manager__paused
pool_manager__protocol_fee_controller_updated
pool_manager__protocol_fee_updated
pool_manager__swap
pool_manager__unpaused
position_manager__approval
position_manager__approval_for_all
position_manager__mint_position
position_manager__modify_liquidity
position_manager__subscription
position_manager__transfer
position_manager__unsubscription
```

## Licence

`MIT OR Apache-2.0`, same as nuthatch.
