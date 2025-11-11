# Solana Account Creation: A Comprehensive Guide

## Understanding Solana Accounts

Solana accounts are the fundamental data storage units on-chain. Every account has a few core properties in common: **lamports** (the account's balance in SOL's smallest unit), **data** (the on-chain stored bytes for programs or tokens), **owner** (the program that "owns" and can modify the account), an **executable** flag (true if the account holds a program's code), and a deprecated **rent_epoch** field.

New accounts must be funded with a minimum balance (in lamports) to cover "rent" for their data size, though if funded above a certain threshold they become rent-exempt (the lamports act as a one-time deposit that can be recovered on account closure). Only the designated owner program can modify an account's data or debit its lamports. Importantly, **only Solana's native System Program can create new accounts** – all other programs must delegate to the System Program to allocate accounts for them.

When an account is first created, its fields are initialized as follows: the **lamports** field is set to the funding amount (often the rent-exemption minimum), **owner** is set to the specified program (or remains the System Program for system-owned accounts), **data** length is allocated to the requested size (with content zero-initialized unless the creating program immediately overwrites it), and **executable** is usually `false` (except when loading a program's code). For example, a newly created token account might show a balance of rent-exempt lamports, `owner` equal to the Token Program's ID, `data` length of 165 bytes (the size of a token account state), and `executable: false`. An account created to deploy a program would have `owner` set to a BPF loader program and `executable: true` after deployment.

### Note on system-owned accounts

By default, any account created by the System Program starts out owned by the System Program itself, unless immediately assigned to another owner as part of creation. Accounts owned by the System Program are often just SOL-holding wallets. In fact, if you send lamports (SOL) to a new address that has no prior account, the runtime will automatically create a system-owned account for that address. This means **transferring SOL to a new public key implicitly creates a System Account** at that address, funded with the transferred lamports. In transaction logs or JSON, you would see the System Program's Transfer instruction and the new account's balance appear post-transaction (with `owner: 11111111111111111111111111111111` – the System Program). However, to create accounts with custom owners or data space, one of the explicit techniques below is used.

## 1. System Program `createAccount` Instruction

The primary way to create a new account is by invoking the System Program's **Create Account** instruction. This method is used for most account types on Solana. The `createAccount` instruction takes parameters for: the funding payer, the public key of the new account, the amount of lamports to transfer into the new account, the space (number of bytes to allocate for the account's data), and the owner program id that will own the new account. The System Program will deduct the lamports from the payer, allocate the account with the given space, and assign its ownership to the specified program.

In a raw transaction's JSON, a System Program create account instruction is identifiable by `programId: "11111111111111111111111111111111"` (the System Program's address) and the instruction data corresponding to a "Create" command. If you use the RPC with JSON parsing, it will explicitly show something like `"type": "createAccount"` along with fields such as `"lamports"`, `"newAccount"`, `"owner"`, and `"space"` under the parsed instruction info. For example, a parsed transaction might include:

```json
{
  "program": "system",
  "parsed": {
    "type": "createAccount",
    "info": {
      "source": "<payer_pubkey>",
      "newAccount": "<new_account_pubkey>",
      "lamports": <amount>,
      "space": <bytes>,
      "owner": "<owner_program_id>"
    }
  }
}
```

This indicates the payer funded a new account with the given lamports, allocating the specified space and assigning the owner. After this instruction executes, the new account's entry will exist on-chain with the chosen owner and data size. In the transaction's accountKeys list, the new account's public key will be present (typically marked as writable and not signer, since it had no private key sign-off). The transaction's `preBalances` for that account will be 0, and `postBalances` will show the lamports deposited, confirming the account was created and funded in that transaction.

### Usage examples

This method is used whenever a program or client needs to create a new account for storing data – for instance, creating a new user data account for a dApp's state, initializing a token mint or token account (alongside the Token Program), or setting up any account for another on-chain program's use. Developers often call `SystemProgram.createAccount(...)` in Solana SDKs or CPI (cross-program invocation) to accomplish this. On-chain, it always results in a System Program invocation as seen in the transaction details.

