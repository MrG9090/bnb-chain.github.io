---
title: MPP SDK
---

# MPP SDK (@bnb-chain/mpp)

EVM Charge implementation of the [Machine Payments Protocol (mppx)](https://github.com/wevm/mppx) — `draft-evm-charge-00`.

Brings the BNB Chain ecosystem (BSC, opBNB) plus the wider EVM landscape (Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, Linea) into the mppx HTTP payment authentication framework. Composes with `Mppx.create()` to expose `402 Payment Required` flows for `permit2`, `transaction`, `hash`, and `authorization` (EIP-3009) credentials.

## Capabilities

| Area                  | Supported                                                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Credential types**  | `authorization` (EIP-3009), `permit2` (single + batch with splits), `transaction` (EIP-1559), `hash`                           |
| **Challenge binding** | `mppx-managed` (under `Mppx.create`), `mppx-hmac` (bare verify), `stored-lookup` (draft §6 zero-deviation)                     |
| **Settlement**        | Server-side broadcast for `permit2` / `authorization` (settlement signer pays gas); payer-broadcast for `hash` / `transaction` |
| **Tokens / chains**   | Curated `(chain, token)` matrix — see [Tokens](#tokens) / [Chains](#chains)                                                    |
| **Receipt**           | `draft §7.6` `Payment-Receipt` via a browser-safe codec (`buildEvmReceipt` / `serializeEvmReceipt`)                            |
| **Replay protection** | 3-state atomic store (inflight / consumed / rejected); durable backend required in production                                  |

All four credential paths are live end-to-end — see [Examples](examples.md). For architecture, spec compliance, and replay protection, use the guides below. Release notes are managed with [Changesets](https://github.com/changesets/changesets).

v1 limits: curated token presets only (no arbitrary BYO ERC-20), and the SDK adds one spec extension (`methodDetails.permit2Spender`) that `draft-evm-charge-00` doesn't define but Permit2 settlement requires — see [Spec compliance](spec-compliance.md).

## Install

```bash
pnpm add @bnb-chain/mpp mppx viem
```

Peers: `mppx ^0.6.28`, `viem ^2.51.0`. Node ≥ 22 (development uses Node 22 stable).

## Tokens

The SDK ships a curated matrix — every `(chain, token)` pair lands alongside its explorer-verified address, on-chain `decimals()` confirmation, and matrix-lock unit tests. EIP-3009 entries additionally have their domain `name`/`version` derived from on-chain `DOMAIN_SEPARATOR()` and locked in the same test file.

The table below lists the **EIP-3009-enabled** anchors (where the `authorization` path is live). Broader issuer-native stablecoin coverage (probe-gated, `authorization` off) is summarized under [Expanded coverage](#expanded-coverage).

| Chain       | Token            | Contract                                                                                           | Decimals | EIP-3009                                                                       |
| ----------- | ---------------- | -------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------ |
| ethereum    | USDC             | [`0xa0b8...eb48`](https://etherscan.io/address/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)         | 6        | yes (Circle native, domain `USD Coin` / `2`)                                   |
| ethereum    | USDT             | [`0xdac1...1ec7`](https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7)         | 6        | no                                                                             |
| base        | USDC             | [`0x8335...2913`](https://basescan.org/address/0x833589fcd6edb6e08f4c7c32d4f71b54bda02913)         | 6        | yes (Circle native, domain `USD Coin` / `2`)                                   |
| bsc         | BINANCE_PEG_USDT | [`0x55d3...7955`](https://bscscan.com/address/0x55d398326f99059ff775485246999027b3197955)          | 18       | no                                                                             |
| bsc         | FDUSD            | [`0xc5f0...6409`](https://bscscan.com/address/0xc5f0f7b66764F6ec8C8Dff7BA683102295E16409)          | 18       | yes (First Digital Labs, domain `First Digital USD` / `1`)                     |
| bsc         | U                | [`0xcE24...6666`](https://bscscan.com/address/0xcE24439F2D9C6a2289F741120FE202248B666666)          | 18       | yes (United Stables `$U`, domain `United Stables` / `1`)                       |
| sepolia     | USDC             | [`0x1c7D...7238`](https://sepolia.etherscan.io/address/0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238) | 6        | yes (Circle native, domain `USDC` / `2`) — testnet, see [Examples](examples.md) |
| bsc-testnet | TEST_USDT        | TODO (pin verified contract before live tests)                                                     | 18       | no                                                                             |

### Expanded coverage

Issuer-native stablecoins curated across the supported chains. These advertise `permit2` / `transaction` / `hash`; `authorization` (EIP-3009) is **off pending a per-chain probe**. Contract addresses + decimals for every pair live in [`src/server/curated.ts`](https://github.com/bnb-chain/mpp-sdk/blob/main/src/server/curated.ts) and are locked by [`src/server/curated.test.ts`](https://github.com/bnb-chain/mpp-sdk/blob/main/src/server/curated.test.ts).

- **Circle USDC** — `arbitrum`, `optimism`, `polygon`, `avalanche`, `linea`, plus testnets `base-sepolia`, `arbitrum-sepolia`, `optimism-sepolia`, `polygon-amoy`, `avalanche-fuji`, `linea-sepolia`. Source: [Circle USDC addresses](https://developers.circle.com/stablecoins/usdc-contract-addresses).
- **Circle EURC** — `ethereum`, `base`, `avalanche`, plus testnets `sepolia`, `base-sepolia`, `avalanche-fuji`. Source: [Circle EURC addresses](https://developers.circle.com/stablecoins/eurc-contract-addresses).
- **PayPal USD (PYUSD)** — `ethereum`, `arbitrum`, plus testnets `sepolia`, `arbitrum-sepolia`. Source: [Paxos PYUSD](https://docs.paxos.com/stablecoin/pyusd/mainnet).
- **Paxos USDP / USDG** — `ethereum`. Source: [Paxos USDP](https://docs.paxos.com/guides/stablecoin/usdp/mainnet) / [Paxos USDG](https://docs.paxos.com/stablecoin/usdg/mainnet).
- **First Digital USD (FDUSD)** — `ethereum`, `arbitrum` (plus the existing `bsc` entry, which is EIP-3009-probed). Source: [First Digital Labs](https://www.firstdigitallabs.com/fdusd).
- **Tether USD₮** — `avalanche` only (issuer-native; bridged L2 USDT on Arbitrum/Optimism/Polygon is deliberately excluded). Source: [Tether supported protocols](https://tether.to/en/supported-protocols/).
- **BNB Chain (BSC)** — native EIP-3009 tokens `FDUSD` and `U` are in the table above. Bridged stablecoins use distinct `BINANCE_PEG_*` names: `BINANCE_PEG_USDC`, `BINANCE_PEG_USDT`, and `BINANCE_PEG_DAI` — all standard BEP-20, no EIP-3009.
- **opBNB** — deliberately **not** in the default matrix yet. Stablecoin provenance there must be cross-verified before any entry lands.

## Chains

| Preset                                                                      | chainId | Default confirmations | Notes        |
| --------------------------------------------------------------------------- | ------- | --------------------- | ------------ |
| ethereum                                                                    | 1       | 12                    |              |
| base                                                                        | 8453    | 1                     |              |
| arbitrum                                                                    | 42161   | 1                     |              |
| optimism                                                                    | 10      | 1                     |              |
| polygon                                                                     | 137     | 5                     | reorg buffer |
| avalanche                                                                   | 43114   | 1                     |              |
| linea                                                                       | 59144   | 1                     |              |
| bsc                                                                         | 56      | 3                     | reorg buffer |
| opbnb                                                                       | 204     | 1                     |              |
| sepolia / _-sepolia / _-amoy / avalanche-fuji / bsc-testnet / opbnb-testnet | various | 0                     | dev velocity |

Permit2 deployment is auto-probed at `preflightCharge` time via `eth_getCode` against the resolved address. v1 does not open arbitrary BYO chain — `rpcUrl` and `chainOverride` may only override an existing preset's RPC / viem `Chain` metadata.

---

## Documentation

| Guide | Description |
|-------|-------------|
| [Quickstart](quickstart.md) | End-to-end server + client walkthrough (BSC Testnet) |
| [Architecture](architecture.md) | Wire schema, server factory, verifiers, client constructors |
| [Spec compliance](spec-compliance.md) | `draft-evm-charge-00` mapping and extensions |
| [Replay store](replay-store.md) | Durable atomic replay protection |
| [Examples](examples.md) | `charge-server`, `charge-demo`, `bnb-wire-demo` |
| [ADR: permit2Spender](adr/0001-permit2-spender.md) | Why `methodDetails.permit2Spender` exists |

## Repository

[https://github.com/bnb-chain/mpp-sdk](https://github.com/bnb-chain/mpp-sdk)

[← Developer Kit overview](../index.md)
