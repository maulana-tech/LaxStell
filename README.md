# Lax-Stell

**A full-privacy platform on Stellar.** Bridge assets into a shielded layer, hold private multi-asset balances, send confidential payments, and trade on a zero-knowledge dark pool — all verified on-chain by Soroban smart contracts.

> Submission for **Stellar Hacks: Real-World ZK**.

## Deployed on Stellar Testnet

The full system is live and the private round-trip is **verified on-chain**. Current pool
(Lax-Stell build, redeployed 2026-07-15):
[`CBZNNVUKTG6YSVT3NGV7MDVL5ZQO5D4KLLIRFAGBCORPH7Q62ZHS5RP3`](https://stellar.expert/explorer/testnet/contract/CBZNNVUKTG6YSVT3NGV7MDVL5ZQO5D4KLLIRFAGBCORPH7Q62ZHS5RP3),
wired to 5 UltraHonk verifiers:

| Circuit | Verifier contract |
|---------|-------------------|
| `withdraw` | [`CDS245ZQLXFIYD2TPSJWZLAO6TOZOGRF6FQWAJ35J5SG6A7WNMHUMD5B`](https://stellar.expert/explorer/testnet/contract/CDS245ZQLXFIYD2TPSJWZLAO6TOZOGRF6FQWAJ35J5SG6A7WNMHUMD5B) |
| `transfer` | [`CCTHUAA3I4R2BRUQEFREHQ3AWVLTCZECAZ7JRG5S23FM44LP27RY5NZB`](https://stellar.expert/explorer/testnet/contract/CCTHUAA3I4R2BRUQEFREHQ3AWVLTCZECAZ7JRG5S23FM44LP27RY5NZB) |
| `place_order` | [`CABWY7YM7C4FCJBGQ7KG47N6NKZOBHWGMHAEYDTOGE6LDPOLZNGR2BF6`](https://stellar.expert/explorer/testnet/contract/CABWY7YM7C4FCJBGQ7KG47N6NKZOBHWGMHAEYDTOGE6LDPOLZNGR2BF6) |
| `match_orders` | [`CCKOCCPIYRRSCFNGDW3BDOGW4R2V7XY6KYHZVJDFB5KTKR5U3LPMAB5T`](https://stellar.expert/explorer/testnet/contract/CCKOCCPIYRRSCFNGDW3BDOGW4R2V7XY6KYHZVJDFB5KTKR5U3LPMAB5T) |
| `cancel_order` | [`CCU4JPTB4KRSG2N6YOTPT7SDXMYIC7RJOMEHUCR44FADEBRBWXQTTB2M`](https://stellar.expert/explorer/testnet/contract/CCU4JPTB4KRSG2N6YOTPT7SDXMYIC7RJOMEHUCR44FADEBRBWXQTTB2M) |

### Live ZK proof — deposit → withdraw round-trip (this pool)

Verified end-to-end on the current pool with a **real Noir/UltraHonk proof** (14,592-byte proof /
1,760-byte VK, keccak transcript) checked inside the Soroban contract — reproduce with
`source ./env.sh && scripts/e2e.sh`:

- **Deposit** 1 XLM → shielded note at leaf 0, commitment `0f090472…78e2…f92d0a`
  ([tx `cdaa631c…`](https://stellar.expert/explorer/testnet/tx/cdaa631c68bedd73a7cf469285e21c4d8ece913100baf9ae6f626db542dca614), SUCCESS).
- **Withdraw** with a real ZK proof → verified on-chain by the `withdraw` verifier, nullifier
  `02e885ea…5f85884`, 1 XLM released to the recipient
  ([tx `d2d2aca3…`](https://stellar.expert/explorer/testnet/tx/d2d2aca363087a082483b905d5e7ae11ede07d934ed9ccfd46ffcfe9c44ad313), SUCCESS).

The remaining flows are gated by the **same UltraHonk verifiers** and were verified live on the prior
byte-identical build (same VKs, verifier wasm and pool source):

- **Pay** — a 2-in/2-out **shielded transfer**, amounts hidden, value conserved in-circuit ([tx](https://stellar.expert/explorer/testnet/tx/8b8eed61eabd219c9d766f496ec19fc333549868fca2308cf7e63e00b8add90f)).
- **Swap** — hidden orders **placed**, **matched at the midpoint** ([tx](https://stellar.expert/explorer/testnet/tx/5bc05ebfa3f95849e6c6e3bff8375e6cfe09544e8c3318feb4096f81c7c4bdb3)), and **cancelled** with refund ([tx](https://stellar.expert/explorer/testnet/tx/51023894faf88329a2bd937c55ba05731860a5189aafe998e4964cf9881a4063)).
- **Soundness**: a tampered proof and a replayed nullifier are both rejected on-chain.

Full contract IDs, transactions, and a one-command reproduction are in [DEPLOYMENT.md](./DEPLOYMENT.md).

## Trustless cross-chain bridge — Ethereum → Stellar (verified live)

The **Bridge** is a genuine **trust-minimized cross-chain bridge**, not a relayer: assets locked on
Ethereum Sepolia arrive as **shielded notes** on Stellar, with provenance proven on-chain. The full
loop is verified live (no trusted relayer):

1. **Lock** 0.001 ETH on Sepolia (`LaxStellBridgeL1` `0xcF40c553…`).
2. An **Ethereum sync-committee BLS signature** is verified **natively on Soroban** (`EthLightClient`),
   recording a real Ethereum execution `state_root` on Stellar — the same trust model as Helios, but
   with **no SNARK-wrap** because Stellar has native BLS12-381 (the check is ~30M of the 100M budget,
   vs ~80M gas on the EVM).
3. **`bridge_in`** proves the lock with an **in-contract Merkle-Patricia storage proof** against that
   `state_root` and **mints a shielded note** ([tx `4b3760d1…`](https://stellar.expert/explorer/testnet/tx/4b3760d1f31b50da6a54bec54fe5f5645fe1719429f5acc05544c3a431289ffc), SUCCESS).

This is the hackathon's "wild" idea — a *cross-chain private bridge using Stellar's BN254/BLS12-381
compatibility to verify another chain's consensus*. Full evidence + reproduction in
[BRIDGE_DEPLOYMENT.md](./BRIDGE_DEPLOYMENT.md); design in [BRIDGE_SPEC.md](./BRIDGE_SPEC.md).

## Modules

| Module | Description |
|--------|-------------|
| **Bridge** | Move classic Stellar assets (XLM, USDC, …) in/out of Lax-Stell via the Stellar Asset Contract. |
| **Portfolio** | View and manage shielded, multi-asset balances. |
| **Pay** | Send private payments — amounts and participants hidden. |
| **Swap** | Dark-pool DEX — hidden orders, ZK-proven fair matching, atomic settlement. |

## How the ZK is load-bearing

Every state transition out of the shielded layer (withdraw, transfer, place/cancel order, match) is gated by an **UltraHonk (Noir) zero-knowledge proof** verified inside a Soroban contract. Without a valid proof, no funds move. Privacy comes from the circuit design (hidden inputs), and integrity from on-chain verification of BN254 / Poseidon2 — the primitives Stellar shipped in Protocol 25–26.

## Repository layout

```
circuits/noir/   Noir circuits (withdraw, transfer, place_order, match_orders, cancel_order)
contracts/       Soroban smart contracts (lax-stell-pool + UltraHonk verifier integration)
sdk/             TypeScript client library (notes, proofs, Merkle tree, tx building)
matcher/         Off-chain order-matching service
frontend/        React app (Bridge / Portfolio / Pay / Swap)
vendor/          Reference repos (rs-soroban-ultrahonk, noir-poseidon) — gitignored
SPEC.md          Full technical specification
SHARED.md        Cross-component invariants (crypto params, encodings) — source of truth
TOOLCHAIN.md     Pinned tool versions and install steps
```

## Quick start

```bash
source ./env.sh        # put nargo / bb / stellar on PATH
# circuits
cd circuits/noir/withdraw && nargo test
# contracts
cd contracts && cargo build --target wasm32-unknown-unknown --release
# sdk / frontend
pnpm install && pnpm -r build
```

See [SPEC.md](./SPEC.md) for the full design and [TOOLCHAIN.md](./TOOLCHAIN.md) for setup.

## Status

Hackathon work-in-progress. Built on Stellar **testnet**. Components marked as MVP / mock in their READMEs where applicable.
