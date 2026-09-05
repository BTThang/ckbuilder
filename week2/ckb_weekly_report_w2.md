# CKB Builder Weekly Report --- Week 2

## 1. Overview

**Program:** CKB Builder\
**Week:** Week 2\
**Focus:** Understanding CKB dApp development through CCC, CKB Clients,
Cells, Scripts, transactions, xUDT, Spore/DOB, and wallet integration\
**Project:** CKB Learning DApp / Backend Service

During Week 2, I focused on understanding **how CKB works from the
application side** rather than only learning individual APIs.

The main learning was to understand the flow:

``` text
Application
    ↓
CCC
    ↓
Client / Signer
    ↓
Transaction / Cell operations
    ↓
CKB Scripts
    ↓
CKB Network
```

I studied how CKB applications find and consume Cells, create new Cells,
attach data or assets to Cells, use Lock Scripts and Type Scripts for
validation, and use CCC to simplify these operations.

I also set up a Node.js backend connected to CKB Testnet and a React
frontend using the CCC React connector.

------------------------------------------------------------------------

## Project Links

- **Frontend Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp
- **Backend Service Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp-service

------------------------------------------------------------------------

# 2. Development Environment

## Backend

-   Node.js
-   `@ckb-ccc/ccc@1.3.0`
-   CKB Testnet
-   `ccc.ClientPublicTestnet()`

## Frontend

-   React
-   TypeScript
-   Vite
-   `@ckb-ccc/connector-react@1.1.9`

The backend and frontend were separated so that I could understand the
different roles of blockchain access, wallet interaction, and
application UI.

------------------------------------------------------------------------

# 3. The Role of CCC in CKB Development

One of the main things I learned this week is that **CCC is an
abstraction layer for interacting with CKB**.

Instead of manually implementing every low-level CKB operation, CCC
provides objects and utilities for:

-   CKB Clients
-   Addresses
-   Scripts
-   Transactions
-   Cells
-   Signers
-   Wallet connectors
-   xUDT
-   Spore

Conceptually:

``` text
React / Node.js Application
          ↓
         CCC
    ┌─────┴─────┐
    ↓           ↓
  Client      Signer
    ↓           ↓
 CKB RPC      Wallet
    ↓           ↓
       CKB Network
```

The important distinction I learned is:

-   **Client** → communicates with the CKB network and queries
    blockchain state.
-   **Signer** → represents the wallet-side ability to authorize and
    sign operations.
-   **Wallet Connector** → connects the frontend application to
    supported wallets.
-   **Transaction** → represents the state transition from consumed
    Cells to newly created Cells.

------------------------------------------------------------------------

# 4. Connecting to CKB Testnet

On the backend, I created a public Testnet client:

``` js
const client = new ccc.ClientPublicTestnet();
```

This creates a CCC client configured to communicate with the CKB
Testnet.

I then used:

``` js
const tip = await client.getTip();
```

to retrieve the current Testnet tip.

The flow is:

``` text
Node.js
   ↓
ccc.ClientPublicTestnet()
   ↓
CKB Testnet RPC
   ↓
Current Tip Block
```

The result was a live Testnet block number.

### What I learned

The Client is the application's connection to the CKB blockchain state.

It can be used to query information from the network, while wallet
signing requires a Signer.

------------------------------------------------------------------------

# 5. Understanding CKB's Cell Model

The most important CKB concept I learned is that CKB is based on
**Cells**.

A simplified Cell can be understood as:

``` text
Cell
├── capacity
├── lock
├── type
└── data
```

A transaction does not simply modify an account balance.

Instead:

``` text
Existing Cells
      ↓
    Inputs
      ↓
 Transaction Validation
      ↓
    Outputs
      ↓
New Cells
```

This model is used for:

-   transferring CKB,
-   storing application data,
-   holding xUDT tokens,
-   representing Spore/DOB objects.

This helped me understand that many different CKB features are actually
different ways of creating, finding, validating, consuming, and creating
Cells.

------------------------------------------------------------------------

# 6. Scripts: Lock Script and Type Script

A Cell can contain different Scripts with different responsibilities.

## Lock Script

The Lock Script controls **who can unlock/consume the Cell**.

Conceptually:

``` text
Cell
 ↓
Lock Script
 ↓
Who is allowed to spend this Cell?
```

The wallet normally provides the signature required by the Lock Script.

## Type Script

The Type Script defines additional rules for how a Cell can be created
or consumed.

Conceptually:

``` text
Cell
 ↓
Type Script
 ↓
What rules must this Cell follow?
```

