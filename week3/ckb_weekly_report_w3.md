# CKB Builder – Week 3 Report

## Overview

**Program:** CKB Builder\
**Week:** Week 3\
**Focus:** This week, I focused on moving from individual CKB/CCC examples in the Playground to a working CKB learning DApp.

The main goal was to understand how a frontend application connects to a CKB wallet, reads on-chain data, creates and sends CKB transactions, and communicates with a backend service connected to CKB Testnet.
**Project:** CKB Learning DApp / Backend Service

------------------------------------------------------------------------

## Project Links

- **Frontend Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp
- **Backend Service Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp-service

------------------------------------------------------------------------

## 1. CKB Wallet Integration with JoyID

I connected the React frontend to JoyID using `@ckb-ccc/connector-react`.

I learned the difference between:

- **Wallet** – the connected wallet integration information.
- **Signer** – the object used by the application for wallet signing and blockchain-related operations.

The application retrieves the recommended CKB address using:

```ts
const signer = ccc.useSigner();

const address = await signer.getRecommendedAddress();
```

My JoyID CKB Testnet address was:

```text
ckt1qrfrwcdnvssswdwpn3s9v8fp87emat306ctjwsm3nmlkjg8qyza2cqgqqyqxav89cr6uhwpjxahkxjrvckgjcja56c63c7ct
```

This helped me understand how a frontend can use a connected wallet's signer to work with CKB.

---

## 2. Reading CKB Balance

I implemented balance retrieval in the frontend:

```ts
const amount = await signer.getBalance();

setBalance(
  ccc.fixedPointToString(amount)
);
```

I learned that CKB balances are represented through the capacity of Cells owned by a Lock Script.

My JoyID address received Testnet CKB and the DApp successfully displayed the balance.

After testing CKB transfers, the balance changed from:

```text
10000 CKB
```

to:

```text
9999.99895482 CKB
```

This allowed me to observe the effect of transaction fees on the CKB balance.

---

## 3. Understanding and Querying Live Cells

I implemented Live Cell querying using the wallet's Lock Script:

```ts
const { script: lock } =
  await signer.getRecommendedAddressObj();

for await (
  const cell of signer.client.findCellsByLock(lock)
) {
  foundCells.push(cell);
}
```

I learned that a CKB Cell contains important components such as:

- Capacity
- Lock Script
- Type Script
- Data

For example, before sending a transaction, my wallet had one Live Cell:

```text
Capacity: 10000 CKB
Type Script: None
Data: 0x
```

I also learned about the **OutPoint**:

```text
transaction hash : output index
```

An OutPoint identifies a specific Cell output created by a transaction.

---

## 4. Querying Transaction History

I implemented transaction history querying using:

```ts
for await (
  const txRecord of signer.client.findTransactionsByLock(
    lock,
    null,
    true,
  )
) {
  ...
}
```

The DApp displays:

- Transaction Hash
- Block Number

Initially, my new JoyID address had no transaction history.

After receiving Testnet CKB, the faucet transaction appeared.

After making a CKB transfer, a second transaction appeared.

This helped me understand the difference between:

- Current blockchain state, represented by Live Cells.
- Historical activity, represented by transactions.

---

## 5. CKB Testnet and Explorer Verification

I obtained Testnet CKB for my JoyID address and verified the result in the CKB Testnet Explorer.

The faucet transaction created a Live Cell containing:

```text
10000 CKB
```

I then used the Explorer to verify the transaction and Cell information.

This was useful because I could compare:

```text
My React DApp
        ↓
CCC
        ↓
CKB Testnet
        ↓
CKB Testnet Explorer
```

I learned that the data displayed by my DApp was actual on-chain data rather than simulated data.

---

## 6. Building and Sending a CKB Transaction

I added a Send CKB function to the frontend.

The transaction flow was:

```text
Receiver Address
       ↓
Address → Lock Script
       ↓
Create Transaction
       ↓
Select Input Cells
       ↓
Calculate Fee + Change
       ↓
JoyID Signing
       ↓
Broadcast
       ↓
CKB Testnet
```

### Creating the transaction

```ts
const { script: lock } = await ccc.Address.fromString(
  receiver.trim(),
  signer.client,
);

const tx = ccc.Transaction.from({
  outputs: [
    {
      capacity: ccc.fixedPointFrom(amount),
      lock,
    },
  ],
});
```

This creates a transaction with the desired output.

### Selecting input Cells

```ts
await tx.completeInputsByCapacity(signer);
```

I learned that the transaction needs input Cells to provide the capacity required by its outputs.

### Completing the fee and change

```ts
await tx.completeFeeBy(signer);
```

This completed the transaction by handling the fee and creating the necessary change output.

### Signing and broadcasting

```ts
const txHash = await signer.sendTransaction(tx);
```

JoyID was used to sign the transaction, and the transaction was then broadcast to CKB Testnet.

---

## 7. Observing the Cell Model Through a Real Transaction

One of the most important things I learned this week was how a CKB transaction changes Cells.

Before the transfer, I had:

```text
1 Live Cell
└── 10000 CKB
```

After sending CKB, the original Cell was consumed and new output Cells were created.

The resulting transaction created two Live Cells:

```text
Cell #0
Capacity: 63 CKB

Cell #1
Capacity: 9936.99948472 CKB
```

The total was:

```text
63 + 9936.99948472
= 9999.99948472 CKB
```

