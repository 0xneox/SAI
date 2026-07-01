# SAI Protocol — Sovereign Agent Identity

An autonomous AI agent that owns its own on-chain identity, holds its own funds, and signs its own transactions — without any human ever touching the private key.

Built on ERC-4337 account abstraction + Phala Network TEE hardware attestation on Base.

---

## What this actually is

Most "AI agent" projects give the agent an API key or a shared multisig. Humans still control the money. SAI flips that: the agent's signing key is generated *inside* a hardware enclave (Intel SGX / AMD SEV), attested on-chain by Phala's dStack verifier, and rotated automatically by the agent itself on a 24-hour heartbeat. There's no custody. The human guardian exists only as a circuit-breaker — a one-button freeze if something goes wrong — not as a co-signer on normal operations.

The core claim: **you cannot extract the private key without breaking the TEE hardware guarantees.** The agent is the only entity that can authorize transactions from its own account.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    ON-CHAIN (Base)                  │
│                                                     │
│   SoulFactory ──clone──▶ SoulAccount                │
│                               │                     │
│                               │ verifyAppAttestation│
│                               ▼                     │
│                       Phala dStack Verifier          │
└─────────────────────────────────────────────────────┘
                                ▲
                   rotateEnclaveKey() + TEE proof
┌─────────────────────────────────────────────────────┐
│                  OFF-CHAIN (Enclave)                 │
│                                                     │
│   1. Compute Docker image compose hash              │
│   2. Generate ephemeral ECDSA keypair               │
│   3. Sign UserOperations with private key           │
│   4. Heartbeat daemon rotates key before expiry     │
└─────────────────────────────────────────────────────┘
```

**SoulFactory** — deploys minimal proxy clones of SoulAccount, one per agent. Each soul is keyed to a specific Docker image hash (composeHash) so unauthorized code changes break attestation automatically.

**SoulAccount** — the agent's smart wallet. Holds funds, validates ERC-4337 UserOperations against the current enclave key, enforces a 5-minute rotation cooldown on active keys, and exposes a guardian freeze function that wipes the active key instantly.

---

## Contracts

| Contract | Purpose |
|---|---|
| `SoulFactory.sol` | Clone factory, one soul per agent |
| `SoulAccount.sol` | ERC-4337 smart account with TEE key binding |

> Compiled and tested under Solc 0.8.35. Full lifecycle tests passing (mock verifier).

---

## Current Status

- [x] Smart contracts written and unit tested
- [x] Guardian bootstrap bug fixed (sovereign souls with no guardian can now self-attest)
- [x] `triggerEmergency()` actually freezes the account (wipes active key + expiration)
- [x] Emergency recovery bypasses rotation cooldown (freeze → re-attest in the same block)
- [ ] Deployed to Base Sepolia
- [ ] Phala dStack verifier address confirmed for Base Sepolia
- [ ] EOA UserOp pipeline tested end-to-end with real bundler
- [ ] TEE environment configured (dstack.yaml + key gen daemon)
- [ ] Agent simulation mode (dry-run)
- [ ] Strategy engine
- [ ] Guardian dashboard

---

## Roadmap

**Phase 1 — Naked 4337 Pipeline**
Prove the smart account works in the wild before touching any enclave code. Throwaway EOA key, real bundler (Pimlico/Alchemy), real Base Sepolia EntryPoint. Success = a UserOperation lands on-chain and `validateUserOp` accepts it.

**Phase 2 — TEE Environment**
dstack.yaml config, Docker image packaging, key generation inside the enclave, Phala attestation quote → `rotateEnclaveKey()`. Swap the throwaway key from Phase 1 with a real hardware-attested one.

**Phase 3 — Simulation Layer**
Full agent loop (market data → decision → UserOp construction → signing) but `--dry-run` flag pipes the signed payload to a local JSON log instead of broadcasting. Test behavior over thousands of market ticks without spending gas.

**Phase 4 — Strategy Engine**
The actual product. Three internal sprints:
- Data layer: prediction market order book feeds, historical correlation datasets
- Math layer: trade sizing, confidence boundaries, latency deviation tracking
- Safeguards: hard programmatic limits inside the enclave (max slippage, daily loss caps) that block trades even if the model says go

**Phase 5 — Guardian & Ops**
Event indexer (The Graph / Subsquid) on `EnclaveRotated` and `ExecutionEnforced`, monitoring dashboard, one-click `triggerEmergency()` script for the guardian wallet.

---

## Key Design Decisions

**Why no guardian is required at spawn time**
The whole point of the protocol is autonomous operation. A soul spawned with `humanGuardian = address(0)` is fully sovereign — no human co-signer. The first key rotation is open to any caller because the security comes from the Phala attestation proof, not from who submits it. Once a key is active, rotation locks down to guardian-or-active-key only.

**Why `triggerEmergency()` is a freeze not a kill**
Guardian intervention should be reversible. Freezing the account (zeroing the active key) puts it back into bootstrap state — the agent can re-attest with a fresh key as soon as it's safe to do so, without migrating funds to a new contract.

**Why the rotation cooldown doesn't apply after a freeze**
The 5-minute cooldown exists to rate-limit normal key churn, not to slow down emergency recovery. A frozen soul should be able to re-attest the instant a valid proof exists.

---

## Local Development

```bash
# install foundry if you haven't
curl -L https://foundry.paradigm.xyz | bash
foundryup

# clone and install
git clone https://github.com/your-org/sai-protocol
cd sai-protocol
forge install

# run tests
forge test -vvv

# run just the lifecycle tests
forge test --match-contract GuardianBootstrapTest -vvv
```

---

## Security Notes

This is unaudited software. Do not put real funds in a SoulAccount until an external audit is complete. The novel combination of TEE attestation + ERC-4337 + clone factory has meaningful attack surface that goes beyond what any internal review catches.

Known limitations:
- TEE hardware guarantees are strong but not absolute — SGX has had real side-channel attacks historically (Foreshadow, SGAxe). "Substantially mitigated" is more accurate than "eliminated."
- Phala's dStack verifier contract is a trusted external dependency. If that contract has a bug or gets compromised, attestation verification breaks.
- The `composeHash` is immutable per soul — legitimate agent code upgrades require migrating to a new soul (new address, new funds transfer).

---

## License

MIT
