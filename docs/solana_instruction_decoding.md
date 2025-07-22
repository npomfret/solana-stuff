# Decoding Solana Smart Contract Calls (Transactions) in Detail

## Using IDLs and Anchor for Modern Contracts  
Modern Solana programs often use **Anchor**, which provides an *Interface Definition Language (IDL)* describing instructions and accounts. If an IDL is available (via the project’s repo or `anchor idl fetch`), decoding becomes straightforward. For example, given an IDL JSON and Anchor’s TypeScript library, you can use Anchor’s `BorshInstructionCoder` to parse instruction data into human-readable form.

```ts
import { BorshInstructionCoder, Idl } from "@coral-xyz/anchor";
import bs58 from "bs58";

const idl = require("./idl/my_program.json") as Idl;
const coder = new BorshInstructionCoder(idl);

const ixDataBase58 = "368Yh7EEPvKQfQdCFA3GKqgk";
const decoded = coder.decode(ixDataBase58, "base58");
console.log(`Instruction name: ${decoded.name}`, decoded.data);
```

Behind the scenes, Anchor prepends an 8‑byte discriminator (hash of the method name) to `ix.data`. The coder maps the rest of the bytes to the fields defined in the IDL. Anchor’s IDL also gives the required accounts, so you can interpret the `accounts` array too.

## Decoding Legacy and Non‑IDL Programs (Serum, Raydium, etc.)
Older programs weren’t built with Anchor and lack public IDLs, so you must decode manually.

### 1. Use Source Code
Many projects eventually open‑sourced. Serum’s Rust enum `DexInstruction` shows variant tags and struct layouts. Write a Borsh schema that mirrors those structs and decode with `borsh`.

### 2. Scrape Client SDKs / UI
If the Rust isn’t public, inspect the project’s TS SDK or front‑end. The JS that *creates* the instruction also reveals its byte layout. Solscan originally did this for Raydium before the IDL was published.

### Example Manual Decoder
```ts
import * as borsh from "borsh";
import bs58 from "bs58";

class BuyV2Args {
  constructor(fields: { buyerPrice: bigint; tokenSize: bigint; buyerStateExpiry: bigint; buyerCreatorRoyaltyBp: number; extraArgs: Uint8Array }) {
    Object.assign(this, fields);
  }
}
const schema = new Map([
  [
    BuyV2Args,
    {
      kind: "struct",
      fields: [
        ["discriminator", "u64"],
        ["buyerPrice", "u64"],
        ["tokenSize", "u64"],
        ["buyerStateExpiry", "i64"],
        ["buyerCreatorRoyaltyBp", "u16"],
        ["extraArgs", ["u8"]],
      ],
    },
  ],
]);

const raw = bs58.decode("UEVYJrzmev2UQ1cVQ8mBufZjnj8LT83s1ukeQvg1kERm5M7F3oaX");
const decoded = borsh.deserialize(schema, BuyV2Args, raw);
console.log(decoded);
```

### 3. Register Custom Parsers
Create a switch‑based parser for the program ID and register it in your transaction‑processing pipeline. Libraries like `solana-tx-parser` let you plug such handlers.

### 4. Reverse‑Engineer the Binary
Dump the program (`solana program dump <ID> out.so`) and disassemble. For Anchor binaries, search for the string `Instruction:` to recover method names; Sec3’s *IDL‑Guesser* automates this. For non‑Anchor, inspect how the code reads `instruction_data` – there’s usually a tag byte followed by packed fields.

## Decision Flow for a Universal Decoder

1. **Does an IDL exist?**  
   *Yes* ➜ load and use Anchor’s coder.  
2. **Is the program a well‑known legacy app?**  
   *Yes* ➜ use a pre‑written custom parser (Serum, Raydium, Mango…).  
3. **No source, no IDL?**  
   ➜ Disassemble, analyze logs, and manually infer a schema; optionally run IDL‑Guesser for Anchor patterns.

By combining these techniques you can decode every Solana transaction from genesis onward, feeding clean, structured data into your accounting system.