## 2. System Program `createAccountWithSeed` (Deterministic Addresses)

Solana also provides a variant: **Create Account With Seed**, which allows deriving the new account's address deterministically from an existing base public key and a seed string. This is useful when you want a predictable address for the account (for example, deriving accounts for a known purpose or splitting stake to a derived address). The instruction takes a base account (which must sign the transaction), a UTF-8 seed string, the new account's pubkey (computed from base+seed+owner), the lamports, space, and owner program. The System Program will create the account at the derived address, provided no account already exists there. Under the hood, `createAccountWithSeed` is essentially doing an Allocate (reserve space) and Assign (set owner) with a seeded address, combined with funding – it's conceptually AllocateWithSeed + AssignWithSeed in one step.

In transaction JSON, a parsed instruction might show `"type": "createAccountWithSeed"` with an info object containing the base public key and seed in addition to the usual lamports/space/owner fields (if using the `jsonParsed` encoding). If not using parsed output, you can still recognize it by the System Program programId and the encoded data (the first byte of System instructions denotes the instruction: 0 = CreateAccount, 1 = Assign, 2 = Transfer, 3 = CreateAccountWithSeed, etc., according to the system_instruction enum). The base account will appear as one of the accounts in the instruction (usually the first after the payer).

### Usage examples

The CLI's `solana-keygen new --seed <seed>` essentially uses this to derive a keypair, and `solana create-stake-account --seed ...` uses it to create a stake account at a derived address. Similarly, you can derive a vote account address from a validator identity using a seed. However, in modern practice, developers often prefer Program Derived Addresses (PDAs, see next) for deterministic accounts on behalf of programs. The `createAccountWithSeed` is still useful for user-driven deterministic accounts (since PDAs are for program-derived keys that don't have a private key).

## 3. Program Derived Addresses (PDAs) and Programmatic Account Creation (CPI)

**Program Derived Addresses (PDAs)** are special addresses that have no associated private key, but are deterministically derived from a program's ID and some seeds. They are typically used by on-chain programs to create and manage accounts without a user-provided keypair. Because only the owning program can "sign" for a PDA (using a bump seed mechanism), PDAs are a secure way for programs to manage their own accounts.

When a program needs to create an account (for example, to store its state or an NFT metadata account), it will perform a **Cross-Program Invocation** to the System Program's create account instruction from within the program's execution. This is often facilitated by calling `invoke_signed` in the program's Rust code, passing the PDA seeds so the System Program will accept the PDA as the "new account" signer. The System Program then creates the account as usual, deducting lamports from a provided payer account (often the user) and assigning ownership to the program. The newly created account's address is the PDA.

From a transaction perspective, account creation via CPI may not show up as a top-level instruction to the System Program. Instead, you'll see an instruction invoking the program that creates the PDA. The actual `SystemProgram::CreateAccount` will appear as an **inner instruction** in the transaction's meta. For instance, when using Anchor framework with an `#[account(init, seeds=..., bump, payer=user)]`, the transaction's outer instruction is a call to the Anchor program, and inside it the runtime logs will show `Program 11111111111111111111111111111111 invoke […]` and a success when the system program is called. If you fetch the transaction with `getTransaction` including inner instructions, you will see an inner entry with `"program": "system", "parsed": { "type": "createAccount", ... }` indicating the program created a PDA. The logs also reflect this: e.g. `"Program <ProgramID> invoke [1]"` followed by `"Program 11111111111111111111111111111111 invoke [2]"… "Program 11111111111111111111111111111111 success"` shows the System Program was called within another program's instruction.

### What to look for

If analyzing raw JSON, check `meta.innerInstructions`. A common pattern is an inner createAccount (system) paired with some follow-up initialization by the program. Also, the new PDA will typically be listed in the transaction's accountKeys with `signer: false` and `writer: true`. Its preBalance will be 0 and postBalance > 0 (funded by the payer). The `owner` in account info will be the program's ID after creation.

### Usage examples

