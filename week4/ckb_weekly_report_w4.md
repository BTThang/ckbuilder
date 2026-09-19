# CKB Learning DApp — Week 4 Report

------------------------------------------------------------------------

## Project Links

- **Frontend Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp
- **Backend Service Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp-service

------------------------------------------------------------------------

## 1. Overview

During Week 4, I focused on implementing and testing advanced features of the CKB Learning DApp using the **CCC (CKBers' Community Chain) library**.

The main objectives were:

- Create and transfer xUDT tokens.
- Query xUDT balances from CKB Testnet.
- Create and read Spore/DOB digital objects.
- Sign and verify messages.
- Connect the frontend with a backend service.
- Retrieve CKB transaction information through the backend.

By the end of Week 4, all planned core features were successfully implemented and tested on **CKB Testnet**.

---

## 2. xUDT — Minting and Querying Token Balance

### 2.1 xUDT Type Script

The xUDT arguments used in the project were:

```text
0xe7b4dfefe01e736895142578e00996e9649f204b05ef1c4cae88d8f4b42b984b
```

The xUDT code hash was:

```text
0x25c29dc317811a6f6f3985a7a9ebc4838bd388d19d0feeecf0bcd60f6c0975bb
```

I used CCC to construct the xUDT Type Script:

```ts
const xudtType =
  await ccc.Script.fromKnownScript(
    signer.client,
    ccc.KnownScript.XUdt,
    xudtArgs,
  );
```

This allowed the application to identify Cells belonging to the specific xUDT token.

### 2.2 Querying xUDT Balance

I used the CKB client to search for Cells containing the xUDT:

```ts
const cells =
  await signer.client.findCells(
    {
      script: xudtType,
      scriptType: "type",
      scriptSearchMode: "exact",
    },
    "asc",
    100,
  );
```

The application decoded `outputData` to obtain the raw token amount.

Because the token uses **8 decimal places**, the raw amount is converted into the human-readable amount.

For example:

```text
2,100,000,000 raw units
        ↓
21 xUDT
```

The second JoyID Testnet wallet was successfully queried and returned:

```text
Total raw xUDT amount: 2100000000
Formatted new xUDT amount: 21
```

---

## 3. xUDT Transfer

I implemented xUDT transfers using the CCC UDT API.

```ts
const udt = new ccc.udt.Udt(
  code,
  xudtType,
);
```

The recipient address was converted into a lock script:

```ts
const { script: receiverLock } =
  await ccc.Address.fromString(
    xudtReceiver,
    signer.client,
  );
```

The transfer was created with:

```ts
const { res: tx } =
  await udt.transfer(
    signer,
    [
      {
        to: receiverLock,
        amount,
      },
    ],
  );
```

CCC was then used to complete the transaction inputs and fee:

```ts
await udt.completeBy(tx, signer);

await tx.completeInputsByCapacity(signer);

await tx.completeFeeBy(signer);
```

Finally, the transaction was signed and broadcast:

```ts
const txHash =
  await signer.sendTransaction(tx);
```

The transfer was successfully confirmed on CKB Testnet. The recipient wallet was subsequently queried and showed **21 xUDT**, confirming the transfer and balance-query logic.

---

## 4. Spore / DOB

### 4.1 Creating a Spore

I used the `@ckb-ccc/spore` package to create a Spore.

The content was stored as UTF-8 bytes:

```ts
const { tx, id } =
  await spore.createSpore({
    signer,
    data: {
      contentType: "text/plain",
      content: new TextEncoder().encode(
        sporeContent,
      ),
    },
  });
```

The test content was:

```text
My first CKB Spore
```

The Spore was successfully created on CKB Testnet.

**Spore ID:**

```text
0x73a4d08b78f401c05b0db6bb4f19decdadd8bafac22d0245d15e94759b83f62d
```

**Transaction Hash:**

```text
0x011798e6c17f0769d9755cce47aed5f4026880de09051795ee2d70a777f29a7b
```

The transaction was confirmed with:

```text
status: committed
```

### 4.2 Reading and Decoding Spore Data

I implemented a function to retrieve the transaction from CKB:

```ts
const sporeTx =
  await signer.client.getTransaction(
    sporeTxHash,
  );
```

The Spore data was stored in `outputsData`.

I decoded the binary data using `Uint8Array`, `DataView`, and `TextDecoder`.

The decoded result was:

```text
Content Type: text/plain
Content: My first CKB Spore
```

This demonstrated how Spore content can be retrieved from on-chain Cell data and decoded by the application.

---

## 5. Sign and Verify Message

I implemented message signing using the CCC signer.

The test message was:

```text
Hello CKB!
```

The message was signed using:

```ts
const result =
  await signer.signMessage(
    signMessage,
  );
```

The returned signature was verified using:

```ts
const isValid =
  await ccc.Signer.verifyMessage(
    signMessage,
    signature,
  );
```

The verification result was:

```text
Valid signature
```

The complete flow was:

```text
Message
   ↓
Wallet signs message
   ↓
Signature
   ↓
Verify signature
   ↓
Valid signature
```

---

## 6. Frontend ↔ Backend Integration

I connected the React frontend with a Node.js backend built using:

- Express
- CORS
- `@ckb-ccc/ccc`

The backend creates a CKB Testnet client:

```js
const client =
  new ccc.ClientPublicTestnet();
```

The backend provides these APIs:

```text
GET /api/ckb/tip
GET /api/ckb/balance
GET /api/ckb/transaction
```

### 6.1 Get Latest CKB Block

The backend uses:

```js
const tip = await client.getTip();
```

The frontend calls:

```ts
const response = await fetch(
  "http://localhost:3000/api/ckb/tip",
);
```

This demonstrates communication between the React frontend and backend service.

### 6.2 Get CKB Balance

The backend accepts a wallet address:

```text
/api/ckb/balance?address=...
```

It converts the address into a lock script:

```js
const { script: lock } =
  await ccc.Address.fromString(
    address,
    client,
  );
```

Then it queries the CKB balance:

```js
const balance =
  await client.getBalanceSingle(lock);
```

The frontend receives the result through `fetch()`.

---

## 7. Transaction Tracking Through Backend

I added a transaction API:

```js
app.get(
  "/api/ckb/transaction",
  async (req, res) => {
```

The backend receives a transaction hash:

```text
/api/ckb/transaction?txHash=...
```

and queries CKB Testnet:

```js
const tx =
  await client.getTransaction(txHash);
```

The API then returns the transaction information to the frontend.

During implementation, I encountered an issue because CKB transaction objects contain `BigInt` values, which cannot be directly serialized to JSON. I solved this by converting `BigInt` values to strings before returning the JSON response.

After the fix, the frontend successfully received:

```text
Transaction Hash:
0x011798e6c17f0769d9755cce47aed5f4026880de09051795ee2d70a777f29a7b

Transaction Status:
committed
```

The complete flow was:

```text
React Frontend
      ↓
HTTP Request
      ↓
Express Backend
      ↓
CCC Client
      ↓
CKB Testnet
      ↓
Transaction Data
      ↓
Express Backend
      ↓
React Frontend
```

---

## 8. Challenges

One challenge was handling CKB transaction data containing `BigInt` values because they cannot be directly serialized into JSON. I also needed to understand how xUDT Cell data and Spore binary data are represented and decoded on-chain. These issues were resolved through testing and by adapting the data-processing logic.

---

## 9. Week 4 Results

| Task | Status |
|---|---|
| Mint xUDT | Completed |
| Query xUDT balance | Completed |
| Transfer xUDT | Completed |
| Verify xUDT on-chain | Completed |
| Create Spore/DOB | Completed |
| Read Spore/DOB | Completed |
| Decode Spore content | Completed |
| Sign Message | Completed |
| Verify Message | Completed |
| Backend → CKB RPC | Completed |
| Frontend → Backend | Completed |
| Retrieve transaction through Backend | Completed |

---

## 10. Conclusion

Week 4 focused on moving beyond basic CKB Cell operations and implementing practical blockchain features.

I successfully worked with **xUDT**, **Spore/DOB**, message signing and verification, and **Frontend ↔ Backend integration**. The project was tested against **CKB Testnet**, including real token transfers and an on-chain Spore transaction.

The completed implementation provides a foundation for the next stage of CKB development, including improving the application structure, user experience, and more advanced blockchain functionality.
