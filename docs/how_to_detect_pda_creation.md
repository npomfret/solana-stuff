# How to Detect PDA Creation

*The practical guide for spotting new Program‑Derived Addresses linked to a wallet.*

---

## Table of Contents

1. [Background](#1-background)
2. [What Data You Need](#2-what-data-you-need)
3. [Baseline Heuristic](#3-baseline-heuristic)
4. [Parsing Transactions – Step by Step](#4-parsing-transactions--step-by-step)
5. [Filtering Out Noise](#5-filtering-out-noise)
6. [Classifying the Finds](#6-classifying-the-finds)
7. [Reference Implementation (TypeScript, Node ≥ 20)](#7-reference-implementation-typescript-node≥20)
8. [Performance Tips](#8-performance-tips)
9. [Limitations & Edge Cases](#9-limitations--edge-cases)
10. [Further Reading](#10-further-reading)

---

## 1  Background

A **Program‑Derived Address (PDA)** is an on‑chain account whose address is deterministically generated from *seeds* plus a program ID.  No private key exists; the owning program signs for it via `invoke_signed`.  Detecting PDA creation tied to a wallet lets you:

- Reconstruct a user’s on‑chain state (positions, vaults, escrow accounts).
- Trigger off‑chain indexing or alerts when new state appears.
- Audit protocols for hidden resource consumption.

Unlike ordinary accounts, PDAs are *created* at runtime—usually within **inner instructions**—so you can’t just watch SystemProgram `CreateAccount` calls at the top level.

---

## 2  What Data You Need

- **All transactions a wallet signed.**   `getSignaturesForAddress(wallet)` paginates sigs.
- **Full parsed transactions.**  `getTransaction(sig, { maxSupportedTransactionVersion: 0, jsonParsed: true })` so you get balances and inner instructions.
- **(Optional)** `getAccountInfo` one slot before a tx for rock‑solid existence checks.

---

## 3  Baseline Heuristic

> **If an account’s **``**, the account *****did not exist***** until this transaction.**

This rule catches PDAs, freshly minted ATAs, ordinary throw‑away System accounts—*everything*.  We’ll narrow it down in the next steps.

---

## 4  Parsing Transactions – Step by Step

1. **Enumerate all account keys** for the message **plus** address‑lookup tables (ALT).
2. **Scan top‑level ****\`\`****.**  Flag where `pre == 0  &&  post > 0`.
3. **Walk inner instructions** (CPI):
   - Look for `SystemProgram::CreateAccount`, `CreateAccountWithSeed`, `Allocate`, `AllocateWithSeed`.
   - The \*\*first \*\*\`\` in those instructions is the newly allocated address.
4. **Merge the two sets** → candidate list of "accounts born in this tx".

Why bother with both?  A PDA often shows up only in **inner instructions**; ATAs and plain System accounts sometimes appear only via the balance heuristic.

---

## 5  Filtering Out Noise

You usually care about *external* PDAs (owned by protocols), not boring self‑owned stuff.

| Filter                                                     | Rationale                                   |
| ---------------------------------------------------------- | ------------------------------------------- |
| `candidate == wallet`                                      | It’s just the wallet itself.                |
| `owner == TOKEN_PROGRAM && candidate == ATA(wallet, mint)` | Canonical ATA—fine to ignore.               |
| `owner == SystemProgram.programId`                         | Plain System account; unlikely to be a PDA. |

Everything left is likely a PDA or some other program‑owned auxiliary account.

---

## 6  Classifying the Finds

1. \*\*Fetch the account’s \*\*\`\`.  This is the program that created it.
2. **Match against known PDA patterns** (see *Solana PDA Derivation Patterns* doc):

```typescript
import { derivePda } from "./derivePda"; // from earlier guide

const [expected] = derivePda([
    Buffer.from("user"), wallet.toBuffer(),
], KNOWN_PROGRAM_ID);

if (expected.equals(candidate)) console.log("It's that program's user PDA");
```

3. **Unknown?**  Tag it `"unknown PDA from <programId>"` and store for later analysis.

---

## 7  Reference Implementation (TypeScript, Node ≥ 20)

```typescript
import { Connection, PublicKey, SystemProgram } from "@solana/web3.js";

/** Return PDAs (and other new program‑owned accounts) created in txs the wallet signed. */
export async function detectPdaCreations(
    conn: Connection,
    wallet: PublicKey,
    limit = 1000, // number of signatures to crawl
): Promise<PublicKey[]> {
    const sigs = await conn.getSignaturesForAddress(wallet, { limit });
    const created: Set<string> = new Set();

    for (const { signature } of sigs) {
        const tx = await conn.getTransaction(signature, { maxSupportedTransactionVersion: 0 });
        if (!tx?.meta) continue;

        const keys = tx.transaction.message.getAccountKeys({
            accountKeysFromLookups: tx.meta.loadedAddresses,
        });

        // 1 — balance heuristic
        keys.forEach((key, i) => {
            if (tx.meta!.preBalances[i] === 0n && tx.meta!.postBalances[i] > 0n) {
                created.add(key.toBase58());
            }
        });

        // 2 — system create in inner instructions
        tx.meta.innerInstructions?.forEach(ixs => {
            ixs.instructions.forEach(ix => {
                if (keys[ix.programIdIndex].equals(SystemProgram.programId)) {
                    const newAddr = keys[ix.accounts[0]].toBase58();
                    created.add(newAddr);
                }
            });
        });
    }

    // 3 — filter
    const out: PublicKey[] = [];
    for (const b58 of created) {
        const pub = new PublicKey(b58);
        if (pub.equals(wallet)) continue; // skip self

        const info = await conn.getAccountInfo(pub, "confirmed");
        if (!info) continue; // shouldn’t happen

        if (info.owner.equals(SystemProgram.programId)) continue; // plain system account
        // TODO: filter ATAs here if you import the ATA derivation helper

        out.push(pub);
    }

    return out;
}
```

*Conventions:* 4‑space indent, `import`, async/await, no `try/catch`.

---

## 8  Performance Tips

- Paginate sigs in **reverse‑chronological order** so you can early‑exit once you hit a date boundary.
- Parallelise `getTransaction` calls (but stay within your RPC rate limits).
- Cache `getAccountInfo` results; ownership rarely changes.
- If you run your own RPC, use the \`\` plugin for bulk export instead of paginated RPC.

---

## 9  Limitations & Edge Cases

- **Factories**: some programs batch‑create PDAs for many users in a single tx; the wallet may not be the fee‑payer, so you must also scan *any* tx where the wallet appears as an account, not just signer.
- **Reallocs**: `SystemProgram::Allocate` can upgrade an existing account’s size; doesn’t violate the balance heuristic.
- **Rent‑exempt zero**: In rare cases an account is funded by another tx then allocated; preBalance may be >0 even though the account had no data.
- **Closed accounts**: De‑allocations (`SystemProgram::Assign` + zero lamports) aren’t covered here; track `postBalance == 0` if you need deletions.

---

## 10  Further Reading

- [Solana Docs — Account Lifecycle](https://docs.solana.com/developing/programming-model/accounts#account-lifecycle)
- [Solana Cookbook — getSignaturesForAddress](https://solanacookbook.com/references/transactions.html#getsig)
- [PDA Derivation Patterns Reference](./Solana%20Pda%20Derivation%20Patterns%20Extended)