Most custom programs use PDAs for their accounts. For example, a Metaplex NFT Metadata account is a PDA derived from the mint address and the metadata program's ID – the metadata program CPIs to system create that PDA. Similarly, an Anchor program uses PDAs for any `init` accounts. The benefit is that the address can be derived by clients and the program, without needing the client to supply a new keypair. PDAs are recommended for on-chain derived accounts over using `createAccountWithSeed` from the client side, since the program can enforce access control for creation and use the seeds internally.

## 4. Associated Token Account Creation (ATA Program)

**Associated Token Accounts** are a specific type of account used to hold SPL tokens. Each wallet address can have at most one associated token account for each mint (token type), at a deterministic PDA address (derived from the wallet, the token mint, and the Associated Token program's ID). To simplify token account management, Solana provides the **Associated Token Account Program** (program ID: `ATokenGPvbd...knL`) which handles creating these accounts.

When you create an associated token account (often via SDKs or the `spl-token` CLI), you actually invoke the Associated Token Account program with an instruction to create the account. That program internally performs two main actions: it calls the System Program to create a new account (the ATA) and assigns it to the Token Program, and then calls the Token Program to initialize the account as an SPL Token account. All of this happens within one high-level instruction.

In a transaction's JSON, the outer instruction will have programId equal to the ATA program (`ATokenGPvbdGVxr...`, a constant address) and typically no significant input data (the program derives everything from the accounts). The accounts list for this instruction usually includes: the funder (payer), the new associated token account address (PDA derived), the wallet address (owner of the token account), the token mint, the System Program, the Token Program, and the Rent sysvar. Because the ATA program uses CPI, you will see inner instructions for the actual creation. For example, an inner instruction with `"type": "createAccount"` (System Program) where `"newAccount"` is the associated token account address and `"owner": "Tokenkeg…"` (the SPL Token program), followed by an inner `"type": "initializeAccount"` (or `initializeAccount3`) under the SPL Token program. The transaction log snippet below illustrates this:

- **Inner Instruction 0**: System Program createAccount – creates the ATA with 165 bytes space, owner set to Token Program, funded by the payer.
- **Inner Instruction 1**: SPL Token Program InitializeAccount – initializes the new token account, associating it with the given mint and owner.
- **Log messages**: `"Program AToken... invoke [1]"` then `"Program 11111111111111111111111111111111 invoke [2] … success"` (the system program call) and `"Program ...Tokenkeg... invoke [2] … Instruction: InitializeAccount3 … success"`, concluding with `"Program AToken... success"`.

After a successful ATA creation, the new account is fully ready to hold tokens. In the accounts list, it will appear with `owner: Tokenkeg...` (Token program ID) if you fetch its account info. The `postTokenBalances` section of the transaction may also list this account (with amount 0 initially) as a new token account.

### Usage

Almost every token wallet uses associated token accounts. When a user receives a new token for the first time, an ATA is often created (sometimes implicitly by the sender). Wallet interfaces or dApps will call this program to create the account so that tokens can be deposited. As an engineer parsing transactions, if you see a call to the ATA program, you can infer a new token holding account was being created. The key indicator is the ATA program ID in the instruction, and inner instructions doing a system create (owner = Token program).

## 5. Manual Creation of SPL Token Accounts (Token Program)

Before the ATA program existed (or in advanced scenarios like creating token accounts not tied to a specific user wallet), developers can create SPL token accounts manually with a two-step process. In this approach, the client explicitly issues a System Program `createAccount` to make a new account owned by the Token Program, then calls the Token Program to initialize it (declare what mint it's for and who is the owner of this token account). The pattern is similar for creating a **token mint account** or a **token multisig account** – only the initialization instruction differs.

### Token Account (for holding tokens)

To do this manually, one would generate a new keypair for the account (or pick a PDA for it), then in a transaction:

1. **Instruction 1**: `SystemProgram::CreateAccount` with `owner = TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` (the SPL Token program's ID), and `space = 165` bytes (the size of token Account data). The lamports should be the rent-exemption amount for 165 bytes. This creates an uninitialized token account.
2. **Instruction 2**: `TokenProgram::InitializeAccount` (or the newer `InitializeAccount2/3` which allow specifying owner/mint in the instruction data). This takes the new account's pubkey, the token **mint** pubkey, and the account's **owner** (the wallet address who will control the tokens). After this, the account's data is set to the appropriate state (it now stores the mint, owner, amount = 0, etc.).

In the transaction JSON, these appear as two separate top-level instructions. The first is a System `createAccount` (with `"owner": "Tokenkeg..."` in parsed form, indicating the Token program will own it). The second is a `programId: Tokenkeg...` instruction; if parsed, it might show `"type": "initializeAccount"` with an info containing the account, mint, and owner. After execution, the account's `owner` field on chain is the Token program, and within its data (which you can decode via token program libraries) it will have the associated mint and owner addresses.

### Mint Account (for a new token mint)

Creating a token **mint** is similar but uses a different size and initialization call. A Mint account requires 82 bytes of space (for the Mint data structure). The steps are:

1. System `createAccount` (payer -> new mint account, lamports for rent, space = 82, owner = Token program)
2. Token Program `InitializeMint` instruction, providing parameters like **decimals**, **mintAuthority**, and an optional **freezeAuthority**

The JSON would show the createAccount (owner Tokenkeg, space 82) and then a `initializeMint` parsed instruction. The new mint's account info after will have `owner: Tokenkeg...` and data with the mint configuration. (For example, an RPC parsed transaction might list a `parsed.info.mintAuthority`, etc., in the initializeMint call.)

### Multisig Account

The SPL Token program also allows a special multisig account (to use N-of-M signing for token authority). That account type needs space for a structure that holds multiple signer addresses. The creation is again System create (with owner = Token program, space = 355 bytes for a 2-of-3 multisig or up to 11 signer multisig, typically 355 bytes covers maximum 11 signers) and then a `InitializeMultisig` instruction on the Token program (specifying the number of signatures required and the signer addresses).

### Usage

These manual methods are typically used by scripts or programs that need more control (for example, creating token accounts for PDAs or creating extra token accounts beyond the one associated with your wallet). As a transaction parser, you'll identify these when you see a System create instruction targeting the Token program, followed by a Token program initialize. It's essentially the same outcome as the associated account creation but without the convenience of the ATA program (and not limited to the one address scheme).

## 6. Creating Stake Accounts

**Stake accounts** are special accounts owned by the Stake Program (`Stake11111111111111111111111111111111111111`) that hold SOL delegated to validators for staking. They have a specific data structure and require a certain size and initial balance. To create a stake account, the common procedure is:

1. Call System Program `CreateAccount` with the new account's public key, lamports to fund (this will be the staked amount plus rent), space = 200 bytes (the size of the stake state data structure, which includes stake delegation info and meta), and owner = Stake Program ID.
2. Then invoke Stake Program's `Initialize` instruction on that new account. The Initialize instruction sets the stake account's **authorized** keys (the staker and withdrawer authorities) and an optional lockup. For instance, CLI commands require you to provide the authorized public keys during creation.

In a transaction, you will see a `programId: 11111111111111111111111111111111` create account instruction where `"owner": "Stake11111111111111111111111111111111111111"` in the parsed info, indicating a new stake account is being allocated. Next, a `programId: Stake11111111111111111111111111111111111111` instruction, likely parsed as `"Initialize"`, with info about the stake and withdraw authority addresses. After this, the account is ready and in an "inactive" state (funded but not yet delegated).

A concrete example: The Solana CLI `solana create-stake-account` does exactly this under the hood. The transaction's instructions might be labeled "System Program: Create Account" and "Stake Program: Initialize (stake)" in a block explorer.

### Stake Account Split

In addition to creating a stake account from scratch, Solana allows **splitting** an existing stake account into a new one. This is a special case of account creation: the new stake account is created and immediately funded from an existing stake's lamports. The mechanism is: you (the stake authority) provide a fresh account (either by creating it with System Program in the same transaction, or specifying a derived address) and call the Stake Program's `Split` instruction. In practice, a split operation typically bundles:

1. System Program create account (for the new stake account, owner = Stake program, lamports = rent-exempt minimum for the stake account data, or sometimes 0 lamports if splitting will transfer the funds; usually the new account must at least be rent-exempt)
2. Stake Program `Split` instruction, specifying how many lamports to move from the source stake to the new stake account

After splitting, the new account carries a portion of the stake (in an identical state as the source, just with reduced balance). The original documentation notes: "splitting the stake account, creating a new stake account with a portion of the funds in the first account". In a transaction log, you'd see something like "System Program: Create Account" followed by "Stake Program: Split" back-to-back. The presence of a Stake Split is a clear sign that a new stake account was formed. (If the new account address was a derived PDA using `withSeed`, the create might be `CreateAccountWithSeed` instead of the regular one.)

### Delegation

Note that after creation (or split), a stake account still needs a separate `DelegateStake` instruction to actually begin staking to a validator. That isn't an account creation step but often comes in workflow after initialization.

As an engineer, to identify stake account creations, look for System instructions that assign to `Stake111111...` and the size ~200, or any use of Stake Program's initialize or split. The new stake accounts will have data that can be decoded to reveal stake details, and their `owner` field on chain will be the Stake program.

## 7. Creating Vote Accounts

**Vote accounts** are another special account type, owned by the Vote Program (`Vote111111111111111111111111111111111111111`), used by validators to record their votes and earn rewards. Only validators (or those setting up a validator) typically create vote accounts. The process is analogous to stake accounts:

1. **System CreateAccount**: to allocate a new account owned by the Vote Program. Vote accounts have a larger data size (several kilobytes; for example, a vote account stores vote credits, authorized voters, a vote history, etc.). The exact space is on the order of a few KB (often around Vote state struct size, which might be ~SizeOf(VoteState), roughly 3730 bytes, but using CLI ensures correct value). The funding lamports must cover rent for that size.
2. **Vote Program InitializeAccount**: an instruction that initializes the vote account with the validator's identity, vote authority, commission percentage, and authorized withdrawer. The Solana CLI's `solana create-vote-account` command packages these: you provide a keypair for the new vote account, a validator identity public key, and optionally a different vote-authority and withdraw-authority key.

In the transaction, expect "System Program: Create Account" (with `owner: Vote111111111...`) followed by "Vote Program: InitializeAccount" (or similar name) containing the node identity, vote and withdraw authority addresses. After that, the vote account is ready and can start receiving votes from the validator node. The vote account address is fixed after creation (if you needed it deterministic, you can use the `--seed` option in the CLI, which essentially uses `createAccountWithSeed` behind the scenes).

Because vote accounts are usually created by validators via CLI or scripts, you might not frequently parse them in typical dApp transaction flows. But if you do, the hallmark is the owner being the Vote program and the Initialize instruction. Once active, stake accounts from delegators will point to this vote account to delegate stake.

## 8. Nonce Accounts (Durable Transaction Nonces)

Solana's System Program also supports **durable transaction nonces**, which require a **Nonce Account**. A nonce account is a system-owned account that stores a one-time nonce value (allowing offline signing or preventing transaction expiration issues). All nonce accounts are simply accounts **owned by the System Program** (owner = `11111111111111111111111111111111`), but with a specific data layout of 80 bytes, including an authority pubkey and the current nonce.

To create a nonce account, one typically uses the convenience of the CLI (`solana create-nonce-account`) or corresponding RPC calls. Under the hood, it performs:

1. **System CreateAccount**: The new account's owner is set to the System Program (it remains a system account, not assigned to a different program). Space allocated is 80 bytes (the length of nonce state). Lamports must cover rent for 80 bytes (and any additional amount you want to put as spendable balance for fees).
2. **System Program InitializeNonce**: An instruction on the System Program that takes the newly created account and sets it into Nonce state. This writes the recent blockhash value into the account's data (as the initial nonce) and designates the nonce authority (usually the creator's key or another provided key) that can advance or withdraw from this account.

In a transaction, these could appear as two back-to-back System instructions. The first create account will show `owner: 11111111111111111111111111111111` and 80 bytes space (parsed type `"createAccount"`). The second might be parsed as `"initializeNonce"` under the system program, with an info including the nonce account pubkey and the authorized pubkey (nonce authority). If not parsed, you'd identify it by the instruction data (the System Program's instruction enum value for InitializeNonce, which is 4). The new nonce account's fields after initialization: it will have some lamports (whatever was funded), owner = System Program, and data containing a Valid nonce state (including the authority's pubkey in bytes 8–40 as noted in the stackexchange snippet).

