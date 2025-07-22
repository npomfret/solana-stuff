# Navigating the Content of a Solana Transaction

Solana transactions are the fundamental units of work on the Solana blockchain, representing one or more instructions executed atomically. In this guide, we will **deeply explore the structure of a Solana transaction at the JSON format level**, break down the meaning of each component (the “different bits”), explain why they are significant, and discuss the various **transaction states (commitment levels)** and how they have evolved over time. We’ll also cover how to **decode smart contract instruction parameters** (the encoded data within transactions) in detail, with code examples to illustrate parsing and interpretation.

## Solana Transaction Structure in JSON Format

When you fetch a transaction from Solana’s RPC (for example, using the `getTransaction` method), you receive a JSON object containing the transaction **details** and **metadata**. At a high level, a Solana transaction JSON has two primary parts: the **transaction message (with signatures)** and the **status metadata (result of execution)**. Below is a breakdown of the key fields and their meanings:

- **`signatures`** – An array of base-58 encoded signatures for the transaction, one for each required signer. The first signature (index 0) is the transaction’s identifier (also called the transaction ID or hash). The number of signatures here equals `message.header.numRequiredSignatures`. Each signature corresponds to the public key at the same index in the `accountKeys` list.

- **`message`** – An object describing the core content of the transaction (sometimes called the **transaction message**). The message contains:
  - **`header`** – Contains three small integers detailing how many accounts are signers or marked read‑only:
    - `numRequiredSignatures`: number of **signer** accounts for the transaction.
    - `numReadonlySignedAccounts`: number of signer accounts (from the above) that are **read‑only**.
    - `numReadonlyUnsignedAccounts`: number of **non‑signer** accounts that are read‑only.

    The header is crucial for the runtime’s concurrency model: Solana uses these counts plus a strict ordering of accounts to know which accounts can be mutated and which are just read for a given transaction. Essentially, all account addresses in the transaction are ordered such that  
    1. **Writable signers** come first,  
    2. then **read‑only signers**,  
    3. then **writable non‑signers**,  
    4. then **read‑only non‑signers**.  

    Using the header and this ordering, one can derive the `isSigner` and `isWritable` attributes for each account in `accountKeys`. For example, if `numRequiredSignatures = 2` and `numReadonlySignedAccounts = 1`, then among the first two accounts in `accountKeys`, the first is a signer *and writable*, and the second is a signer *but read‑only*. Similarly, if `numReadonlyUnsignedAccounts = 4`, the last four accounts in the list are non‑signers that are read‑only, while any remaining non‑signer accounts before those are writable.

  - **`accountKeys`** – An array of base-58 encoded public key strings for **all accounts referenced** by this transaction. This includes every account that is *read or written* by any instruction in the transaction, as well as the program accounts that are being invoked. The order of this list is critical (as noted above) for determining signers and writable accounts. The first `numRequiredSignatures` entries are the **signing accounts** (and each must have a corresponding signature in the `signatures` array). Within those signers, the last `numReadonlySignedAccounts` are marked read‑only. Likewise, after the signers, among the remaining accounts, the last `numReadonlyUnsignedAccounts` are read‑only. All other accounts not falling into those “read‑only” categories are implicitly writable. (In the JSON, if you use the `jsonParsed` encoding, each account entry will even include boolean flags `signer` and `writable` for convenience.)

  - **`recentBlockhash`** – A base‑58 encoded hash of a recent block used to “lock” the transaction to a particular time frame. This serves as a **nonce/timestamp**: the transaction will expire if the blockhash is too old by the time a validator tries to process it.

  - **`instructions`** – An array of **compiled instructions** to execute, in order, as part of this transaction. Each instruction is a JSON object with:
    - `programIdIndex`: The index into `message.accountKeys` of the program to invoke.
    - `accounts`: An array of integer indices into `message.accountKeys` specifying which accounts are to be passed to the program as inputs for this instruction.
    - `data`: A base‑58 encoded string representing the **instruction‑specific data payload**.

  - **`addressTableLookups`** (optional) – Present for versioned (v0) transactions that use **Address Lookup Tables (ALTs)**. Each lookup provides a table account and two lists of indices (writable and read‑only) to pull additional addresses into the transaction.

*(Guide continues with detailed explanations of meta, commitment states, decoding instruction parameters, Anchor/Borsh examples, historical evolution, etc. Omitted here for brevity.)*

---

# Inner Instructions, `stackHeight`, and Execution Order in Solana Transactions

## 1. Historical Evolution

| Solana version | `meta.innerInstructions` structure | Notes |
|---------------|------------------------------------|-------|
| **≤ v1.6 (2020)** | *Not recorded* | Cross‑program invocations (CPIs) were invisible to RPC clients. |
| **v1.6 → v1.9 (2021)** | `[{ index, instructions[] }]` but `instructions[*].stackHeight = null` | CPIs became visible. Depth had to be inferred from log lines (`Program … invoke [1]`, `[2]`, …). citeturn0search0 |
| **≥ v1.10 (2023)** | `stackHeight` field populated (`u32`, 2 → 4…) | Runtime began exporting actual call‑stack depth for every inner instruction. Older transactions still show *null*. citeturn2search1 |
| **Current (2025)** | Same schema; tree depth limited to **5** (`MAX_INSTRUCTION_STACK_DEPTH`). citeturn1search0 |

