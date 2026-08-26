## Builder Track Weekly Report — Week 1

**Name:** Thang Bui
**Week Ending:** 08-24-2026

### Courses Completed

- Completed the first **two modules** of the CKB Academy:
- Reading introduction to Nervos CKB - basic technical concepts and terminology explained
  - **CKB Theoretical Knowledge**:
    - Structure of a cell
    - Structure of a transaction
  - **First CKB Transaction:** [View on Explorer](https://testnet.explorer.nervos.org/transaction/0xf47dfc83255f7f048c9a990e41246f31d74cea28f7ae126393644fba40ca03ce)
  - **Reading [RFC 0022](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md)** on Transaction Structure:
    - Deep dive into `cell_deps`, `witnesses`, `lock` and `type` scripts
    - Understood code locating and execution mechanisms in CKB
- Studied blockchain fundamentals through a review of the **Bitcoin whitepaper** to deepen foundational understanding
- Studied sections 1, 2, and 3 of the [Rust Book](https://doc.rust-lang.org/book/)

### Key Learnings

- Developed a detailed understanding of the **UTXO-based transaction model in CKB**, including:
  - The role and structure of **witnesses**, **lock scripts**, and **type scripts**
  - How **script grouping** and **syscalls** function within the CKB-VM execution environment
- Learned some **Rust** elementary notions:
  - Variables and mutability
  - Functions
  - Data types

### Practical Progress

- **Set up a local CKB dev node** successfully
- Executed a transaction from a **miner address to a secondary account**
- Began using CLI tools:
  - `wallet get-capacity`
  - `transfer`
  - `wallet get-live-cells`
- Wrote and ran a **guided Rust project** (Guessing game) to begin learning Rust for on-chain development

### Environment

- CKB node and local dev environment installed and functional
- Basic CLI usage and debugging started
- Rust and Cargo installed