### Usage

Nonce accounts are used for offline transaction signing or ensuring a transaction can remain valid beyond the usual blockhash timeout. From a parsing standpoint, nonce account creation is less common in everyday app usage, but it appears in workflows involving offline fee paying or wallet custodians. You can spot nonce account operations by the presence of System Program instructions like InitializeNonce, AdvanceNonce, or WithdrawNonce in a transaction. When a nonce account is eventually closed (with WithdrawNonce draining its lamports), it becomes a normal system account again with 0 data and can be deleted.

## 9. Program Deployments (BPF Program Accounts)

Solana "programs" (smart contracts) themselves live in special accounts. Deploying a program involves creating one or more accounts to hold the program's code and metadata. The **BPF Loader programs** (e.g., `BPFLoaderUpgradeab1e11111111111111111111111111` for the upgradeable loader) manage these, but the creation of the accounts still ultimately goes through the System Program or the loader's internal create logic.

When you deploy a program using `solana program deploy`, the process will:

1. **Create a Program Account**: a small account that will ultimately have the executable flag set. If using the upgradeable loader (v3), this is actually the Program account (owned by BPFLoaderUpgrade program) which is a pointer to a Program Data account. For immutable programs (v2 or earlier), the single account holds the code.
2. **Create a Program Data Account**: (for upgradeable programs) which holds the actual ELF binary bytes and is owned by BPFLoaderUpgrade. This account's size equals the program's code length and is rent-exempt funded. It's usually created via the BPF loader's own instructions (`Allocate` and `Write`), but those underneath use system account creation to allocate the space.
3. **Possibly a Buffer Account**: In some deployment flows, a temporary buffer account is created to upload the program data, then the buffer is consumed to finalize the program. The CLI often does: System create a buffer account (owner = BPFLoaderUpgrade), upload chunks to it, then invoke a loader instruction to finalize which internally may create the Program account or assign the buffer to it.

