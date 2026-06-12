---
title: MPP SDK Examples
---

# Examples

Three runnable examples under `examples/`. The first two are designed to
run **together** — the browser demo drives real end-to-end flows against
the local server; the third (`bnb-wire-demo`) is a standalone CLI
inspector that needs no server.

| Example                                      | What it is                                                                                                                                                                                                     |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`charge-server`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-server) | Minimal Hono HTTP server using `@bnb-chain/mpp/server`. Six protected routes (article / download / tip / split / hash-only / stored-lookup) + a public `/api/config`, BSC Testnet USDT, `permit2` / `transaction` / `hash`. |
| [`charge-demo`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-demo)     | React + shadcn/ui + wagmi browser app driving the full client flow against the server.                                                                                                                         |
| [`bnb-wire-demo`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/bnb-wire-demo) | Standalone CLI wire-shape inspector for BNB Chain stablecoins — resolves each `(chain, token)` via the SDK and prints the challenge / credentialTypes / EIP-712 domain / receipt shape. No on-chain broadcast. |

## Running both together

```bash
# Terminal 1 — server on :3000 (needs examples/charge-server/.env)
pnpm --filter @bnb-chain/mpp-example-charge-server start

# Terminal 2 — demo on :5173
pnpm --filter @bnb-chain/mpp-example-charge-demo dev
```

The demo's Vite dev server proxies `/api/*` → `http://localhost:3000`
(override with `VITE_CHARGE_SERVER_URL`), so the browser talks to the
server with no CORS setup. Both `start` / `dev` run a `prestart` /
`predev` hook (`pnpm -C ../.. build`) to rebuild the SDK `dist/` first.

## charge-server

Configured for `bsc-testnet` / `TEST_USDT`, accepting `permit2` /
`transaction` / `hash` (the token is a plain BEP-20, no EIP-3009, so no
`authorization`). Because `permit2` settles server-side, it requires
`SETTLEMENT_PRIVATE_KEY` (a hot signer holding tBNB for gas) alongside
`RECIPIENT_ADDRESS` + `MPP_SECRET_KEY`.

Its `402` advertises `permit2Spender` (the settlement signer's address)
in `methodDetails` — required so Permit2 clients sign with the right
EIP-712 `spender` (see [spec-compliance.md](spec-compliance.md)).

For a fully payer-funded path (no server gas), hit `/api/hash-only` — its
handler advertises only `['transaction', 'hash']` and carries no
settlement signer. The sponsored routes (`/api/article`, `/api/download`,
`/api/tip`) also auto-degrade to that payer-funded handler when the
settlement signer runs low on balance / over its hourly gas budget, while
`/api/split` and `/api/stored/article` return `503` (they have no
payer-funded equivalent). See the hardening notes in the README.

Full setup + the client-side settlement snippet:
[`examples/charge-server/README.md`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-server/README.md).

## charge-demo

A 4-step flow (Fetch challenge → Build credential → Local verify → Submit
& settle), per credential type, with each type's state kept in its own
pool. End-to-end mode (default) does the real server roundtrip; toggle it
off for local-only wire-shape inspection. Includes a Permit2 allowance
panel that handles the one-time `approve(Permit2, max)`.

Per-credential realism (what's on-chain vs in-page), faucet links, and
the source layout:
[`examples/charge-demo/README.md`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-demo/README.md).

## bnb-wire-demo

A standalone CLI that resolves a matrix of BNB Chain stablecoins
`(chain, token)` through the SDK and prints, for each, the wire shapes a
real deployment would emit — resolved currency / decimals / chainId /
`permit2Address`, the advertised `credentialTypes`, the EIP-712 domain,
and the receipt shape. It does **no** on-chain broadcast and needs no
server or signer, so it's the fastest way to inspect how the curated
matrix resolves a given token before wiring up a charge-server.

```bash
pnpm --filter @bnb-chain/mpp-example-bnb-wire-demo start
```

## What you need on-chain (Sepolia)

- **hash** — the connected wallet broadcasts the transfer: needs Sepolia
  ETH (gas) + Sepolia USDC.
- **permit2** — wallet needs Sepolia USDC + a one-time Permit2 approval
  (the demo's allowance panel does it); the server's settlement signer
  needs Sepolia ETH for gas.
- **authorization** — wallet needs Sepolia USDC; server signer needs
  Sepolia ETH for gas. No prior approval.
- **transaction** — the demo signs with an in-page random key (MetaMask
  can't expose a pre-signed-unbroadcast RLP), so end-to-end settlement
  intentionally fails at broadcast (the key is unfunded). The wire shape
  is real; production callers sign with their own funded keystore.

Faucets: [sepoliafaucet.com](https://sepoliafaucet.com) (ETH),
[faucet.circle.com](https://faucet.circle.com) (USDC).
