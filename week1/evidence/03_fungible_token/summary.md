# Exercise 3 - Create a Fungible Token

> Date: 28/08/2026

## Goal

Learn how to create a simple fungible token on CKB using xUDT, query the issued token, and transfer it to another account.

---

## What I Did

- Ran the `xudt` example on the local devnet.
- Issued **1000 xUDT tokens**.
- Got the xUDT args used to identify the token.
- Queried the token Cells using the xUDT args.
- Transferred **250 tokens** to another devnet account.
- Checked that the transaction was committed.
- Queried the token again and saw the token Cells after the transfer.

---

## What I Learned

### 1. xUDT uses a Type Script

In the previous exercises, I mainly worked with the Lock Script.

In this exercise, the token Cell also has a **Type Script**.

```ts
const typeScript = await ccc.Script.fromKnownScript(
  signer.client,
  ccc.KnownScript.XUdt,
  xudtArgs
);
```

The output Cell is created with both:

```ts
outputs: [{
  lock: lockScript,
  type: typeScript
}]
```

My basic understanding is:

```text
Token Cell
├── Lock Script → who can use the Cell
├── Type Script → xUDT token rules
└── Data        → token amount
```

This is the first exercise where I used a Type Script.

---

### 2. xUDT args are used to identify the token

The example creates the xUDT args from the issuer's Lock Script hash:

```ts
const xudtArgs = lockScript.hash() + "00000000";
```

Then the xUDT Type Script is created using these args.

My current understanding is:

```text
Issuer Lock Script
       ↓
     hash()
       ↓
   xUDT args
       ↓
xUDT Type Script
```

The same `xudtArgs` are used later when I want to query or transfer this token.

So in this exercise, I can think of the xUDT args as the identifier for the token I issued.

---

### 3. Token amount is stored in Cell data

When I issued **1000 tokens**, the amount was stored in `outputsData`:

```ts
outputsData: [
  ccc.numLeToBytes(amount, 16)
]
```

So the token Cell is roughly:

```text
Token Cell
├── Lock → my account
├── Type → xUDT
└── Data → 1000
```

This is similar to the previous Store Data exercise, but instead of storing a text message, the Cell data stores the token amount in the format expected by xUDT.

---

### 4. Tokens can be queried by their Type Script

To find the issued token, the code creates the same xUDT Type Script:

```ts
const typeScript = await ccc.Script.fromKnownScript(
  cccClient,
  ccc.KnownScript.XUdt,
  xudtArgs
);
```

and then searches for Cells with that Type Script:

```ts
cccClient.findCellsByType(typeScript, true);
```

My understanding is:

```text
xUDT args
    ↓
xUDT Type Script
    ↓
Find matching Live Cells
    ↓
Token Cells
```

This is how the application can find all Cells that contain the same issued token.

---

### 5. Transferring tokens also follows the Cell Model

For the transfer, I sent **250 tokens** to another account.

The receiver output is created with:

```ts
outputs: [{
  lock: receiverLockScript,
  type: xUdtType
}]
```

and:

```ts
outputsData: [
  ccc.numLeToBytes(amount, 16)
]
```

So the receiver gets a new Cell:

```text
Receiver Token Cell
├── Lock → receiver
├── Type → same xUDT
└── Data → 250
```

The important point is that the token type does not change. Only the owner and token amount of the new Cells are different.

---

### 6. Remaining tokens are returned to the sender

The code calculates the difference between input and output token amounts:

```ts
const balanceDiff =
  (await tx.getInputsUdtBalance(signer.client, xUdtType)) -
  tx.getOutputsUdtBalance(xUdtType);
```

If there are remaining tokens:

```ts
if (balanceDiff > ccc.Zero) {
  tx.addOutput(
    {
      lock: senderLockScript,
      type: xUdtType,
    },
    ccc.numLeToBytes(balanceDiff, 16)
  );
}
```

For my test:

```text
Before:

Sender
1000 xUDT

       ↓ transfer 250

After:

Receiver       Sender
250 xUDT       750 xUDT
```

So the original token Cell is consumed and new token Cells are created.

This helped me understand the Cell Model more clearly.

---

## Result

| Item | Result |
| --- | --- |
| Issued Token | 1000 xUDT |
| Issue Transaction | Committed |
| Transfer Amount | 250 xUDT |
| Receiver | 250 xUDT |
| Sender Remaining | 750 xUDT |

---

## Main Takeaway

The biggest thing I learned from this exercise is how `lock`, `type`, and `data` work together in a token Cell.

```text
Token Cell
│
├── Lock Script
│      └── Who owns/can unlock the Cell
│
├── Type Script
│      └── xUDT token rules
│
└── Data
       └── Token amount
```

I also understood that transferring tokens on CKB is still based on the Cell Model.

It is not simply:

```text
sender.tokenBalance -= 250
receiver.tokenBalance += 250
```

Instead:

```text
1000 Token Cell
       ↓
    consumed
       ↓
  Transaction
    /       \
   ↓         ↓
250 Cell   750 Cell
Receiver   Sender
```

The new Cells still use the same xUDT Type Script, but they have different Lock Scripts and token amounts.