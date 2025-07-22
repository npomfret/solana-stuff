
# Solana Transaction JSON Guide

## Overview
This document explains **every part** of a Solana transaction object returned by RPC (`getTransaction`) and how to decode smart‑contract parameters and inner instruction call trees.  
Target audience: developers writing analytics or monitoring code in modern TypeScript (Node ≥ 20).

---

## 1  Transaction JSON Anatomy

### 1.1 `signatures`
* Array of base‑58 signatures.
* Element 0 doubles as the transaction ID.
* Count equals `message.header.numRequiredSignatures`.

### 1.2 `message.header`
| field | description |
|-------|-------------|
| `numRequiredSignatures` | signer count (first *N* entries in `accountKeys`) |
| `numReadonlySignedAccounts` | last *k* of the signers are **read‑only** |
| `numReadonlyUnsignedAccounts` | last *m* of the non‑signers are **read‑only** |

The ordering lets the runtime infer `isSigner` + `isWritable` with no per‑account flags.

### 1.3 `accountKeys`
* Ordered list of **all** accounts touched (signers first).  
* Indexing is used everywhere else (program IDs, instruction account lists).  
* Writable/non‑writable is derived from header counts.

### 1.4 `recentBlockhash`
* 32‑byte block hash.  
* Expires after ≈150 blocks (~60‑90 s). Prevents replay.

### 1.5 `instructions`
Each instruction object:

```json
{
  "programIdIndex": 8,
  "accounts": [0, 3, 5],
  "data": "3Bxs4g=="        // base‑58 or base‑64
}
```

* The program ID and account pubkeys are looked up by **index** inside `accountKeys`.
* `data` is program‑specific binary (often Borsh).

### 1.6 `addressTableLookups` (v0 only)
Loads extra addresses from on‑chain lookup tables so a transaction can reference more accounts than fit in 1232 B.

---

## 2  Execution Metadata — `meta`

| field | meaning |
|-------|---------|
| `err` | `null` ⇒ success, else error object |
| `fee` | lamports charged |
| `preBalances` / `postBalances` | lamport balances per account |
| `preTokenBalances` / `postTokenBalances` | SPL‑Token balances, if enabled |
| `logMessages` | program logs |
| `innerInstructions` | CPIs grouped by parent instruction (`index`) |
| `rewards` | staking / rent deltas |
| `loadedAddresses` | addresses pulled from lookup tables |
| `returnData` | last `sol_set_return_data` payload |
| `computeUnitsConsumed` | actual compute used |

`innerInstructions` has existed **since launch of CPI recording (v1.6, mid‑2021)**.  
Older txs just show `innerInstructions:null`.

---

## 3  Commitment Levels

| level | guarantee | typical latency |
|-------|-----------|-----------------|
| **Processed** | in leader’s block, not yet voted | ~1–2 s |
| **Confirmed** | block voted by ≥66 % stake | ~5–8 s |
| **Finalized** | rooted (≥31 descendant slots) | ~15–20 s |

No main‑net transaction that reached **Confirmed** has ever been rolled back.

---

## 4  Decoding Instruction Data

### 4.1 Identify the program
```ts
const ix = tx.transaction.message.instructions[i];
const programPk = tx.transaction.message.accountKeys[ix.programIdIndex];
```

### 4.2 Decode the bytes
```ts
import bs58 from "bs58";
const bytes = bs58.decode(ix.data);
```

### 4.3 Interpret
* **SPL Token example** (variant 3 = `Transfer`, next 8 bytes = amount):

```ts
if (bytes[0] === 3) {
    const amount = Number(BigInt("0x" + Buffer.from(bytes.slice(1, 9)).reverse().toString("hex")));
}
```

* **Anchor program** with IDL:

```ts
import { BorshCoder } from "@project-serum/anchor";
const coder = new BorshCoder(idl);
const decoded = coder.instruction.decode(ix.data, "base58");
```

---

## 5  Rebuilding the Inner Instruction Call‑Tree

### 5.1 Concept
* Outer instructions run **in array order**.
* At runtime each instruction may perform CPIs (depth‑first).  
  Recorder outputs:

```json
"innerInstructions": [
  { "index": 1, "instructions": [...] },
  { "index": 3, "instructions": [...] }
]
```

* All CPIs from outer `1` appear, then CPIs from outer `3`, etc.  
* In `jsonParsed` mode every instruction also carries `stackHeight` (0 = outer).

### 5.2 Helper (modern TypeScript)

```ts
export interface IxNode {
    ix: PartiallyDecodedInstruction | ParsedInstruction;
    children: IxNode[];
}

export function buildIxTree(tx: ParsedTransactionWithMeta): IxNode[] {
    if (!tx.meta) throw new Error("meta missing");
    const outer = tx.transaction.message.instructions;
    const roots = outer.map(ix => ({ ix, children: [] }));

    tx.meta.innerInstructions?.forEach(g => {
        const parent = roots[g.index];
        g.instructions.forEach(inner => parent.children.push({ ix: inner, children: [] }));
    });

    // optional true nesting via stackHeight
    const all = roots.flatMap(r => [r, ...r.children]);
    const byH = new Map<number, IxNode[]>();
    all.forEach(n => {
        const h = (n.ix as any).stackHeight ?? 0;
        byH.set(h, [...(byH.get(h) ?? []), n]);
    });
    for (let h = 1; byH.has(h); h++) {
        byH.get(h)!.forEach(n => {
            const parent = byH.get(h - 1)!.at(-1)!;
            parent.children.push(n);
        });
    }
    return roots;
}
```

`roots` holds outer instructions in true execution order; depth‑first traversal yields the exact timeline.

---

## 6  Historical Changes

| era | change |
|-----|--------|
| 2020 | only legacy tx; no `innerInstructions`, no token balances |
| 2021‑07 (v1.6) | CPI recording added ⇒ `innerInstructions` + `logMessages` |
| 2021‑11 | `pre/postTokenBalances` live |
| 2022‑07 | Transaction **v0** + `addressTableLookups`, `loadedAddresses`, `version` |
| 2023 | `returnData`, `computeUnitsConsumed` recorded |
| 2024 | RPC clean‑up: `getConfirmedTransaction` removed; always use `getTransaction` |
| 2025 | Nothing new—format stable |

---

## 7  Example: End‑to‑End Parser

```ts
import { Connection, clusterApiUrl } from "@solana/web3.js";
import { decodeTransferInstruction, TOKEN_PROGRAM_ID } from "@solana/spl-token";
import bs58 from "bs58";

const conn = new Connection(clusterApiUrl("mainnet-beta"), "confirmed");
const sig = "5V7k...";  // tx signature

(async () => {
    const tx = await conn.getTransaction(sig, { commitment: "confirmed", maxSupportedTransactionVersion: 0 });
    if (!tx) throw new Error("tx not found");

    tx.transaction.message.instructions.forEach((ix, i) => {
        const programPk = tx.transaction.message.accountKeys[ix.programIdIndex];
        if (programPk.equals(TOKEN_PROGRAM_ID)) {
            const decoded = decodeTransferInstruction({
                programId: programPk,
                keys: ix.accounts,
                data: bs58.decode(ix.data)
            });
            console.log(`token transfer: ${decoded.data.amount}`);
        }
    });

    const tree = buildIxTree(tx);
    console.dir(tree, { depth: null });
})();
```

---

## 8  Resources
* Solana Docs – Transactions & Instructions
* SPL Token Program spec
* Anchor Book – Instruction decoding
* Solana bigtable / explorer for raw JSON examples
