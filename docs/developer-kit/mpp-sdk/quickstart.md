---
title: MPP SDK Quickstart
---

# Quickstart

A runnable, end-to-end walkthrough of `@bnb-chain/mpp`: stand up a server
that gates a route behind a stablecoin payment (`402 Payment Required`),
then build a client credential, submit it, and read the `Payment-Receipt`.

This guide uses **BSC Testnet (chainId 97) + `TEST_USDT`** so you can run it
end-to-end without mainnet funds — the same pair the bundled
[`examples/`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples) run on. Swap to any other curated `(chain, token)`
pair (e.g. `ethereum` / `USDC`) by changing two strings; see
[Curated pairs](#curated-chain--token-pairs).

> Prefer reading code? [`examples/charge-server`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-server)
> + [`examples/charge-demo`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-demo) are the full, running
> version of everything below.

## Contents

- [Install](#install)
- [Concepts in 30 seconds](#concepts-in-30-seconds)
- [1. Server — protect a route](#1-server--protect-a-route)
- [2. Client — pay the 402](#2-client--pay-the-402)
- [3. Read the receipt](#3-read-the-receipt)
- [Curated chain / token pairs](#curated-chain--token-pairs)
- [Challenge binding modes](#challenge-binding-modes)
- [Production checklist](#production-checklist)

## Install

```bash
pnpm add @bnb-chain/mpp mppx viem
```

Peers: `mppx ^0.6.28`, `viem ^2.51.0`. **Node ≥ 22.**

Three entry points:

| Import | Use it for |
| --- | --- |
| `@bnb-chain/mpp/server` | The server factory (`chargeAsync` / `preflightCharge`), composed with `Mppx.create()`. |
| `@bnb-chain/mpp/client` | The four credential constructors (`createHashCredential`, `createPermit2Credential`, …). |
| `@bnb-chain/mpp` | Universal helpers — `chargeFromDecimal` (decimal → base units) and the `Payment-Receipt` codec. |

## Concepts in 30 seconds

```
client                                    server (@bnb-chain/mpp/server)
  │                                       │
  │ GET /article (no credential)          │
  │ ─────────────────────────────────────►│ 402 + WWW-Authenticate: Payment <challenge>
  │ ◄─────────────────────────────────────│ (challenge binds chain/token/amount/recipient)
  │                                       │
  │ build credential for one of:          │
  │   hash | transaction | permit2 |      │
  │   authorization                       │
  │                                       │
  │ GET /article                          │
  │ Authorization: Payment <credential>   │
  │ ─────────────────────────────────────►│ verify → settle on-chain (permit2) or
  │ ◄─────────────────────────────────────│ confirm the payer's tx (hash) →
  │ 200 + Payment-Receipt: <receipt>      │ 200 + content
```

- **`hash` / `transaction`** are payer-funded — the payer broadcasts their own
  transfer; the server only verifies it. No settlement signer needed.
- **`permit2` / `authorization`** are server-settled — the server broadcasts
  `permitWitnessTransferFrom` / `transferWithAuthorization`, so it needs a
  funded **settlement signer** (`settlementAccount`).
- A token only advertises `authorization` if it implements EIP-3009 (e.g.
  Circle USDC). Plain BEP-20s like `TEST_USDT` advertise
  `['permit2', 'transaction', 'hash']`.

## 1. Server — protect a route {#1-server--protect-a-route}

`chargeAsync(params)` resolves the curated `(chain, token)` to a contract
address + decimals, probes the Permit2 deployment, and returns a charge
method. Hand it to `Mppx.create()`, then call
`handler.evm.charge({ amount })(request)` per route: it returns a `402`
challenge when there's no valid credential, or settles and gives you a
`withReceipt()` wrapper when there is.

```ts
// server.ts — run with: node --import tsx --env-file=.env server.ts (Node ≥22)
import { serve } from '@hono/node-server'
import { Hono } from 'hono'
import { Mppx } from 'mppx/server'
import { privateKeyToAccount } from 'viem/accounts'

import { chargeFromDecimal } from '@bnb-chain/mpp'
import { chargeAsync } from '@bnb-chain/mpp/server'

// Settlement signer — broadcasts Permit2 settlement; must hold gas (tBNB on
// BSC Testnet). Required because TEST_USDT advertises permit2.
const settlementAccount = privateKeyToAccount(
  process.env.SETTLEMENT_PRIVATE_KEY as `0x${string}`,
)

const charge = await chargeAsync({
  chain: 'bsc-testnet',
  token: 'TEST_USDT',
  recipient: process.env.RECIPIENT_ADDRESS as `0x${string}`, // your merchant address
  settlementAccount,
  challengeBinding: { mode: 'mppx-managed' },
  // Optional: override the curated default RPC.
  rpcUrl: 'https://data-seed-prebsc-1-s1.binance.org:8545',
  // store: createDurableReplayStore(...) // REQUIRED in production — see checklist
})

const handler = Mppx.create({
  methods: [charge],
  secretKey: process.env.MPP_SECRET_KEY!, // HMAC challenge-binding key
  // No `transport` — chargeAsync auto-wires evmHttpTransport (receipt codec).
})

const app = new Hono()

app.get('/article', async (c) => {
  // 1 TEST_USDT (18 decimals) → base units. The amount is per-route; the
  // (chain, token, recipient) come from chargeAsync above.
  const { amount } = chargeFromDecimal({ amount: '1', decimals: 18 })

  const result = await handler.evm.charge({ amount })(c.req.raw)
  if (result.status === 402) return result.challenge // sends WWW-Authenticate

  // Paid + verified + settled. withReceipt attaches the Payment-Receipt header.
  return result.withReceipt(
    c.json({ title: 'Premium Article', paidAt: new Date().toISOString() }),
  )
})

serve({ fetch: app.fetch, port: 3000 }, (i) =>
  console.log(`listening on http://localhost:${i.port}`),
)
```

`.env`:

```bash
RECIPIENT_ADDRESS=0xYourMerchantAddress
MPP_SECRET_KEY=...            # openssl rand -hex 32
SETTLEMENT_PRIVATE_KEY=0x...  # 32-byte hex; fund with tBNB (https://testnet.bnbchain.org/faucet-smart)
```

Probe the challenge phase:

```bash
curl -i http://localhost:3000/article
# HTTP/1.1 402 Payment Required
# WWW-Authenticate: Payment id="…", realm="…", method="evm", intent="charge", request="<base64url>", …
```

> Already have a route hook before `chargeAsync` resolves? Use the two-step
> form: `const prepared = await preflightCharge(params)` (does the curated
> resolution + Permit2 probe) then `charge(prepared)` (synchronous).
> `chargeAsync` is just sugar for `charge(await preflightCharge(params))`.

## 2. Client — pay the 402 {#2-client--pay-the-402}

Deserialize the challenge from the server's `WWW-Authenticate` header, then
call the constructor matching one of the advertised
`methodDetails.credentialTypes`. Each constructor returns the **complete**
`Authorization` header value (the `Payment ` prefix is already included — do
not add it again).

All signing inputs (`chainId`, `currency`, `recipient`, `amount`,
`permit2Address`) must equal the challenge's — read them off
`challenge.request`.

```ts
// pay.ts — a Node payer (tsx). In a browser, see the wallet note below.
import { Challenge } from 'mppx'
import {
  http,
  createWalletClient,
  encodeFunctionData,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { bscTestnet } from 'viem/chains'

import { createHashCredential, createPermit2Credential } from '@bnb-chain/mpp/client'

const URL = 'http://localhost:3000/article'
const payer = privateKeyToAccount(process.env.PAYER_PRIVATE_KEY as `0x${string}`)

// ── Fetch the 402 challenge ──────────────────────────────────────────────
const res = await fetch(URL)
if (res.status !== 402) throw new Error(`expected 402, got ${res.status}`)
const challenge = Challenge.deserialize(res.headers.get('WWW-Authenticate')!)

const req = challenge.request as {
  amount: string
  currency: `0x${string}`
  recipient: `0x${string}`
  methodDetails: { chainId: number; permit2Address: `0x${string}`; credentialTypes: string[] }
}
const { amount, currency, recipient } = req
const { chainId, permit2Address } = req.methodDetails
```

### Option A — `hash` (payer broadcasts, then references the tx)

```ts
// Broadcast the ERC-20 transfer yourself (needs the token + gas), then point
// the credential at the resulting tx hash.
const wallet = createWalletClient({ account: payer, chain: bscTestnet, transport: http() })
const txHash = await wallet.sendTransaction({
  to: currency,
  data: encodeFunctionData({
    abi: [{ type: 'function', name: 'transfer', stateMutability: 'nonpayable',
      inputs: [{ name: 'to', type: 'address' }, { name: 'amount', type: 'uint256' }],
      outputs: [{ type: 'bool' }] }],
    functionName: 'transfer',
    args: [recipient, BigInt(amount)],
  }),
})

const credential = await createHashCredential({ challenge, hash: txHash })
```

### Option B — `permit2` (sign EIP-712; the server settles)

```ts
// Permit2 nonces are unordered + single-use, so a fresh random 256-bit value
// per credential is fine (uniqueness, not sequence, is what matters).
const randomNonce = (): string => {
  const b = crypto.getRandomValues(new Uint8Array(32))
  return BigInt('0x' + [...b].map((x) => x.toString(16).padStart(2, '0')).join('')).toString()
}

const credential = await createPermit2Credential({
  challenge,
  account: payer,            // a viem LocalAccount
  chainId,
  permit2Address,            // from the challenge — NOT a hard-coded constant
  currency,
  recipient,
  amount,
  nonce: randomNonce(),                                  // unordered, single-use
  deadline: String(Math.floor(Date.now() / 1000) + 600), // +10 min
  // splits: omit — the SDK reads them from the challenge if present
})
```

> Permit2 prerequisite: the payer must have approved Permit2 once for the
> token (`ERC20.approve(permit2Address, max)`), and the credential is signed
> against the server's `methodDetails.permit2Spender` (the SDK reads it from
> the challenge automatically).

### Submit the credential

```ts
const paid = await fetch(URL, { headers: { Authorization: credential } })
console.log(paid.status)                          // 200
const receiptHeader = paid.headers.get('Payment-Receipt')!
console.log(await paid.json())
```

> **Browser wallets (MetaMask, etc.):** pass an `account` whose
> `signTypedData` delegates to the wallet (e.g. wagmi's
> `walletClient.signTypedData`) instead of a `privateKeyToAccount`. See
> [`examples/charge-demo/src/actions`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-demo/src/actions)
> for the adapter.

## 3. Read the receipt

The `Payment-Receipt` header is a `draft §7.6` EVM receipt. Decode it with
the codec from the top-level barrel:

```ts
import { deserializeEvmReceipt } from '@bnb-chain/mpp'

const receipt = deserializeEvmReceipt(receiptHeader)
// { method, challengeId, reference, status, timestamp, chainId, externalId? }
// `reference` is the on-chain settlement / transfer tx hash.
```

## Curated chain / token pairs {#curated-chain--token-pairs}

`chain` / `token` are restricted to the curated matrix (v1 — no arbitrary
BYO ERC-20). A few pairs:

| `chain` | `token` | decimals | notes |
| --- | --- | --- | --- |
| `bsc-testnet` | `TEST_USDT` | 18 | testnet (this guide); permit2 / transaction / hash |
| `ethereum` | `USDC` | 6 | EIP-3009 → also `authorization` |
| `base` | `USDC` | 6 | EIP-3009 → also `authorization` |
| `bsc` | `BINANCE_PEG_USDT` | 18 | permit2 / transaction / hash |

An unsupported pair throws `CuratedLookupError`. The full matrix +
verification notes live in `src/server/curated.ts`.

## Challenge binding modes

`challengeBinding` is required on `ServerParameters`:

- `{ mode: 'mppx-managed' }` — the common path under `Mppx.create()`; mppx
  runs `Challenge.verify` + `Expires.assert`. Pair with `secretKey` on
  `Mppx.create()`.
- `{ mode: 'mppx-hmac', secretKey }` — bare `Method.toServer(...).verify`
  path for custom (non-`Mppx.create`) hosts.
- `{ mode: 'stored-lookup', challengeStore }` — HMAC-free (draft §6); the
  server persists each issued challenge (`rememberChallenge`) and compares
  canonical bytes at verify.

## Production checklist

- **Replay store** — pass a durable, atomic `store` (Redis / Postgres /
  Cloudflare KV). Under `NODE_ENV=production`, omitting it throws at startup.
  `Store.memory()` is dev/test only. See [`replay-store.md`](replay-store.md).
- **Settlement signer** — fund it and treat the key as a hot wallet (rotate,
  scope per-deployment). Only needed for permit2 / authorization.
- **RPC** — pin your own provider via `rpcUrl`; public endpoints rate-limit.
- **Hardening** — rate-limit, gas budget, and degrade-to-payer-funded
  patterns are shown in [`examples/charge-server`](https://github.com/bnb-chain/mpp-sdk/tree/main/examples/charge-server)
  (`src/hardening.ts`).

## See also

- [`examples.md`](examples.md) — the runnable examples
- [`architecture.md`](architecture.md) — how the pieces fit
- [`spec-compliance.md`](spec-compliance.md) — `draft-evm-charge-00` mapping + the `permit2Spender` extension
- [`replay-store.md`](replay-store.md) — durable replay-store backends
