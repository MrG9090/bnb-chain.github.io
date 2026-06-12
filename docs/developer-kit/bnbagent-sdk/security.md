---
title: BNBAgent SDK Security
---

## Security

### Wallet & key handling

- **Encrypted keys** — `EVMWalletProvider` uses Keystore V3; plaintext keys are cleared from memory after import.
- **Submit-time verification** — `submit_result()` re-verifies `FUNDED`, assignment, expiry, and `budget >= service_price` before every on-chain submission.
- **Budget protection** — Underpriced jobs are rejected with HTTP 402 at `/status`, `/job/{id}/verify`, and at submit time inside `submit_result()`.
- **Permissionless settle** — `router.settle` is callable by anyone. The SDK does not gatekeep settlement; operators run their own settle script when ready.
- **Non-pausable refund** — `claimRefund` on the kernel is intentionally not pausable and not hookable: funds can always be reclaimed past `expiredAt`.
- **Storage permissions** — `LocalStorageProvider` uses `0600`/`0700`.

### EIP-712 typed-data signing (`SigningPolicy`)

`EVMWalletProvider.sign_typed_data` is **policy-gated by default**. Without
explicit configuration, the wallet only accepts EIP-3009
`TransferWithAuthorization` / `ReceiveWithAuthorization` against the registered
U-token deployments (BSC mainnet/testnet). All Permit variants
(ERC-2612 `Permit`, Permit2 `PermitSingle`/`PermitBatch`) are denylisted —
even if your own code mistakenly allowlists them, the denylist wins.

The threat: U token (and most ERC-20s) support EIP-2612 `Permit` on-chain.
Without `SigningPolicy`, an LLM agent receiving a 402 challenge from a
malicious server could be talked into signing a Permit that grants unbounded
allowance, draining the wallet over time. The default policy refuses
unconditionally; you opt in explicitly when you know what you're signing.

**Canonical example — direct SDK usage:**

```python
from bnbagent import EVMWalletProvider, X402Signer
from bnbagent.networks import get_address, BSC_MAINNET_CHAIN_ID

U = get_address(BSC_MAINNET_CHAIN_ID).payment_token

# Strict default applied automatically — zero config needed for U-token TWA.
wallet = EVMWalletProvider(password=os.environ["WALLET_PASSWORD"])

# Pass a scoped signer (not the wallet) to your @tool functions:
signer = X402Signer(
    wallet,
    max_value_per_call={U: 1_000_000},   # 1 USDC equivalent
    session_budget={U: 50_000_000},      # 50 USDC across this session
)

def pay_for_resource(challenge: dict, expected_to: str) -> dict:
    return signer.sign_payment(
        domain=challenge["domain"],
        types=challenge["types"],
        message=challenge["message"],
        expected_to=expected_to,   # caller MUST commit to the payee
    )
```

`X402Signer` enforces (a) byte-equal `expected_to == message['to']`
(case-insensitive), (b) `message['from'] == wallet.address` (so a tampered
challenge cannot authorize a payment "from" another account or burn the
session budget on a doomed sign), (c) per-call `max_value`, (d) cumulative
session budget. `expected_to` MUST come from a source independent of the 402
response (config / on-chain registry) — never from the challenge body itself.
The underlying `SigningPolicy` simultaneously enforces (chain_id,
verifyingContract) allowlist, primary-type allowlist/denylist, and
validity-window bounds (default ≤ 600s window / ≤ 900s future).

**Extending the policy for custom contracts:**

```python
from bnbagent import EVMWalletProvider, SigningPolicy
from bnbagent.networks import BSC_MAINNET_CHAIN_ID

extended = SigningPolicy.strict_default().extend(
    domain_allowlist={(BSC_MAINNET_CHAIN_ID, "0xMyCustomVerifyingContract")},
)
wallet = EVMWalletProvider(
    password=os.environ["WALLET_PASSWORD"],
    signing_policy=extended,
)
```