In transaction records, deploying a program can be complex – it might span multiple transactions (one for each write chunk and one for finalize). The final transaction to finalize an upgradeable program typically has a Loader instruction like `DeployProgram` or `UpgradeProgram` that might implicitly create the Program Data account if not already present. The simpler case is that you will see explicit System creates for the program accounts. For example, a transaction might list:

- "System Program: Create Account" (with owner = BPFLoaderUpgradeab1e... for the program data, space = some large number for code size)
- another "System Program: Create Account" (owner = BPFLoaderUpgrade... for the program account itself, space = 0 or a small fixed size, just enough to hold the program's metadata like authority)
- then one or more BPFLoader instructions (Write, Finalize)

After deployment, the program account's executable flag becomes `true`. If you query that account, it will have `owner` equal to the loader program (so not System Program but one of the loaders), and data either empty (in case of program account that just delegates to program-data) or filled with the code (for older loader). The program data account (if upgradeable) will have the bulk of data and be owned by the loader as well.

### Identifying program account creation

Look for create instructions with owner being one of the BPF Loader IDs (the common ones: `BPFLoader2111111111111111111111111111111111` for legacy, or the upgradeable `BPFLoaderUpgradeab1e11111111111111111111111111`). Also, the lamports amounts are usually quite large because the program account must be rent-exempt (and code can be hundreds of KB to a few MB). This is a rarer scenario unless you specifically track program deployments.

## Recognizing Account Creation in Transactions (Summary)

Every method above ultimately relies on the System Program to allocate new accounts, either directly or indirectly. When scanning transaction data for account creations, here are some general signs to look for:

- **System Program invoked**: If the transaction (or inner instructions) contains a call to the System Program (`11111111111111111111111111111111`) with an instruction that has parameters for lamports and space, it's very likely creating or allocating an account. In parsed JSON this appears as `type: "createAccount"` or `"createAccountWithSeed"` (or occasionally `allocate` / `assign` in advanced usage). The presence of `"newAccount"` in the parsed info is a dead giveaway.

- **New account key in account list**: The public key of the account being created will be listed among the accountKeys in the transaction. It will often be a non-signer and marked as writable (since it's being initialized). Its index in `preBalances` will have 0, and in `postBalances` will have the funded amount.

- **Owner field change**: After creation, if you query the account or look at post-transaction state (in `meta.postTokenBalances` for token accounts, or via a separate getAccountInfo), the account's `owner` will be set to the program that it was intended for (Token program, Stake program, etc.). For instance, a token account creation will result in an account owned by the Token program with data length 165 bytes.

- **Associated token account creation**: Look for the ATA Program's ID (`ATokenGPvbdGVxr...`) in the instruction – that single instruction implies a create+init is happening internally. Inner instruction records will confirm the system create.

- **Program-specific init instructions**: Usually, creation of a specialized account is followed by some initialization call. E.g., after any System create for a non-system owner, expect an instruction from the new owner program: `Initialize...` (stake, vote, token mint/account, etc.), or in Anchor programs a custom initialization. These paired instructions in sequence strongly indicate a new account was made and then set up.

- **Transfer to new address**: As noted earlier, a pure System Transfer to a new address will create a system account implicitly. So if you see a transfer instruction where the destination had no prior account (and the transaction didn't explicitly create it), that destination account will be created as a system account with the transferred lamports. The explorer or RPC might not call it out as a "create" event, but the account's existence post-tx (and 0 pre-balance) is the evidence.

By combining these clues with the instruction data, you can detect every instance of account creation in Solana transactions. Whether it's a user wallet being created by an airdrop, a new token account for an SPL token, or a complex on-chain PDA initialization, the fingerprint in the transaction record will include the System Program doing an allocation of lamports and space for a fresh account. Keep an eye on those and the subsequent owner assignments, and you'll successfully identify all account creation techniques on Solana.

## Sources

- Solana Documentation – Accounts and Programs
- Solana Cookbook – Creating Accounts
- QuickNode Solana RPC – Parsed transaction example
- Sec3 Deep Dive – Native Programs (System, Token, ATA, Stake)
- Solana Stack Exchange – System Program internals and Nonce accounts
- Solana Stake Accounts Reference

### References

1. [Accounts | Solana](https://solana.com/docs/core/accounts)
2. [How to Create an Account | Solana](https://solana.com/developers/cookbook/accounts/create-account)
3. [Programs | Solana](https://solana.com/docs/core/programs)
4. [getTransaction RPC Method | Solana Docs](https://www.quicknode.com/docs/solana/getTransaction)
5. [instructions - Implementation of AssignWithSeed? - Solana Stack Exchange](https://solana.stackexchange.com/questions/3924/implementation-of-assignwithseed)
6. [Validator Guide: Vote Account Management | Solana Validator](https://docs.solanalabs.com/operations/guides/vote-accounts)
7. [Solana Internals Part 1: What Are the Native On-Chain Programs and Why Do They Matter? | Sec3 Blog](https://www.sec3.dev/blog/solana-internals-part-1-what-are-the-native-on-chain-programs-and-why-do-they-matter)
8. [Stake Accounts | Solana](https://solana.com/docs/references/staking/stake-accounts)
9. [Transaction - Explorer | Solana](https://explorer.solana.com/tx/41MYXRKrHEY8d8RYLca1zCugkXwJEdSrfZ6MprihiPgGuQd67pA59NGWEJKTNz2pk6wuK3SBQyPyMQtRBuPFUvjw)
10. [Transaction - Explorer | Solana](https://explorer.solana.com/tx/5RWJNS5DUke5rYkKikUHMSoDZrwHnMAJobDNCKcLQvPFcwLtUfauavDfaHieZFbSaMC1k9uEEFZiGC94oDnkWuY?cluster=ethereum)
11. [solana program - Is there any way to get nonce accounts of an authorized account? - Solana Stack Exchange](https://solana.stackexchange.com/questions/9650/is-there-any-way-to-get-nonce-accounts-of-an-authorized-account)
