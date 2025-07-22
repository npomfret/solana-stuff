# Solana PDA Derivation Patterns — **Detailed Reference**

## Table of Contents

0. [TL;DR Quick Recap](#0—tldr-quick-recap)
1. [The Core Concept: PDAs](#1—the-core-concept-pdas)
2. [Pattern A – User‑Centric PDAs](#2—pattern-a--user-centric-pdas)
3. [Pattern B – Account‑Centric PDAs](#3—pattern-b--account-centric-pdas)
4. [Pattern C – Singleton / Global PDAs](#4—pattern-c--singleton--global-pdas)
5. [Associated Token Accounts (ATA)](#5—associated-token-accounts-ata)
6. [Pattern D – Metaplex Standard](#6—pattern-d--metaplex-standard)
7. [Governance & Multisig](#7—governance--multisig)
8. [Advanced Moves & Edge Cases](#8—advanced-moves--edge-cases)
9. [Security & Gotchas](#9—security--gotchas)
10. [Reference Table — Popular Protocols](#10—reference-table--popular-protocols)
11. [Handy Utilities (TypeScript)](#11—handy-utilities-typescript)
12. [Further Reading](#12—further-reading)

---

## 0 — TL;DR Quick Recap

- A **Program‑Derived Address (PDA)** is produced by iterating SHA‑256 over `(seeds || bump || programId)` until the 32‑byte digest *falls off* the Ed25519 curve—meaning **no private key exists**.
- Hard limits: **≤ 16 seeds**, **each ≤ 32 bytes**. `bump ∈ 0‥255` is the retry‑counter that finds an off‑curve point.
- **Deriving ≠ creating.** After deriving, you still allocate the account (e.g. via `SystemProgram.createAccount`, `create_account`, or Anchor’s `init`).
- Programs “sign” for their PDAs with `invoke_signed` (or Anchor’s `CpiContext::with_signer`).

```typescript
import { PublicKey } from "@solana/web3.js";

export const derivePda = (
    seeds: (Buffer | Uint8Array)[],
    programId: PublicKey,
): [PublicKey, number] => PublicKey.findProgramAddressSync(seeds, programId);
```

---

## 1 — The Core Concept: PDAs

A **PDA** is a deterministic address owned by a program, allowing that program to create and manage accounts without holding a private key.  Think of it as a *predictable table row* where the primary‑key is derived from business data (the *seeds*).  When a client and a program use the **same seed recipe**, they will always reach the same address, enabling trust‑minimised coordination.

> **Why not just random addresses?**  Because every Solana account must be explicitly listed in the transaction upfront. Predictability avoids on‑chain look‑ups that would otherwise require an extra transaction.

Key points for newcomers:

- **Seeds are arbitrary byte arrays.** Most protocols use UTF‑8 strings, public keys, or little‑endian integers.
- A PDA can itself be used as a seed for *another* PDA (nested PDAs).
- Derivation is **pure math**—no RPC request is required; it runs locally in JS/TS, Rust, Go, etc.

---

## 2 — Pattern A – User‑Centric PDAs

These PDAs encode a user’s `Pubkey` directly (sometimes plus extra context).

| Sub‑Pattern             | Seed Recipe               | Typical Use‑Case                | Notable Protocols                |
| ----------------------- | ------------------------- | ------------------------------- | -------------------------------- |
| **A1 — Classic**        | `[b"user", user]`         | One profile per user            | Clone, Vaultka, Grass, Zelo, Lulo, Aurory |
| **A2 — User + State**   | `[state, b"voter", user]` | Per‑market/realm positions      | Helium DAO, 01 Protocol, Phoenix, Marinade, Jupiter, Hawksight, Magic Eden, Star Atlas |
| **A3 — User + Token**   | `[b"stake", mint, user]`  | Multi‑token staking or deposits | Allbridge, Nosana, Flash |
| **A4 — User + Counter** | `[b"user", user, u16LE]`  | Multi‑account (sub‑accounts)    | Drift, Zeta, Parcl |
| **A5 — Franken‑Seeds**  | Any mix of the above      | Complex DeFi / LP positions     | Parcl, Kamino, Francium, Save |

> **Tip – Little‑Endian Counters**  Use a fixed‑width `BN` when serialising counters so the seed length never changes.

```typescript
import { BN } from "@coral-xyz/anchor";

const subAccountId = 2; // u16
const [subAccountPda] = derivePda([
  Buffer.from("user"),
  wallet.toBuffer(),
  new BN(subAccountId).toArrayLike(Buffer, "le", 2),
], PROGRAM_ID);
```

---

## 3 — Pattern B – Account‑Centric PDAs

PDAs derived from other **on‑chain accounts** rather than the user directly.

### B1 — Simple Child

`[b"vault", parent]` → custody vaults (Raydium CLMM, Orca Whirlpool, Picasso, Pyth, Loopscale).

### B2 — Nested / Hierarchical

Use an existing PDA as part of the seed for deeper nesting. Example (Tensor margin):

```typescript
const [groupPda] = derivePda([Buffer.from("margin_group"), authorityPubkey.toBuffer()], PROGRAM_ID);
const [marginPda] = derivePda([Buffer.from("margin"), groupPda.toBuffer(), traderPubkey.toBuffer()], PROGRAM_ID);
```

---

## 4 — Pattern C – Singleton / Global PDAs

When a program needs exactly **one** instance of something—e.g. a config or fee‑collector.

```typescript
const [configPda] = derivePda([Buffer.from("config")], PROGRAM_ID);
```

Examples: Vaultka’s `LENDING` & `STRATEGY` accounts, many DAO registries, Tensor's global margin account, NX Finance's global ID.

---

## 5 — Associated Token Accounts (ATA)

Program ID: `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL` (Associated Token Program).

The ATA is the canonical SPL‑token account for `(owner, mint)`. Seeds:

`[owner, TOKEN_PROGRAM_ID, mint]`

Allocation is zero‑rent‑exempt bytes (created on‑demand). Every wallet tooling understands ATAs.

```typescript
const [ata] = derivePda([
  wallet.toBuffer(),
  TOKEN_PROGRAM_ID.toBuffer(),
  mint.toBuffer(),
], ASSOCIATED_TOKEN_PROGRAM_ID);
```

---

## 6 — Pattern D – Metaplex Standard

Program ID: `metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s`

| Account Type                | Seed Recipe                                                          |
| --------------------------- | -------------------------------------------------------------------- |
| Metadata                    | `[b"metadata", programId, mint]`                                     |
| Master Edition              | `[b"metadata", programId, mint, b"edition"]`                         |
| Collection Authority Record | `[b"metadata", programId, mint, b"collection_authority", authority]` |

Used by **every** NFT on Solana.

---

## 7 — Governance & Multisig

- **SPL Governance:** `[b"governance", realm, governingTokenMint]`
- **Safe Multisig:** `[b"safe", owner, indexLE]`

---

## 8 — Advanced Moves & Edge Cases

| Technique                 | Why / When                                                         | Code Sketch                                           |
| ------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------- |
| **Hashed Seeds**          | Raw data >32 B (e.g. large strings)                                | `const seed = sha256(utf8.encode(name)).slice(0,32)`  |
| **Sliced Seeds**          | Take first 31 bytes of a public key (`pubkey[0..31]`) to fit limit | `pub.toBuffer().slice(0, 31)` (used by **Loverflow**) |
| **Multi‑Chain Addresses** | When seeding with EVM addresses, drop the `0x` prefix & hex‑decode | `Buffer.from(addr.slice(2), "hex")` (**DeBridge**)    |
| **Cross‑Program PDA**     | Use PDA‑A as a seed for PDA‑B                                      | Safe because PDAs are off‑curve                       |
| **Nested PDAs**           | Build trees of authority / state                                   | `[b"child", parentPda, idxLE]`                        |
| **Numeric Prefixes**      | Space-efficient alternative to string prefixes                     | `[1, 0, owner, market]` (**Kamino**)                  |
| **Exotic Integer Encoding** | Using `i32` or `i64` for seeds                                     | `i32ToBytes(startIndex)` (**Raydium**, **Meteora**)   |
| **Seeds from Account Data** | Using a value from another account's data as a seed                | `[b"promotion_authority", seedValue]` (**Lulo**)       |
| **Singleton-as-Seed**     | Using a global PDA as a seed for other PDAs                        | `[b"margin", singletonPda, owner]` (**Tensor**, **NX Finance**) |

Anchor macro for reference:

```rust
#[account(
  seeds = [b"user", authority.key().as_ref()],
  bump
)]
pub struct User {/* fields */}
```

---

## 9 — Security & Gotchas

1. **Never trust client‑submitted seeds.**  Re‑derive on‑chain.
2. Respect the **32‑byte seed limit**—fixed‑width LE encoding avoids surprise length changes.
3. Distinguish **mint vs token‑account**; mixing them yields a different address.
4. Two different programs ≠ same PDA, even with identical seeds.
5. Verify signers: in raw Rust, call `assert_eq!(pda, Pubkey::create_program_address(...))`; Anchor does this automatically.
6. **Avoid `createWithSeed`:** This deprecated method is less flexible and can be brittle. Prefer `findProgramAddressSync`. The **Save** protocol uses this.
7. **Avoid custom seed generation:** The **Save** protocol also uses a fragile custom seed generation function. This is highly discouraged. Stick to simple, well-understood seed recipes.

---

## 10 — Reference Table — Popular Protocols

| Protocol                  | PDA Recipe (simplified)                    |
| ------------------------- | ------------------------------------------ |
| **01 Protocol**           | `[owner, state, b"marginv1"]`              |
| **Adrena**                | `[b"user_staking", owner, stakingPda]`      |
| **Allbridge**             | `[b"user_deposit", tokenAddress, owner]`    |
| **Aurory**                | `[owner]`                                  |
| **Clone**                 | `[b"user", owner]`                          |
| **DeBridge**              | `[receiptBuffer, season, evmOwner]`        |
| **Drift**                 | `[b"user", owner, subAccountIdLE]`          |
| **Elemental**             | `[b"position", ownerAddress, poolAddress]` |
| **Flash**                 | `[b"stake", owner, poolKey]`                |
| **Francium**              | `[userPublicKey, farmingPoolAccount, associatedAccount]` |
| **Grass**                 | `[owner, b"userStakeInfo"]`                 |
| **Hawksight**             | `[b"multi-user", HAWKSIGHT_FARM, owner]`    |
| **Helium Voter**          | `[registrar, b"voter", user]`              |
| **Jupiter Escrow**        | `[b"Escrow", locker, user]`                |
| **Kamino**                | `[1, 0, owner, market, pair[0], pair[1]]`  |
| **Loopscale**             | `[b"order_ledgers", order]`                 |
| **Loverflow**             | `[owner.slice(0, 31)]`                     |
| **Lulo**                  | `[b"promotion_authority", seedValue]`       |
| **Magic Eden**            | `[b"m2", m2AuctionHouse, owner]`            |
| **Marinade**              | `[b"claim_record", hardcodedPk, owner]`     |
| **Meteora DLMM**          | `[b"bin_array", lbPair, indexLE]`           |
| **Meteora Farm**          | `[owner, farm]`                            |
| **Nosana**                | `[b"stake", nosMint, owner]`                |
| **NX Finance Lend**       | `[b"lending_account", nxMarket, lending_pool_pda, owner]` |
| **NX Finance Stake**      | `[stakePool, b"nx_account", owner]`         |
| **NX Finance Leverage**   | `[globalIdPda, owner, b"account"]`          |
| **Orca Whirlpool**        | `[b"whirlpool", config, mintA, mintB]`     |
| **Parcl LP Position**     | `[b"lp_position", market, owner, indexLE]` |
| **Phoenix Seat**          | `[b"seat", market, trader]`                |
| **Picasso**               | `[b"vault_params", ei]`                     |
| **Pyth**                  | `[b"custody", stakingAccount]`              |
| **Quarry**                | `[b"Miner", quarry, owner]`                 |
| **Rain**                  | `[b"poolv2", owner]`                        |
| **Raydium CLMM Position** | `[b"position", nftMint]`                   |
| **Raydium Stake**         | `[hardcodedPk, owner, b"staker_info_v2_associated_seed"]` |
| **Save**                  | `createWithSeed(owner, customSeed)`        |
| **Star Atlas**            | `[b"Proxy", proxyEscrowAddress, owner]`     |
| **Tensor Margin**         | `[b"margin", singletonPda, owner, 0LE]`     |
| **Vaultka**               | `[b"USER_INFOS", owner]`                    |
| **Zelo**                  | `[b"user-v2", owner]`                       |
| **Zeta Margin**           | `[b"margin", group, user]`                 |

---

## 11 — Handy Utilities (TypeScript)

```typescript
/** Convert bigint → fixed‑width LE buffer (u64 = 8 bytes) */
export const u64le = (n: bigint) => {
  const buf = Buffer.alloc(8);
  buf.writeBigUInt64LE(n);
  return buf;
};

/** Convert number → fixed-width LE buffer (i32 = 4 bytes) */
export const i32le = (n: number) => {
    const arr = new ArrayBuffer(4);
    const view = new DataView(arr);
    view.setInt32(0, n, true);
    return new Uint8Array(arr);
}

/** First 31 bytes of a PublicKey buffer – avoids >32 B seeds */
export const first31 = (pub: PublicKey) => pub.toBuffer().subarray(0, 31);
```

---

## 12 — Further Reading

- Solana Docs – [Program Derived Addresses](https://docs.solana.com/developing/programming-model/calling-between-programs#program-derived-addresses)
- Metaplex – [Understanding PDAs](https://developers.metaplex.com/programming-guides/pdas)
- Helius – [PDA Deep Dive](https://www.helius.dev/blog/solana-pda)