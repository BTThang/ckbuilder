# Exercise 1 - Transfer CKB

> Date: 28/08/2026

## Goal

Get familiar with transferring CKB on a local devnet and understand the basic flow of how a transaction works on CKB.

---

## What I Did

- Started a local CKB devnet using OffCKB.
- Ran the `simple-transfer` project.
- Used two devnet accounts as the sender and receiver.
- Transferred **62 CKB** from the sender to the receiver.
- Got the transaction hash after sending the transaction.
- Used RPC to check the transaction and confirmed that it was **committed**.
- Checked the receiver again and confirmed that its balance increased by **62 CKB**.

---

## What I Learned

### 1. Private Key and CKB Address

In the code, the private key is used to create a **Signer**.

```ts
const signer = new ccc.SignerCkbPrivateKey(...)
```

From the signer, we can get information such as:

- Public key
- Lock Script
- CKB Address

My basic understanding is:

```text
Private Key
    ↓
  Signer
    ↓
Lock Script
    ↓
CKB Address
```

The private key is also used by the signer when signing a transaction.

---

### 2. Balance is Related to the Lock Script

At first, I thought the application would simply use the address to get an account balance.

However, the code first converts the address:

```ts
const addr = await ccc.Address.fromString(address, cccClient);
```

and then uses its Lock Script to get the balance:

```ts
cccClient.getBalance([addr.script]);
```

My understanding is that the balance on CKB comes from the capacity of Cells controlled by the corresponding Lock Script, rather than just a `balance` value stored in an account.

---

### 3. The Receiver Gets a New Output Cell

The receiver address is converted into a Lock Script:

```ts
const { script: toLock } =
  await ccc.Address.fromString(toAddress, cccClient);
```

Then that Lock Script is used to create the transaction output:

```ts
outputs: [{ lock: toLock }]
```

So my basic understanding is:

```text
Receiver Address
       ↓
Receiver Lock Script
       ↓
New Output Cell
```

The transaction does not directly update the receiver's balance. It creates a new output Cell that is controlled by the receiver's Lock Script.

---

### 4. The Transaction Needs Enough Input Capacity

After creating the output, the code calls:

```ts
await tx.completeInputsByCapacity(signer);
```

CCC finds suitable Cells controlled by the sender to provide enough capacity for the transaction.

My current understanding is:

```text
Sender's Cells
      ↓
Input Cells
      ↓
Transaction
      ↓
Output Cell + Change
```

This was one of the first places where I could see the CKB Cell Model working in practice.

---

### 5. Transaction Hash Does Not Mean the Transaction is Already Committed

After clicking **Transfer**, I received a transaction hash.

I then checked the transaction using the `get_transaction` RPC method.

The transaction was successfully completed when the RPC result showed:

```text
status: committed
```

So the flow I observed was:

```text
Send Transaction
      ↓
Get Transaction Hash
      ↓
Transaction Processing
      ↓
status: committed
```

---

## Result

| Item | Result |
| --- | --- |
| Transfer Amount | 62 CKB |
| Transaction Status | Committed |
| Receiver Balance Change | +62 CKB |

---

## Main Takeaway

Before this exercise, I mainly thought of a transfer like this:

```text
Sender -= 62
Receiver += 62
```

After completing the exercise, my understanding is closer to:

```text
Sender's Cells
      ↓
Consumed as Inputs
      ↓
Transaction
      ↓
New Output Cell for Receiver
```

The main thing I learned from this exercise is that CKB does not simply update an account balance. A transaction consumes existing Cells and creates new Cells as outputs.

This helped me understand the Cell Model more clearly through an actual transaction.