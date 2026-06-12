---
title: MPP SDK Replay Store
---

# Replay store

The replay store is the durable, atomic backend that guarantees a given
credential settles **at most once**. It's independent of
`challengeBinding`: the same store backs `mppx-managed`, `mppx-hmac`, and
`stored-lookup`. Spec §9 +
[`draft-evm-charge-00`](https://paymentauth.org/draft-evm-charge-00.html)
both require it.

Implementation: `src/server/Replay.ts`.

## State machine

Each credential maps to a deterministic key (see [Keys](#keys)). A slot
moves through three states:

```
            reserve (atomic CAS — fails if key already present)
   (absent) ─────────────────────────────▶ inflight
                                              │
              settle succeeds on-chain        │  settle fails BEFORE
            ┌───────────────────────────────┐ │  on-chain commit
            ▼                                 │ ▼
        markConsumed                          release ──▶ (absent)
            │                                    (retryable — nonce/tx
            ▼                                     not consumed on-chain)
        consumed  (terminal — replay rejected)

              settle committed on-chain but a
              post-commit check failed (e.g. Transfer-log
              mismatch, or a store write threw after success)
                          │
                          ▼
                      rejected  (terminal — known-bad, NOT retryable;
                                 the nonce/tx IS consumed on-chain)
```

- **reserve** — atomic compare-and-set. Reserving a key that's already
  `inflight` / `consumed` / `rejected` MUST fail without racing. This is
  what stops two concurrent requests from both settling the same
  credential.
- **release** — only valid from `inflight`, and only when settlement did
  NOT commit on-chain (broadcast rejected, simulate failed, balance /
  allowance check failed). Returns the slot to absent so a corrected
  retry can proceed.
- **markConsumed** — `inflight` → `consumed` after a confirmed on-chain
  settlement. Terminal.
- **markRejected** — `inflight` → `rejected` when the credential is
  known-bad in a way that consumed the on-chain nonce/tx anyway (so it
  can never replay, but we don't pretend it succeeded). Terminal.

### Terminal-commit phase (double-spend guard)

Once a verifier confirms the on-chain settlement succeeded, it flips an
internal `terminalPhase` flag. After that point the verifier MUST NOT
`release` the slot even if a _subsequent_ step throws (e.g. the
`Transfer`-log assertion fails, or the `markConsumed` store write itself
errors). Releasing post-commit would let the same already-settled
credential be replayed for a second on-chain settlement. In the
terminal phase a failure routes to `markRejected` (best-effort) and the
slot stays non-absent — never back to absent.

### Store-error normalization

Store backends throw heterogeneously (Redis `ECONNRESET`, Postgres pool
timeout, etc.). `getReplaySlot` / the reserve-and-settle path normalize
these into `ReplayStoreUnavailableError` so the verifier can decide
deterministically. The terminal-phase gate above takes precedence: a
store error AFTER on-chain commit never releases the slot.

## Keys

Keys are namespaced per credential type + deployment so the same nonce on
two different Permit2 deployments (or two chains) doesn't collide. Factory
helpers in `Replay.ts`, e.g. `permit2Key(chainId, permit2Address, signer,
nonce)`. The key always includes the dimensions that make a settlement
unique on-chain (chain, contract, signer, nonce/tx-hash) so replay
protection matches on-chain replay protection exactly.

## Production requirements

Spec §9 requires the store to be:

1. **Durable across processes / pods.** A single Node-process `Map` makes
   replay protection per-pod on a multi-pod deployment — N pods could each
   settle the same credential once. Not acceptable in production.
2. **Atomic.** `reserve` under an already-`consumed` key MUST fail
   without racing.

What the SDK enforces vs. what it can't:

| `NODE_ENV`            | `params.store` omitted                                 |
| --------------------- | ------------------------------------------------------ |
| `production`          | `preflightCharge` **throws** at startup                |
| `development` / unset | defaults to `Store.memory()` + one-time `console.warn` |
| `test`                | silent default to `Store.memory()` (no log noise)      |

When a store IS provided under `production`, it's accepted on presence
alone — the SDK can't structurally tell a Redis client from a `Map`
wrapper across the FFI boundary. Durability is therefore a
deployment-side claim: pass a real durable store and own the §9 promise.

## Suggested durable backends

- **Redis** — `SET key value NX PX <ttl>` for atomic reserve. Upstash,
  ElastiCache, self-hosted.
- **Postgres** — `INSERT ... ON CONFLICT DO NOTHING` for atomic reserve.
  Neon, Supabase, RDS.
- **Cloudflare KV / Durable Objects** — `put` with conditional-write
  (KV) or the single-writer model (DO, stronger consistency).

The store implements `ChargeStore` (an `mppx` `Store.AtomicStore`-shaped
interface). `Store.memory()` from mppx is acceptable **only** for tests
and local single-process dev.