**Capability model:** registered agent tool functions must never close over
a raw `WalletProvider` — they should receive an `X402Signer` (or any other
scoped wrapper) instead.

**Tests-only escape:** `SigningPolicy.permissive()` disables all gates and
logs a WARNING; `EVMWalletProvider._DANGEROUS_sign_typed_data_no_policy()`
bypasses the gate per-call and logs the caller's filename+line. Both are
audit-friendly; production / agent-reachable code MUST NOT call them.

**Inspecting the current policy** at runtime:

```python
wallet = EVMWalletProvider(password=...)
print(wallet.signing_policy)
# SigningPolicy(
#   domain_allowlist (2 entries):
#     - chain_id=56 verifyingContract=0xcE24439F2D9C6a2289F741120FE202248B666666
#     - chain_id=97 verifyingContract=0xc70B8741B8B07A6d61E54fd4B20f22Fa648E5565
#   primary_type_allowlist=['ReceiveWithAuthorization', 'TransferWithAuthorization']
#   primary_type_denylist=['Permit', 'PermitBatch', 'PermitSingle']
#   validity: window<=600s, future<=900s, required_for=[...]
#   allow_unknown_domain=False
# )
```

`SigningPolicy.to_dict()` / `from_dict()` round-trip the policy through
plain dicts (JSON / TOML-friendly) so downstream tools (CLIs, deploy
manifests) can store and reload configurations declaratively.

### Decision tree — "do I need to configure anything?"

```
What are you signing?
│
├── EIP-3009 TransferWithAuthorization / ReceiveWithAuthorization
│   against U-token on BSC mainnet (56) or testnet (97)
│   → ✅ zero config — strict_default() already allows it
│
├── Same EIP-3009 type but a different token / chain
│   (e.g. USDC on Ethereum mainnet)
│   → 🟡 extend domain_allowlist with (chain_id, token_address)
│
├── A custom typed-data primary type
│   (e.g. "MyOrder" / "BondQuote" / "Auction")
│   → 🟡 extend primary_type_allowlist with the type name
│      AND extend domain_allowlist with the verifying contract
│
├── EIP-2612 Permit  /  Permit2 PermitSingle/PermitBatch
│   (unbounded allowance grants)
│   → ❌ refused unconditionally (denylist takes precedence)
│      Don't sign these in agent flows.
│
├── Permit2 PermitTransferFrom / PermitBatchTransferFrom
│   (single-use signature transfer — safer subset)
│   → 🟡 opt in by extending primary_type_allowlist;
│      witness validation stays caller-side unless / until the x402
│      ecosystem standardises around Permit2
│
└── A longer validity window (e.g. 30-minute authorizations)
    → 🟡 extend max_validity_window_seconds=1800
```

| Scenario | Extension snippet |
|---|---|
| Add a custom token on chain 56 | `extend(domain_allowlist={(56, "0xMyToken")})` |
| Add a custom primary type "MyOrder" on chain 56 / contract X | `extend(domain_allowlist={(56, X)}, primary_type_allowlist={"MyOrder"})` |
| Allow Ethereum-mainnet USDC | `extend(domain_allowlist={(1, "0xA0b8...eB48")})` |
| Opt into Permit2 SignatureTransfer | `extend(primary_type_allowlist={"PermitTransferFrom"})` |
| Widen validity to 30 min | `extend(max_validity_window_seconds=1800)` |

Examples: see [`examples/security_e2e.py`](https://github.com/bnb-chain/bnbagent-sdk/blob/main/examples/security_e2e.py) (signing + recovery loop, 6 assertions)
and [`examples/x402_buyer_demo.py`](https://github.com/bnb-chain/bnbagent-sdk/blob/main/examples/x402_buyer_demo.py) (complete buyer flow with mock 402 server).

Full design rationale and threat model: see ADR #30 in the
[bnbchain-studio](https://github.com/bnb-chain/bnbchain-studio) repo
(`docs/decisions.md`).

---