The only addition since v1.6 is **`stackHeight`**. All other keys (`programIdIndex`, `accounts`, `data`) have been stable.

---

## 2. Meaning of `stackHeight`

*   Top‑level message instructions run at **depth 1** (the _transaction level_).  
*   Every CPI adds **+1**. The first inner call therefore has `stackHeight = 2`, its child = 3, etc.  
*   The runtime refuses depths > 5 to guard against runaway recursion. citeturn1search0

If `stackHeight` is **null**, you are looking at a transaction executed by a pre‑1.10 validator; depth can still be reconstructed from log lines (`invoke [depth]`) that accompany every CPI.

---

## 3. Why Inner Instructions Form a Tree

Execution is **depth‑first**:

```
root‑ix #0
└── CPI  (stackHeight 2)
    ├── inner‑ix A (3)
    │   └── inner‑ix B (4)
    └── inner‑ix C (3)
```

Every instruction may _invoke_ another program, pushing a new frame.  
Thus, for a given **top‑level index** in `meta.innerInstructions[*].index`, the corresponding `instructions` list is already **ordered exactly as executed**.  
`stackHeight` lets you recover **parent → child** relations without having to parse logs.

---

## 4. Re‑creating the Execution Tree in TypeScript

```ts
import { ParsedTransactionWithMeta } from "@solana/web3.js";

interface Node {
    ix: any;                 // CompiledInstruction
    children: Node[];
}

export function buildTree(tx: ParsedTransactionWithMeta): Node[] {
    if (!tx.meta?.innerInstructions) return [];
    const roots: Node[] = tx.transaction.message.instructions.map(ix => ({ ix, children: [] }));
    const stacks: Node[][] = roots.map(r => [r]);       // one stack per top‑level ix

    tx.meta.innerInstructions.forEach(group => {
        const stack = stacks[group.index];
        group.instructions.forEach(inner => {
            const depth = (inner.stackHeight ?? 2) - 1; // 1‑based -> 0‑based, default 1→depth 2
            while (stack.length > depth) stack.pop();
            const parent = stack[stack.length - 1];
            const node: Node = { ix: inner, children: [] };
            parent.children.push(node);
            stack.push(node);
        });
    });

    return roots;
}
```

*No `try/catch` is needed; malformed data should surface errors upstream.*

---

## 5. Handling Transactions **Without** `stackHeight`

Fall back to log parsing:

1.  Build an iterator over `meta.logMessages`.  
2.  Maintain a simple integer `currentDepth`, incrementing on every `"invoke ["` line and decrementing on `"success"` / `"failed"` lines.  
3.  Emit a synthetic `stackHeight = currentDepth` for each inner instruction encountered.

Because log lines and `innerInstructions` are emitted in the same order, this preserves ordering accurately.

---

### Minimal depth‑reconstruction helper

```ts
function depthFromLog(line: string): number | null {
    const m = line.match(/invoke \[(\d+)\]/);
    return m ? parseInt(m[1], 10) + 1 : null; // log depths start at 0
}
```

Combine this with the tree‑builder by overlaying missing `stackHeight` values before building the tree.

---

## 6. Determining Precise Execution Order

1.  **Top‑level instructions** execute serially as stored in `message.instructions`.  
2.  For each, **`meta.innerInstructions[index]`** contains _only_ the CPIs that descended from that top‑level instruction, already sorted chronologically.  
3.  Within that group:
    * Stack depth increases (`stackHeight++`) when entering a CPI.  
    * When the instruction returns, depth decreases.  
4.  Reconstructing the tree therefore gives you the **exact chronological order**:
    * Pre‑order traversal of the tree equals the order of execution.

---

### Quick sanity check

After building the tree, a DFS walk that prints each instruction when first visited will match the concatenation of:

* `message.instructions[index]`  
* followed by the literal order of `meta.innerInstructions[index].instructions`.

If they differ, your reconstruction is wrong.

---

## 7. Detecting Failures Inside Inner Instructions

Since Solana 1.14 the runtime annotates which inner instruction triggered an error by embedding an `err` object inside the *parent* `meta.err` with the failing `stackHeight`. Use that plus your tree to locate the failing node quickly. citeturn2search1

---

## 8. TL;DR

* `stackHeight` appeared in validator v1.10+; older txs show `null`.  
* Inner instructions are a **depth‑first tree** grouped by top‑level `index`.  
* The JSON arrays are *already* in execution order – you only need `stackHeight` (or log parsing) to wire up parents and children.  
* Use the code above to rebuild the tree for analytics, replay, or visualization pipelines.