This distinction becomes particularly important when working with tokens
such as xUDT.

------------------------------------------------------------------------

# 7. CKB Transaction Flow

I learned to think about a CKB transaction as a multi-step process
rather than simply "send coins".

The general flow is:

``` text
1. Define the desired output
        ↓
2. Find suitable input Cells
        ↓
3. Add inputs
        ↓
4. Calculate capacity requirements
        ↓
5. Create change if necessary
        ↓
6. Calculate transaction fee
        ↓
7. Validate / complete transaction
        ↓
8. Wallet signs
        ↓
9. Broadcast
        ↓
10. Transaction becomes committed
        ↓
11. New Cells become live
```

CCC helps automate several of these steps.

For example:

``` ts
await tx.completeInputsByCapacity(signer);
await tx.completeFeeBy(signer);
```

The first method helps complete the transaction with sufficient input
capacity.

The second completes the transaction fee requirements.

Then:

``` ts
const txHash = await signer.sendTransaction(tx);
```

sends the completed transaction through the Signer.

### What I learned

There is a difference between:

``` text
Transaction intent
```

and:

``` text
Complete transaction ready for signing
```

The application first describes what it wants to create, then CCC helps
complete the transaction, and finally the wallet signs and broadcasts
it.

------------------------------------------------------------------------

# 8. CKB Transfer Flow

For a basic CKB transfer, the recipient address must first be
interpreted as a CKB lock Script.

For example:

``` ts
const { script: lock } =
  await ccc.Address.fromString(
    receiver,
    signer.client
  );
```

### What this code does

``` text
receiver
   ↓
CKB address
   ↓
Address.fromString(...)
   ↓
Lock Script
```

The address is therefore not simply stored as a string in the
transaction.

It is converted into the Script that defines the ownership conditions of
the output Cell.

Then the transaction can be created:

``` ts
const tx = ccc.Transaction.from({
  outputs: [
    {
      lock,
      capacity: ccc.fixedPointFrom(amount),
    },
  ],
});
```

This creates the desired output Cell.

Conceptually:

``` text
Recipient Address
       ↓
    Lock Script
       ↓
 Output Cell
 ├── capacity = amount
 └── lock = recipient lock
```

Then CCC completes the transaction:

``` ts
await tx.completeInputsByCapacity(signer);
await tx.completeFeeBy(signer, feeRate);
```

Finally:

``` ts
const txHash = await signer.sendTransaction(tx);
```

The overall transfer flow is:

``` text
Sender Wallet
     ↓
Find Input Cells
     ↓
Create Recipient Output Cell
     ↓
Create Change Cell if necessary
     ↓
Calculate Fee
     ↓
Sign
     ↓
Broadcast
     ↓
Transaction Hash
```

------------------------------------------------------------------------

# 9. Storing UTF-8 Application Data in a Cell

I studied how arbitrary application data can be stored in the `data`
field of a CKB Cell.

The first step is converting text into bytes.

For example:

``` ts
const bytes = new TextEncoder().encode(message);
```

The bytes can then be represented as hexadecimal:

``` text
UTF-8 text
    ↓
Bytes
    ↓
Hexadecimal
    ↓
Cell data
```

The transaction can then contain:

``` ts
const tx = ccc.Transaction.from({
  outputs: [
    {
      lock: owner.script,
      capacity: ccc.fixedPointFrom(capacity),
    },
  ],
  outputsData: [data],
});
```

Here:

``` text
outputs
    ↓
defines the new Cell

outputsData
    ↓
defines the data stored inside that Cell
```

The resulting structure can be viewed conceptually as:

``` text
Output Cell
├── Capacity
├── Lock Script
├── Type Script (if used)
└── Data
       ↓
   UTF-8 message
```

------------------------------------------------------------------------

# 10. Reading a Live Cell

After a transaction creates an output, I learned that the output does
not necessarily become immediately available as a live Cell.

A Cell can be identified by:

``` text
Transaction Hash
+
Output Index
```

For example:

``` ts
const cell = await signer.client.getCellLive(
  {
    txHash,
    index: "0x0",
  },
  true
);
```

### What this code does

It asks the CKB client for the Cell corresponding to:

``` text
transaction = txHash
output index = 0
```

and checks whether that output is currently a **live Cell**.

The flow is:

``` text
Transaction
    ↓
Transaction Hash
    ↓
Output Index
    ↓
getCellLive(...)
    ↓
Live Cell
    ↓
Cell Data
    ↓
Decode bytes
    ↓
Original UTF-8 message
```

### What I learned

A transaction output is not simply permanent data.

The Cell Model is based on **state transition**:

``` text
Live Cell
    ↓
Consumed
    ↓
No longer live
```

Therefore, when an application needs current Cell state, it must
consider whether the Cell is still live.

------------------------------------------------------------------------

# 11. xUDT: Understanding Token Cells

I studied how xUDT uses CKB's Cell Model to represent fungible tokens.

A key concept is that an xUDT token is identified through a **Type
Script**.

The code:

``` ts
const typeScript =
  await ccc.Script.fromKnownScript(
    cccClient,
    ccc.KnownScript.XUdt,
    xudtArgs
  );
```

can be understood step by step.

### `ccc.KnownScript.XUdt`

This identifies the known xUDT Script.

``` text
KnownScript.XUdt
       ↓
Known xUDT validation logic
```

### `xudtArgs`

The arguments identify the particular xUDT instance/token configuration.

``` text
XUdt Script
    +
xudtArgs
    ↓
Specific xUDT Type Script
```

### `ccc.Script.fromKnownScript(...)`

This creates the Script object that represents the xUDT Type Script.

The result is:

``` ts
typeScript
```

which can then be used to identify Cells belonging to that xUDT.

------------------------------------------------------------------------

# 12. Finding Cells Containing a Specific xUDT

Once the xUDT Type Script has been constructed, I studied how the
application can search for Cells associated with it.

For example:

``` ts
const collector =
  cccClient.findCellsByType(
    typeScript,
    true
  );
```

The important concept here is:

``` text
typeScript
    ↓
findCellsByType(...)
    ↓
Cells using that Type Script
```

The second argument indicates that the search is looking for live Cells.

Conceptually:

``` text
CKB Network
     ↓
Find Cells
     ↓
Type Script = xUDT Type Script
     ↓
Live Cells
     ↓
Cells associated with this xUDT
```

These Cells are the on-chain locations where the token state is
represented.

The **Lock Script** on each Cell determines who can unlock/consume that
Cell, while the xUDT **Type Script** determines the token-specific
validation rules.

This helped me understand an important CKB concept:

``` text
Lock Script
    ↓
Who can spend the Cell?

Type Script
    ↓
What rules does the Cell have to follow?

Data
    ↓
What state/value is stored in the Cell?
```

------------------------------------------------------------------------

# 13. xUDT Token Flow

I learned to understand an xUDT transfer as a Cell transformation.

Conceptually:

``` text
Sender xUDT Cells
        ↓
      Inputs
        ↓
xUDT Type Script validation
        ↓
      Outputs
   ┌────┴─────┐
   ↓          ↓
Recipient   Sender Change
xUDT Cell   xUDT Cell
```

The important point is that the token does not move by simply changing a
balance field in an account.

Instead, token-bearing Cells are consumed and new token-bearing Cells
are created.

This connects xUDT directly to the CKB Cell Model.

------------------------------------------------------------------------

# 14. Spore / DOB Flow

I studied Spore/DOB as another example of using CKB Cells to represent
application-level digital objects.

Conceptually:

``` text
Digital Content
      ↓
Spore Protocol
      ↓
CKB Cell
      ↓
Digital Object
```

The content can have information such as:

``` text
contentType
content
```

For example:

``` ts
const { tx, id } = await createSpore({
  signer,
  data: {
    contentType: "text/plain",
    content: new TextEncoder().encode(content),
  },
});
```

### What this code does

The application provides:

``` text
contentType
    ↓
Describes the content format

content
    ↓
Actual digital content
```

The Spore protocol then uses CKB transaction/Cell mechanisms to create
the digital object.

The created object receives an identifier:

``` ts
id
```

which can be used to refer to that Spore.

### Spore lifecycle

I learned the conceptual lifecycle:

``` text
Create
  ↓
Spore Cell
  ↓
Transfer
  ↓
New owner
  ↓
Melt
  ↓
Spore consumed according to protocol rules
```

This demonstrated that the Cell Model can represent persistent digital
objects, not only monetary value.

------------------------------------------------------------------------

# 15. Message Signing Without a Transaction

I studied the difference between signing a message and signing a
blockchain transaction.

A message can be signed directly:

``` ts
const result =
  await signer.signMessage(message);
```

The message does not need to become a Cell or be broadcast to CKB.

The signature can then be verified:

``` ts
const verified =
  await ccc.Signer.verifyMessage(
    message,
    result
  );
```

The flow is:

``` text
Message
   ↓
Signer / Wallet
   ↓
Signature
   ↓
Verify Signature
   ↓
Proof of wallet control
```

This is different from a transaction:

``` text
Transaction
   ↓
Complete
   ↓
Sign
   ↓
Broadcast
   ↓
CKB State Change
```

### What I learned

Message signing is useful when an application needs to verify that a
user controls a wallet without changing blockchain state.

------------------------------------------------------------------------

# 16. Wallet Integration with React

For the frontend, I used:

``` text
@ckb-ccc/connector-react
```

The application is wrapped with:

``` tsx
<ccc.Provider>
  <App />
</ccc.Provider>
```

This provides the CCC React context to the application.

Then the application can use:

``` ts
const { open, wallet } =
  ccc.useCcc();
```

The `open` function opens the unified wallet connection interface.

The wallet selector successfully displayed multiple wallet options:

``` text
JoyID Passkey
MetaMask
OKX Wallet
UniSat
UTXO Global Wallet
```

This demonstrated the main benefit of the connector layer:

``` text
React Application
       ↓
ccc.Provider
       ↓
ccc.useCcc()
       ↓
Unified Wallet Connector
       ↓
Supported Wallets
```

The dApp does not need to implement an entirely separate UI flow for
every supported wallet.

------------------------------------------------------------------------

# 17. Wallet → Signer → CKB Operation

The wallet integration also helped me understand the role of the Signer.

Conceptually:

``` text
User
 ↓
Wallet
 ↓
Signer
 ↓
CKB Operation
```

The Signer can be used for operations requiring wallet authorization,
such as:

-   obtaining the recommended address,
-   signing messages,
-   completing transactions,
-   signing transactions,
-   sending transactions.

This is different from the Client:

``` text
Client
 ↓
Read blockchain state
```

versus:

``` text
Signer
 ↓
Authorize / sign blockchain operations
```

------------------------------------------------------------------------

# 18. Backend and Frontend Responsibilities

I learned that a CKB dApp can separate responsibilities between the
frontend and backend.

## Frontend

The frontend can handle:

``` text
UI
 ↓
Wallet connection
 ↓
Address
 ↓
Transaction composition
 ↓
User approval
 ↓
Wallet signing
```

## Backend

The backend can handle:

``` text
Application API
 ↓
CKB RPC queries
 ↓
Transaction lookup
 ↓
Transaction tracking
 ↓
Application database
```

The important security principle is:

``` text
Private Key
     ↓
User Wallet
```

The backend does not need the user's private key.

------------------------------------------------------------------------

# 19. Transaction Lifecycle

I learned that broadcasting a transaction does not necessarily mean the
transaction is immediately committed.

A simplified lifecycle is:

``` text
Transaction created
       ↓
Signed
       ↓
Broadcast
       ↓
Pending / Proposed
       ↓
Committed
```

After commitment, newly created Cells can become live and can later be
consumed by another transaction.

This is important when building applications that need to display
transaction status.

The application should not assume:

``` text
sendTransaction()
      ↓
immediately available Cell
```

Instead, it should consider the asynchronous nature of blockchain state.

------------------------------------------------------------------------

# 20. CKB Architecture I Learned

After studying the different features, I can now understand the overall
CKB dApp flow as:

``` text
                    User
                     ↓
              React Frontend
                     ↓
              CCC Connector
                     ↓
                  Wallet
                     ↓
                 Signer
                     ↓
              Transaction
                     ↓
          ┌──────────┴──────────┐
          ↓                     ↓
      Input Cells          Output Cells
          ↓                     ↓
    Lock / Type Rules      Lock / Type Rules
          └──────────┬──────────┘
                     ↓
                 CKB Network
                     ↓
                New State
```

For reading blockchain state:

``` text
Application
    ↓
CCC Client
    ↓
CKB RPC
    ↓
Cells / Transactions / Block State
```

For writing blockchain state:

``` text
Application
    ↓
CCC Transaction
    ↓
Complete Inputs + Fee
    ↓
Signer / Wallet
    ↓
Signature
    ↓
Broadcast
    ↓
CKB Network
```

------------------------------------------------------------------------

# 21. How the Different CKB Features Fit Together

One of the biggest things I learned is that CKB features that initially
look unrelated are connected through the same Cell Model.

## CKB Transfer

``` text
CKB Capacity
    ↓
Input Cells
    ↓
Output Cells
```

## Store Data

``` text
Cell
+
Data
```

## xUDT

``` text
Cell
+
xUDT Type Script
+
Token Data
```

## Spore / DOB

``` text
Cell
+
Spore Protocol Rules
+
Digital Content
```

All of these follow the same underlying pattern:

``` text
Find / Create Cells
        ↓
Apply Lock / Type Script Rules
        ↓
Consume Old Cells
        ↓
Create New Cells
```

This was the most important conceptual connection I gained during Week
2.

------------------------------------------------------------------------

# 22. Important Code Patterns Learned

## Create a CKB Testnet Client

``` ts
const client =
  new ccc.ClientPublicTestnet();
```

**Purpose:** Connect the application to CKB Testnet.

------------------------------------------------------------------------

## Convert a CKB Address to a Lock Script

``` ts
const { script: lock } =
  await ccc.Address.fromString(
    receiver,
    signer.client
  );
```

**Purpose:** Convert a human-readable CKB address into the Lock Script
used by the output Cell.

------------------------------------------------------------------------

## Create a Transaction Output

``` ts
const tx = ccc.Transaction.from({
  outputs: [
    {
      lock,
      capacity: ccc.fixedPointFrom(amount),
    },
  ],
});
```

**Purpose:** Declare the Cell that the transaction should create.

------------------------------------------------------------------------

## Complete Transaction Inputs

``` ts
await tx.completeInputsByCapacity(
  signer
);
```

**Purpose:** Find/add sufficient input capacity to fund the transaction.

------------------------------------------------------------------------

## Complete Transaction Fee

``` ts
await tx.completeFeeBy(
  signer,
  feeRate
);
```

**Purpose:** Complete the transaction's fee requirements.

------------------------------------------------------------------------

## Send a Transaction

``` ts
const txHash =
  await signer.sendTransaction(tx);
```

**Purpose:** Sign/send the transaction through the connected signer and
obtain the transaction hash.

------------------------------------------------------------------------

## Build a Known xUDT Type Script

``` ts
const typeScript =
  await ccc.Script.fromKnownScript(
    cccClient,
    ccc.KnownScript.XUdt,
    xudtArgs
  );
```

**Purpose:** Construct the Type Script representing a specific xUDT
configuration.

------------------------------------------------------------------------

## Find Cells by Type Script

``` ts
const collector =
  cccClient.findCellsByType(
    typeScript,
    true
  );
```

**Purpose:** Search for live Cells associated with the specified Type
Script.

The resulting Cells are the on-chain Cells associated with that xUDT.

The Lock Script of each Cell indicates who is authorized to
unlock/consume the Cell, while the xUDT Type Script determines the
token-specific validation rules.

------------------------------------------------------------------------

## Read a Live Cell

``` ts
const cell =
  await signer.client.getCellLive(
    {
      txHash,
      index: "0x0",
    },
    true
  );
```

**Purpose:** Retrieve a transaction output and determine/read its
current live Cell state.

------------------------------------------------------------------------

## Sign a Message

``` ts
const result =
  await signer.signMessage(
    message
  );
```

**Purpose:** Create a wallet signature without broadcasting a
transaction.

------------------------------------------------------------------------

## Verify a Message

``` ts
const verified =
  await ccc.Signer.verifyMessage(
    message,
    result
  );
```

**Purpose:** Verify that the signature corresponds to the message and
signer.

------------------------------------------------------------------------

# 23. Overall Learning Outcome

By the end of Week 2, I learned to view CKB development as a combination
of:

``` text
CCC
 +
Wallet / Signer
 +
Client / RPC
 +
Cells
 +
Scripts
 +
Transactions
 +
Protocols
```

The most important flow I learned is:

``` text
                    READ
                     │
                     ▼
Application ──→ CCC Client ──→ CKB RPC
                     │
                     ▼
             Cells / Tx / State


                   WRITE
                     │
                     ▼
Application ──→ CCC Transaction
                     │
                     ▼
              Complete Inputs
                     │
                     ▼
                Complete Fee
                     │
                     ▼
                  Signer
                     │
                     ▼
                  Wallet
                     │
                     ▼
                 Broadcast
                     │
                     ▼
                CKB Network
                     │
                     ▼
             New Live Cells
```

From there, different CKB applications build on the same foundation:

``` text
CKB Transfer
     │
     ├── Cell Capacity
     │
Store Data
     │
     ├── Cell Data
     │
xUDT
     │
     ├── Type Script
     ├── Token Cells
     │
Spore / DOB
     │
     ├── Protocol Rules
     └── Digital Object Cells
```

The key understanding I gained is that **CKB is fundamentally about
Cells and Script validation**. CCC does not replace this model; instead,
it provides developer-friendly tools for working with that model from
JavaScript/TypeScript applications.

This gave me a clearer understanding of the complete flow from a user's
wallet interaction, through CCC and transaction construction, to Cell
creation and validation on the CKB network.
