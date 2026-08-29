# Exercise 2 - Store Data on Cell

> Date: 28/08/2026

## Goal

Understand how data can be stored inside a CKB Cell and how to read the stored data back from the Cell.

---

## What I Did

- Ran the `store-data-on-cell` example on the local devnet.
- Used the message `Hello CKB from Thang!` as test data.
- Created and sent a transaction to store the message.
- Checked the transaction and confirmed that it was `committed`.
- Read the message back from the created Cell.

---

## What I Learned

### 1. Data can be stored inside a Cell

In this exercise, the message is stored in the `data` field of a Cell.

The transaction is created with:

```ts
const tx = ccc.Transaction.from({
  outputs: [{ lock: signerAddress.script }],
  outputsData: [onChainMemoHex],
});
```

My basic understanding is:

```text
Output Cell
├── capacity
├── lock
└── data
     └── "Hello CKB from Thang!"
```

So a CKB Cell is not only used to hold CKB capacity. It can also contain data.

---

### 2. Text is converted to hex before storing

The message starts as normal text:

```text
Hello CKB from Thang!
```

Before putting it into `outputsData`, the application converts it:

```ts
const onChainMemoHex = utf8ToHex(onChainMemo);
```

The result looks like:

```text
0x48656c6c6f20434b422066726f6d205468616e6721
```

So the basic flow is:

```text
Text
 ↓
UTF-8 bytes
 ↓
Hex
 ↓
Cell Data
```

The Cell stores raw data, so the application needs to encode the human-readable string before storing it.

---

### 3. `outputsData` belongs to the transaction outputs

The transaction contains:

```ts
outputs: [{ lock: signerAddress.script }],
outputsData: [onChainMemoHex],
```

I understand that the elements correspond by index:

```text
outputs[0]  <->  outputsData[0]
```

In this exercise:

```text
outputs[0]
├── lock = my Lock Script
└── data = "Hello CKB from Thang!"
```

This creates a new Cell controlled by my Lock Script with my message stored in its data.

---

### 4. A Cell can be found using Transaction Hash + Output Index

To read the message, the application uses:

```ts
const cell = await cccClient.getCellLive(
  { txHash, index },
  true
);
```

The default index is:

```ts
index = "0x0"
```

My understanding is that a Cell can be identified by:

```text
Transaction Hash
       +
Output Index
       ↓
    OutPoint
       ↓
      Cell
```

The transaction hash tells us which transaction created the Cell, and the index tells us which output of that transaction we want.

---

### 5. Reading the stored message

After finding the Cell:

```ts
const data = cell.outputData;
```

The data is still in hex format.

The application converts it back using:

```ts
const msg = hexToUtf8(data);
```

So the complete flow is:

```text
Write:

"Hello CKB from Thang!"
          ↓
      utf8ToHex()
          ↓
      Cell Data


Read:

      Cell Data
          ↓
      hexToUtf8()
          ↓
"Hello CKB from Thang!"
```

I tested this and was able to read back the same message that I stored.

---

## Result

| Item | Result |
| --- | --- |
| Message | `Hello CKB from Thang!` |
| Transaction | Committed |
| Read message from Cell | Successful |

---

## Main Takeaway

The main thing I learned from this exercise is that a CKB Cell can contain both capacity and data.

In the previous Transfer CKB exercise, I mainly worked with:

```text
Cell
├── capacity
└── lock
```

After this exercise, my understanding becomes:

```text
Cell
├── capacity
├── lock
└── data
```

The data can be written when creating an output Cell and later read by finding that Cell using its transaction hash and output index.

This exercise also helped me understand the basic meaning of an **OutPoint**:

```text
OutPoint = Transaction Hash + Output Index
```