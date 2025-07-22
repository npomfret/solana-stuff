# Solana PDA Derivation Patterns

This document outlines the common design patterns used across the Solana ecosystem to create accounts using Program Derived Addresses (PDAs). While the high-level technique is consistent, the implementation details vary significantly between protocols.

## The Core Concept: PDAs

A Program Derived Address (PDA) is a unique, predictable address derived from a program's ID and a set of "seeds." This mechanism allows a smart contract (program) to manage data accounts programmatically.

The primary tool for this in the `@solana/web3.js` library is `PublicKey.findProgramAddressSync`.

---

## Pattern 1: User-Centric PDAs

These are the most common patterns, where the PDA's derivation is primarily seeded by a user's public key. This directly links an on-chain account to a specific user.

### 1.1: Classic `[string, user_pubkey]`

The most fundamental pattern, combining a static string with the user's key.

*   **Use Case:** Creating a single, primary user account within a protocol.
*   **Protocols:**
    *   **Clone:** `[Buffer.from('user'), owner.toBuffer()]`
    *   **Grass:** `[owner.toBytes(), Buffer.from('userStakeInfo')]`
    *   **Rain:** `[Buffer.from('poolv2'), owner.toBuffer()]`
    *   **Vaultka:** `[Buffer.from('USER_INFOS'), owner.toBuffer()]`
    *   **Zelo:** `[Buffer.from('user-v2'), owner.toBuffer()]`

### 1.2: User & State `[string, state_pubkey, user_pubkey]`

Links a user to a global protocol account (like a market, registrar, or pool).

*   **Use Case:** Creating a user-specific record within the context of a larger protocol component.
*   **Protocols:**
    *   **01 Protocol:** `[owner.toBuffer(), state.toBuffer(), Buffer.from('marginv1')]`
    *   **DAOs (Helium):** `[registrar.toBuffer(), Buffer.from('voter'), owner.toBuffer()]`
    *   **Elemental:** `[Buffer.from('position'), owner.toBuffer(), poolAddress.toBuffer()]`
    *   **Flash:** `[Buffer.from('stake'), owner.toBuffer(), poolKey.toBuffer()]`
    *   **Hawksight:** `[Buffer.from('multi-user'), HAWKSIGHT_FARM.toBuffer(), owner.toBuffer()]`
    *   **Jupiter:** `[Buffer.from('Escrow'), lockerPubkey.toBytes(), owner.toBytes()]`
    *   **MagicEden:** `[m2Prefix.toBuffer(), m2AuctionHouse.toBuffer(), owner.toBuffer()]`
    *   **Marinade:** `[Buffer.from('claim_record'), claim_mint.toBuffer(), owner.toBuffer()]`
    *   **Meteora:** `[owner.toBuffer(), farm.toBuffer()]`
    *   **Phoenix:** `[Buffer.from('seat'), market.toBuffer(), trader.toBuffer()]`
    *   **Raydium:** `[stake_seed.toBuffer(), owner.toBuffer(), staker_info_seed.toBuffer()]`
    *   **Zeta:** `[Buffer.from('margin'), zetaGroup.toBuffer(), userKey.toBuffer()]`

### 1.3: User & Token `[string, token_mint_pubkey, user_pubkey]`

Differentiates user accounts based on a specific token.

*   **Use Case:** For protocols where users can stake or deposit many different types of tokens.
*   **Protocols:**
    *   **Allbridge:** `[Buffer.from('user_deposit'), tokenAddress.toBuffer(), owner.toBuffer()]`
    *   **Nosana:** `[Buffer.from('stake'), nosMint.toBuffer(), owner.toBuffer()]`

### 1.4: User & Counter `[string, user_pubkey, number]`

Allows a user to have multiple, numbered, or indexed accounts.

*   **Use Case:** When a user needs to open several distinct positions, loans, or vaults.
*   **Protocols:**
    *   **Drift:** `[Buffer.from('user'), owner.toBuffer(), new BN(subAccountId).toArrayLike(Buffer, 'le', 2)]`
    *   **Zeta:** `[Buffer.from('stake-account'), userKey.toBuffer(), Uint8Array.from([bit])]`

### 1.5: Complex User PDAs (e.g., User + State + Counter)

Combines multiple elements for highly specific account derivation.

*   **Use Case:** For complex DeFi protocols requiring multiple layers of indexing.
*   **Protocols:**
    *   **Parcl:** `[Buffer.from('lp_position'), market_key.toBuffer(), owner.toBuffer(), new BN(index).toArrayLike(Buffer, 'le', 8)]`
    *   **Kamino (Multiply/Leverage):** `[version, [0], owner, market, token1, token2]`

---

## Pattern 2: Account-Centric PDAs

These PDAs are derived from other on-chain accounts (not users), such as token mints, state accounts, or even other PDAs.

### 2.1: Simple `[string, account_pubkey]`

Derives a child account directly from a parent account's public key.

*   **Use Case:** Finding a related account, like a vault or custody account for a given position.
*   **Protocols:**
    *   **Orca:** `[Buffer.from('position'), nft_mint.toBuffer()]` (derives position from its NFT)
    *   **Picasso:** `[Buffer.from('vault_params'), external_id.toBuffer()]`
    *   **Pyth:** `[Buffer.from('custody'), staking_account.toBuffer()]`
    *   **Raydium (CLMM):** `[Buffer.from('position'), nft_mint.toBuffer()]`

### 2.2: Nested PDAs

An advanced pattern where a PDA's address is used as a seed for another PDA.

*   **Use Case:** Creating hierarchies of accounts or linking accounts across programs.
*   **Protocols:**
    *   **Adrena:** `[Buffer.from('user_staking'), owner.toBuffer(), stakingPda.toBuffer()]`
    *   **Francium:** `[user.toBuffer(), farmingPool.toBuffer(), associated_token_account.toBuffer()]`
    *   **NX Finance:** `[lending_pool_pda.toBuffer(), owner.toBuffer(), Buffer.from('account')]`
    *   **Tensor:** `[Buffer.from('margin'), margin_group_pda.toBytes(), owner.toBytes(), ...]`

---

## Pattern 3: Singleton PDAs

PDAs with static seeds (no user or account input), creating a single global account for a program.

*   **Use Case:** Storing global configuration or state that needs to be accessible from a predictable address.
*   **Protocols:**
    *   **Vaultka:** `[Buffer.from('LENDING')]` or `[Buffer.from('STRATEGY')]`

---

## Pattern 4: The Metaplex Standard

A specialized, ecosystem-wide standard for attaching metadata to tokens.

*   **Use Case:** Finding the metadata or master edition account for any SPL token (fungible or NFT).
*   **Protocols:** Virtually all token/NFT protocols.
*   **Derivations:**
    *   **Metadata:** `[Buffer.from('metadata'), METADATA_PROGRAM_ID.toBuffer(), mint.toBuffer()]`
    *   **Master Edition:** `[Buffer.from('metadata'), METADATA_PROGRAM_ID.toBuffer(), mint.toBuffer(), Buffer.from('edition')]`

---

## Appendix: Edge Cases & Variations

*   **Sliced Seeds:** Some protocols use a slice of a public key as a seed. **Loverflow** uses `[owner.toBuffer().slice(0, 31)]`.
*   **Multi-Chain Address Formatting:** When interacting with non-Solana addresses, seeds might be formatted differently. **DeBridge** uses `Buffer.from(owner.slice(2), 'hex')` for EVM addresses.
