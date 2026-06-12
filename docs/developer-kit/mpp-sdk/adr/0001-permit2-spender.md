---
title: MPP SDK ADR: permit2Spender
---

# ADR 0001 — `permit2Spender` wire field

- **Status:** Accepted
- **Scope:** `permit2` credential path only

## Context

`draft-evm-charge-00` defines the `permit2` credential: the client signs
an off-chain Permit2 authorization and the server submits the on-chain
`permitWitnessTransferFrom`, paying gas. The draft's `methodDetails`
(Table 2) carries `chainId` / `permit2Address` / `credentialTypes` /
`decimals` / `splits`. §6.1's verification rules say "the EIP-712
signature is valid and recovers to the source address" but never specify
what `spender` value the client signs.

That omission is load-bearing because of how Permit2 works. Permit2's
`SignatureTransfer` does NOT take `spender` as a function parameter — it
hardcodes `msg.sender` into the EIP-712 struct hash:

```solidity
// Uniswap/permit2 — src/libraries/PermitHash.sol, hash(PermitTransferFrom)
keccak256(abi.encode(
  _PERMIT_TRANSFER_FROM_TYPEHASH, tokenPermissionsHash,
  msg.sender,                       // the spender
  permit.nonce, permit.deadline
))
```

So a signature is only valid when the signer used `spender = the address
that ultimately calls Permit2` — i.e. the server's settlement signer EOA.
If the client signs with any other address, the signature recovers
correctly off-chain (the client's own recovery uses the same wrong
spender, so local checks pass) but the on-chain call reverts
`InvalidSigner()` (selector `0x815e1d64`).

The client cannot know the server's settlement signer address from the
draft-defined challenge. We need a channel for the server to publish it.

## Options considered

1. **Publish the spender in the challenge** (chosen). Add an OPTIONAL
   `methodDetails.permit2Spender`; server injects from
   `settlementAccount.address`; client signs with it; verifier
   cross-checks it.
2. **Require `recipient` == settlement signer.** No new field — the
   client signs `spender = recipient`. Rejected: forces the merchant's
   fund-receiving address to also hold the hot settlement key (or a
   contract), and breaks the common relayer pattern where the gas payer
   differs from the payee.
3. **Deploy a forwarder contract** whose address is the fixed spender.
   Rejected for v1: requires an on-chain deployment + a discovery
   mechanism for its address (which lands back at "publish an address in
   the challenge" anyway).
4. **Drop `permit2` support.** Rejected: it's the draft's RECOMMENDED
   credential type.

## Decision

Add `methodDetails.permit2Spender` (option 1). OPTIONAL on the wire;
REQUIRED for `permit2` credentials. The same field name + semantics as
[quiknode-labs/mpp](https://github.com/quiknode-labs/mpp), so independent
EVM Charge implementations interoperate.

- **Server** (`preflightCharge` + `charge`): derive from
  `settlementAccount.address`, inject into `defaults.methodDetails`, bind
  into the challenge HMAC (Tier-1 id coverage + Tier-2 `stableBinding`).
- **Client** (`createPermit2Credential`): read from the challenge, sign
  the EIP-712 `spender` with it; hard error if absent.
- **Verifier** (`verifyPermit2`): require it, assert it equals
  `ctx.settlementSigner.account.address`, use it for recovery + the
  on-chain call.

Unaffected: `hash`, `transaction`, `authorization`. EIP-3009 binds
`from`→`to`→`value` in the signed payload and lets anyone relay, so it has
no `msg.sender` constraint.

## Consequences

- Permit2 settlement works end-to-end against a server whose settlement
  signer differs from the recipient (the normal case).
- A tampered challenge that swaps `permit2Spender` while keeping the same
  `id` fails `Challenge.verify` (HMAC binding); even if binding were
  bypassed, the verifier's `permit2Spender === ctx.settlementSigner`
  cross-check fails closed.
- Interop cost: a strict `draft-evm-charge-00` client that ignores
  unknown `methodDetails` fields cannot produce a settleable Permit2
  credential against this server (it wouldn't know to sign with the
  spender). This is inherent to Permit2, not to our encoding — any
  conformant Permit2 integration needs the same information. We should
  push `permit2Spender` (or an equivalent) upstream into the draft.
- A server with no `settlementAccount` simply omits the field and doesn't
  advertise `permit2` (the credential set collapses to the no-signer
  subset).
