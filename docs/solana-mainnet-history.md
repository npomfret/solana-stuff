# Solana Mainnet Release History (2020–2025)

Below is a chronological technical history of **Solana mainnet-beta** releases, focusing on core updates from the official `solana-labs/solana` repository. Each release entry includes the version, release date, and highlights of changes: new features, bug fixes, performance improvements, and architectural updates.

> **Note:** Solana’s mainnet was considered a **beta** network through these releases. Only mainnet-beta releases are listed (testnet/devnet releases are omitted). Many features were gated (disabled by default) until activated by cluster consensus. Dates are given in UTC.

## Mainnet Beta Launch – v1.0.0 (16 Mar 2020)

Solana’s first public mainnet-beta block was produced on 16 Mar 2020. The initial release introduced:

* **Proof of History (PoH)** – verifiable delay function used as a global clock  
* **Tower BFT** – PoS consensus optimized with PoH clock  
* **Turbine** – UDP-based block propagation  
* **Sealevel** – parallel smart‑contract runtime  
* **Cloudbreak & Pipelining** – storage + processing pipeline  
* Initial validator & JSON‑RPC clients

Baseline TPS: ~50–65 k in tests; block time ~400 ms.

---

## v1.1.0 (31 Mar 2020)

Minor maintenance:

* Bug fixes & polish  
* New stake‑account CLI utility  
* Slight temporary TPS regression noted

---

## v1.2.0 (27 May 2020)

Major runtime features:

* **Cross‑Program Invocation (CPI)** – contracts call other contracts  
* **Program‑Derived Addresses (PDAs)** – program‑signed accounts  
* **Optimistic Confirmation** – 1‑block finality for clients

---

## v1.3.0 (07 Aug 2020)

* Launch of **Solana Program Library (SPL)** with Token & Memo programs  
* BigTable export for long‑term transaction history  
* CPI & PDA performance improvements

---

## v1.4.0 (14 Oct 2020)

* **Persistent tower vote record** on disk (`--require-tower`)  
* Accounts DB and gossip traffic optimizations  
* Foundation enabled “pico‑inflation” soon after

---

## v1.5.x (Feb 2021)

* Stability + memory leak fixes  
* Better transaction/vote coalescing  
* Full staking rewards activated

---

## v1.6.x (Mid 2021)

* Memory & performance tuning  
* Prepared for rising DeFi load  
* Lessons from Sept 2021 17‑hr outage

---

## v1.7.x (Oct 2021)

* **Incremental snapshots** for fast restarts  
* Initial stake‑weighted QoS groundwork  
* BankingStage vote‑channel fix  
* RPC index improvements

---

## v1.8.x (Dec 2021 – Jan 2022)

* UDP flow‑control & spam mitigation  
* Fork cleanup logic  
* Durable‑nonce temporarily disabled after bot abuse

---

## v1.9.x (Early 2022)

* Nonce overhaul & spam fix  
* Prototype fee‑priority instruction  
* Memory optimisations; bridge to QUIC era

---

## v1.10.x (Jun 2022)

* **QUIC** transport replaces raw UDP  
* **Stake‑weighted QoS** live  
* Fee‑market code (behind gate)  
* Big memory & stability gains

---

## v1.11.x (Oct 2022)

* **Local fee markets** (priority fees) activated  
* QUIC/QoS tuning  
* Durable nonce safely re‑enabled

---

## v1.13.x (Dec 2022 – Q1 2023)

* **State compression** support (compressed NFTs)  
* Early groundwork for diet/light clients  
* Validator RAM reduction beginnings

---

## v1.14.x (May 2023)

Feature‑gated megapatch:

* **Disk‑backed accounts index** (lower hardware)  
* Stake program upgrades (minimum delegation, deactivate delinquent)  
* **Estimated fee RPC**  
* Turbine erasure coding improvements  
* Higher transaction account‑limit

---

## v1.16.x (Nov 2023)

* **Confidential Transfers** (Token‑2022 privacy)  
* **Alt_bn128 zk‑syscalls** & proof compression  
* Extensive security audit & fuzzing

---

## v1.17.x (Apr 2024)

* Congestion‑relief patch (v1.17.31)  
* QUIC optimizations & better staked/unstaked throttling  
* BankingStage forwarding filter

---

## v1.18.x (Jul 2024)

* **Central transaction scheduler**  
* Runtime efficiency & rbpf 0.8.3  
* Transition of repo to **Agave** (anza‑xyz)  
* Sets stage for Agave v2.0

---

## Roadmap (2024–2025)

* **Agave v2.0** – runtime decoupling, cleaner codebase  
* **Firedancer** C++ validator for client diversity & >1 M TPS  
* **SIMD‑10 Diet Clients** – trustless light nodes  
* **Timely Vote Credits** & fee‑market tweaks  
* Further gossip/repair & optimistic parallelism upgrades
