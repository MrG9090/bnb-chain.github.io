---
title: MPP SDK Spec Compliance
---

# Spec compliance & extensions

`@bnb-chain/mpp` implements
[`draft-evm-charge-00`](https://paymentauth.org/draft-evm-charge-00.html)
on top of [`mppx@0.6.28`](https://github.com/wevm/mppx) (commit
[`5aed74b`](https://github.com/wevm/mppx/tree/5aed74bfe46315ff3f27524ea8bb72e251bf771d)).

This document lists where the SDK extends or makes choices the draft
leaves open. There is exactly **one wire extension** (`permit2Spender`);
everything else is a compliant implementation choice.

## Extension: `methodDetails.permit2Spender`

**Status:** wire extension — not defined by `draft-evm-charge-00`,
required for `permit2` settlement.
**ADR:** [`adr/0001-permit2-spender.md`](adr/0001-permit2-spender.md).

### Why it exists

Permit2's `SignatureTransfer` reconstructs the EIP-712 hash using
`msg.sender` as the `spender` field — it is NOT a signed parameter:

```solidity
// Uniswap/permit2 — src/libraries/PermitHash.sol
function hash(ISignatureTransfer.PermitTransferFrom memory permit) internal view returns (bytes32) {
    bytes32 tokenPermissionsHash = _hashTokenPermissions(permit.permitted);
    return keccak256(
        abi.encode(_PERMIT_TRANSFER_FROM_TYPEHASH, tokenPermissionsHash,
                   msg.sender,            // ← the spender, hardcoded
                   permit.nonce, permit.deadline)
    );
}
```

So the payer MUST sign the Permit2 typed data with `spender = the address
that will call permitWitnessTransferFrom on-chain` = the server's
settlement signer. Signing with anything else (e.g. the Permit2 contract
address) produces a signature that recovers fine locally but reverts
`InvalidSigner()` (selector `0x815e1d64`) at settlement.

The draft is silent on this (its `methodDetails` table —
`chainId` / `permit2Address` / `credentialTypes` / `decimals` / `splits` —
has no spender field, and §6.1 doesn't show the signed `spender`). The
client therefore has no spec-sanctioned way to learn the settlement
signer's address. The SDK closes the gap by publishing it.

### Wire contract

| Field            | Type   | Required                                             | Description                                                       |
| ---------------- | ------ | ---------------------------------------------------- | ----------------------------------------------------------------- |
| `permit2Spender` | string | OPTIONAL on wire; REQUIRED for `permit2` credentials | Settlement signer EOA = `msg.sender` at the on-chain Permit2 call |

- **Server**: `preflightCharge` derives it from
  `settlementAccount.address` and `charge()` injects it into
  `defaults.methodDetails.permit2Spender` (only when a settlement signer
  is configured). It is bound into the challenge HMAC (mppx Tier-1 covers
  the full serialized request via the challenge `id`; the SDK's
  `stableBinding` Tier-2 also pins it explicitly).
- **Client**: `createPermit2Credential` reads
  `challenge.request.methodDetails.permit2Spender` and signs the EIP-712
  typed data with it. A challenge missing the field is a hard error
  (issued by a server pre-dating this fix → can't produce a settleable
  credential).
- **Verifier**: `verifyPermit2` requires the field, cross-checks it
  equals `ctx.settlementSigner.account.address` (rejects a tampered
  challenge that tries to redirect the spender), and uses it for both
  EIP-712 recovery and the on-chain call.

Affects only the `permit2` path. `hash` / `transaction` / `authorization`
do not have the `msg.sender` constraint (EIP-3009 binds `from`→`to` in the
signed payload; anyone may relay it).

Naming follows the [quiknode-labs/mpp](https://github.com/quiknode-labs/mpp)
convention.

## Compliance choices

### Challenge binding

`challengeBinding.mode` on `ServerParameters` — `mppx-managed` /
`mppx-hmac` / `stored-lookup`. See
[architecture.md](architecture.md#challenge-binding-modes). `stored-lookup`
is `draft §6` zero-deviation (constant-time compare against a persisted
snapshot, no server secret).

### Receipt compatibility (`draft §7.6`)

The receipt must carry `method` / `challengeId` / `reference` / `status`
/ `timestamp` / `chainId` (+ optional `externalId`). mppx's default
`Receipt.Schema` drops `challengeId` / `chainId`, so the SDK ships its own
`evmHttpTransport` (auto-wired by `charge()` on the per-method transport
slot) whose `Payment-Receipt` encoding is `serializeEvmReceipt` — a
deterministic, browser-safe codec independent of future mppx
`Receipt.serialize` changes. Deployments do NOT configure
`Mppx.create({ transport })`.

### v1 token support

- Curated token presets only — no arbitrary BYO ERC-20. `currency` on the
  wire is always the resolved curated address.
- Preset names mean the **native issuer's** token only. Bridged / wrapped
  variants must use a distinct preset name (e.g. no `(bsc, USDC)` — BSC
  has no Circle-native USDC).
- `authorization` is advertised only for `(chain, token)` pairs whose
  curated entry has `eip3009Supported: true` (with a verified EIP-712
  `name` / `version`).
- `TEST_USDT` is testnet-only and must never appear in a mainnet matrix
  entry or be treated as Tether-official.

See the [Tokens](index.md#tokens) table for the current matrix and
the per-preset semantics.

### Replay protection (`spec §9`)

3-state atomic store; durable backend required in production. See
[replay-store.md](replay-store.md).