The difference from the original 10,000 CKB represented the transaction fee.

This helped me understand an important CKB concept:

> A transaction consumes existing input Cells and creates new output Cells. The original Cell is not modified directly.

I also learned that Cell capacity is not simply the same concept as the amount entered into a transfer form. A Cell needs sufficient capacity to store its Lock Script, Type Script, and Data.

---

## 8. Transaction Hash and Explorer Link

I updated the DApp so that after a successful transaction it displays the transaction hash and a direct link to the CKB Testnet Explorer.

Example:

```text
Transaction sent successfully!

TX Hash:
0xaa815233ad93cab3c7b32bc3f6ad6a2b97e3a801d2e9fb939b50e2b941bad381

View transaction on CKB Testnet Explorer
```

I also added Explorer links to transaction history so each transaction can be inspected on-chain.

---

## 9. Backend: Node.js + Express + CCC

This week I also started connecting the frontend to a backend service.

The backend is a Node.js application using:

- Node.js
- Express
- `@ckb-ccc/ccc`

The backend creates a CKB Testnet client:

```js
const client = new ccc.ClientPublicTestnet();
```

This allows the backend to communicate with CKB Testnet.

---

## 10. Backend API – Latest Block

I implemented:

```text
GET /api/ckb/tip
```

The API uses:

```js
const tip = await client.getTip();
```

and returns the latest CKB Testnet block.

I then called the API from the React frontend using `fetch()`.

The frontend successfully displayed:

```text
Latest CKB Testnet Block from Backend: 22384072
```

This demonstrated the complete flow:

```text
React Frontend
      ↓
HTTP Request
      ↓
Node.js Backend
      ↓
CCC
      ↓
CKB Testnet
```

---

## 11. Backend API – Balance by Address

I implemented:

```text
GET /api/ckb/balance?address=...
```

The backend receives a CKB address:

```js
const address = req.query.address;
```

It converts the address into a Lock Script:

```js
const { script: lock } =
  await ccc.Address.fromString(
    address,
    client,
  );
```

Then it queries the balance:

```js
const balance =
  await client.getBalanceSingle(lock);
```

The API returned:

```json
{
  "address": "ckt1qrfrwcdnvssswdwpn3s9v8fp87emat306ctjwsm3nmlkjg8qyza2cqgqqyqxav89cr6uhwpjxahkxjrvckgjcja56c63c7ct",
  "balance": "9999.99895482",
  "unit": "CKB"
}
```

I also connected the frontend to this endpoint.

The frontend successfully displayed:

```text
Balance from Backend: 9999.99895482 CKB
```

This confirmed that the frontend, backend, CCC, and CKB Testnet were communicating successfully.

---

## 12. CORS and Frontend–Backend Communication

When the frontend first called:

```text
http://localhost:3000/api/ckb/tip
```

the browser returned:

```text
net::ERR_FAILED 200 (OK)
```

The backend had actually returned HTTP 200, but the browser blocked the frontend from reading the response because the frontend and backend were running on different origins.

I solved this by adding the Express CORS middleware:

```js
const cors = require("cors");

app.use(cors());
```

After this change, the frontend could successfully call the backend API.

This helped me understand that successful backend execution and successful browser access to an API are not always the same thing.

---

# 13. Main Concepts Learned This Week

### CKB Cell Model

I learned that CKB stores assets in Cells and that a Cell contains:

```text
Capacity
Lock Script
Type Script
Data
```

### Lock Script

The Lock Script defines the conditions required to unlock and spend a Cell.

### Type Script

The Type Script provides additional validation rules for a Cell when it is used by a protocol such as a token or other application-specific asset.

### OutPoint

An OutPoint identifies a particular transaction output:

```text
Transaction Hash + Output Index
```

### Transaction Inputs and Outputs

Inputs reference existing Cells that are being consumed.

Outputs create new Cells.

### Change

When the input capacity is greater than the amount needed for the transaction, the remaining capacity can be returned as change.

### Transaction Fee

The difference between input capacity and output capacity is used to pay the transaction fee.

### Signer

The signer provides the interface needed to sign transactions and interact with wallet capabilities.

### CCC

CCC provides APIs for working with CKB, including:

- Addresses
- Scripts
- Transactions
- Cells
- Signers
- CKB clients
- UDT-related functionality

### Frontend and Backend Architecture

I learned that the frontend and backend can communicate through HTTP APIs while interacting with CKB using CCC.

---

# 14. Week 3 Result

By the end of Week 3, I built a working CKB learning DApp with:

- JoyID wallet connection
- CKB address display
- CKB balance display
- Live Cell querying
- Transaction history querying
- CKB Testnet transaction creation
- CKB Testnet transaction signing through JoyID
- Transaction broadcasting
- Transaction hash display
- CKB Testnet Explorer links
- Node.js + Express backend
- Latest block API
- Balance-by-address API
- Frontend-to-backend API communication

The most important learning this week was moving from individual CCC/CKB examples to a working application and observing how wallet, transaction, Cell, backend, and blockchain components fit together.

---

## Key Takeaway

This week helped me understand CKB more practically.

Instead of thinking of a balance as a simple account value, I now understand that CKB is represented through Cells and that transactions consume existing Cells and create new ones.

I also learned how CCC can be used across the application stack to connect a frontend wallet experience and a backend service to CKB Testnet.
