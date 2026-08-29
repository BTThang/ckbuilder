# CKB Builder - Weekly Report - Week 1

## Week 1 Overview

This was my first week learning CKB.

I spent this week on two main things:

1. Reading the basic CKB concepts from the Nervos documentation and CKB Academy.
2. Doing the beginner exercises to understand those concepts in practice.

Before starting the exercises, I went through the basic theory to get a general idea of how CKB works.

References I used:

- [Nervos Blockchain](https://docs.nervos.org/docs/ckb-fundamentals/nervos-blockchain)
- [CKB Academy - Basic Theory](https://academy.ckb.dev/courses/basic-theory)

After that, I set up a local Devnet with OffCKB and worked through the five beginner exercises.

---

## What I studied

This week I mainly focused on the basic CKB concepts.

### Nervos / CKB

I learned that CKB (Common Knowledge Base) is the Layer 1 blockchain of the Nervos ecosystem.

One thing that was new to me was the relationship between CKByte and on-chain storage.

My current understanding is that CKByte is not only used as a native token. It also represents the right to occupy state/storage on CKB.

This became much easier to understand later when I worked with Cells containing text, tokens and an image.

### Cell Model

The Cell Model was the main topic I tried to understand this week.

At first, I was still thinking about blockchain mainly as accounts with balances.

After reading the theory and doing the exercises, my basic mental model became:

    Existing Cells
          ↓
        Inputs
          ↓
      Transaction
          ↓
        Outputs
          ↓
       New Cells

A transaction does not directly modify an existing Cell.

It consumes existing Cells and creates new Cells.

I think this was the most important concept for me in Week 1.

### Cell structure

I also learned the basic parts of a Cell:

    Cell
    ├── Capacity
    ├── Lock Script
    ├── Type Script (optional)
    └── Data

At this point, my understanding is:

- Capacity represents the Cell's occupied CKB/storage capacity.
- Lock Script controls who or what condition can consume the Cell.
- Type Script defines additional rules for the Cell.
- Data stores the actual state/data associated with the Cell.

I am still learning the details, especially around Scripts.

### Transaction

I spent some time looking at the basic transaction structure:

    Transaction
    ├── inputs
    ├── outputs
    ├── outputs_data
    ├── cell_deps
    └── witnesses

The exercises helped me understand these fields better than reading the definitions alone.

For example:

- `inputs` reference Cells that will be consumed.
- `outputs` describe the new Cells.
- `outputs_data` contains data for the output Cells.
- `witnesses` can contain information required to unlock input Cells.
- `cell_deps` provide dependencies needed by Scripts.

I still need more practice with `cell_deps` and `witnesses`.

### Scripts

I learned the basic difference between Lock Script and Type Script.

My current mental model is:

    Lock Script
        ↓
    Can this Cell be consumed?

    Type Script
        ↓
    Does this Cell follow the required rules?

The Simple Lock exercise helped me understand Lock Script much better because I could build a custom unlocking condition instead of only using the default lock.

---

# Hands-on Work

## 0. Development Environment

I set up the local development environment with OffCKB and started a local Devnet.

I also checked the generated accounts and tested the RPC connection before starting the exercises.

What I did:

- Installed and checked OffCKB.
- Started the local Devnet.
- Checked Devnet accounts.
- Verified the node through RPC.

Evidence:

- [Environment evidence](./evidence/00_environment/)

---

## 1. Transfer CKB

The first exercise was transferring CKB between two Devnet accounts.

I transferred **62 CKB** and checked the transaction status until it was committed.

The main thing I got from this exercise was seeing the Cell Model in a real transaction.

Instead of thinking:

    sender balance - 62
    receiver balance + 62

I started thinking:

    Input Cells
        ↓
    Transaction
        ↓
    Output Cells

I also learned that receiving a transaction hash does not mean the transaction is already committed.

Details and screenshots:

- [Exercise notes](./evidence/01_transfer_ckb/summary.md)
- [Evidence](./evidence/01_transfer_ckb/)

---

## 2. Store Data on Cell

In this exercise, I stored:

    Hello CKB from Thang!

inside Cell data.

The message was converted to bytes, written into an output Cell, and later read back from the Cell.

This helped me understand `outputs_data` and also introduced me to OutPoint.

My basic understanding of an OutPoint is:

    Transaction Hash + Output Index
                  ↓
            Specific Cell

Details and screenshots:

- [Exercise notes](./evidence/02_store_data_cell/summary.md)
- [Evidence](./evidence/02_store_data_cell/)

---

## 3. Create a Fungible Token

In this exercise, I worked with xUDT.

I issued **1000 xUDT tokens**, queried the token Cells and transferred part of the tokens to another Devnet account.

This was where Type Script became easier for me to understand.

A token Cell looked roughly like this to me:

    Token Cell
    ├── Lock Script → owner
    ├── Type Script → xUDT
    └── Data        → token amount

The transfer also gave me another example of the Cell Model.

For example, transferring 250 from a 1000-token Cell can result in:

        1000
          ↓
      Transaction
       /        \
      ↓          ↓
     250        750
    Receiver    Sender

The original token Cell is consumed and new token Cells are created.

Details and screenshots:

- [Exercise notes](./evidence/03_fungible_token/summary.md)
- [Evidence](./evidence/03_fungible_token/)

---

## 4. Create a DOB

I used Spore to create an on-chain digital object from a JPEG image.

The basic flow I followed was:

    JPEG Image
        ↓
    Binary Data
        ↓
    Create Spore
        ↓
      CKB Cell
        ↓
     Read Cell
        ↓
    Render Image

This exercise connected back to the Store Data exercise.

Instead of storing a short text message, I could see how larger and more structured content can be represented using a Cell.

I also used the transaction hash and output index again to find the created Spore Cell.

Details and screenshots:

- [Exercise notes](./evidence/04_create_dob/summary.md)
- [Evidence](./evidence/04_create_dob/)

---

## 5. Build a Simple Lock

This was probably the exercise I spent the most time understanding this week.

I built the `hash-lock` contract, deployed it to the local Devnet and used the frontend to interact with it.

I used:

    Hello World

as the preimage and deposited **300 CKB** into the generated hash-lock address.

The important part for me was understanding the relationship between Script args and Witness.

My current understanding is:

    Lock Script args
          ↓
     Expected Hash

    Witness
          ↓
       Preimage
          ↓
        hash()
          ↓
    Compare with Expected Hash

If they match, the Lock Script returns success and the Cell can be consumed.

This changed how I thought about Lock Scripts.

Before doing this exercise, I mostly associated a Lock Script with an address/private key.

Now I understand that a Lock Script is a program that defines the condition required to consume a Cell.

Details and screenshots:

- [Exercise notes](./evidence/05_simple_lock/summary.md)
- [Evidence](./evidence/05_simple_lock/)

---

# What became clearer this week

After combining the theory with the exercises, these are the concepts that became clearer to me:

- CKB uses the Cell Model instead of a traditional account/state model.
- Transactions consume old Cells and create new Cells.
- CKByte is related to on-chain storage/capacity.
- A Cell can contain capacity, scripts and data.
- Lock Script controls the condition for consuming a Cell.
- Type Script can define additional rules for a Cell.
- Cell data can represent different things: text, token amounts, digital objects, etc.
- A specific Cell can be referenced using a transaction hash and output index.
- Witness can provide data required by a Lock Script when spending a Cell.
- A submitted transaction is not necessarily committed yet.

There are still parts I only understand at a basic level, especially CKB-VM, Script execution, `cell_deps`, Witness structure and how Script args are constructed.

These are things I want to continue looking into.

---

# Questions I wrote down

While doing the exercises, I wrote down some questions that I want to understand better:

1. What is the difference between a Lock Script and a Type Script?
2. What exactly happens to a Cell after it is consumed?
3. Why do we need both a transaction hash and an output index to locate a Cell?
4. How is the minimum capacity of a Cell calculated?
5. Why does storing a large image require more Cell capacity than storing a short text message?
6. What is the difference between Script args and Witness?
7. How does a Lock Script use Witness data when validating a transaction?
8. What exactly is `cell_deps` used for during Script execution?
9. How does CKB-VM execute a Script?
10. Why does the Simple Hash Lock example use `slice(35)` when reading Script args?

I don't have complete answers to all of these yet. I plan to revisit them as I continue with the next exercises.

---

# Issues / Debugging

One issue I ran into this week was while building the Simple Lock example.

`pnpm` initially stopped because some dependency build scripts were not approved:

    ERR_PNPM_IGNORED_BUILDS

I checked the dependencies and used:

    pnpm approve-builds

After approving the required build scripts, I was able to build the Hash Lock successfully.

I kept separate notes for problems/debugging here:

- [Issues and debugging](./issue_and_debugging.md)

---

# Week 1 Status

Theory / reading:

- [x] Nervos / CKB overview
- [x] Basic Cell Model
- [x] Cell structure and capacity
- [x] Basic transaction structure
- [x] Basic Lock Script / Type Script concepts
- [ ] Need deeper understanding of CKB-VM and Script execution

Hands-on:

- [x] Set up local CKB Devnet
- [x] Transfer CKB
- [x] Store Data on Cell
- [x] Create a Fungible Token
- [x] Create a DOB
- [x] Build and deploy a Simple Lock

All screenshots and detailed exercise notes are available in the [evidence](./evidence/) directory.