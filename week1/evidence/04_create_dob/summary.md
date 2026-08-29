# Exercise 4 - Create a DOB

> Date: 28/08/2026

## Goal

Learn how to create an on-chain digital object using Spore and read its content back from a CKB Cell.

---

## What I Did

- Ran the `create-dob` example on the local devnet.
- Selected a JPEG image as the test file.
- Created a DOB using Spore SDK.
- Got the transaction hash after creating the DOB.
- Checked the transaction and confirmed that it was `committed`.
- Read the Spore Cell again and displayed the same image from the stored data.

---

## What I Learned

### 1. Spore is used to create a digital object on CKB

The example uses:

```ts
createSpore(...)
```

to create the DOB.

The main data passed to Spore is:

```ts
data: {
  contentType: "image/jpeg",
  content,
}
```

My basic understanding is:

```text
Image
  ↓
Binary Data
  ↓
Spore
  ↓
CKB Cell
```

In this exercise, the digital object is a JPEG image stored through a Spore Cell.

---

### 2. The image is stored as binary data

The `content` parameter is:

```ts
content: Uint8Array
```

So the image is not stored as a normal filename like:

```text
cat.jpg
```

Instead, the image file is read and converted into binary data before creating the Spore.

The Spore data also keeps the content type:

```text
contentType = image/jpeg
```

This tells the application what kind of content is stored in the Cell.

---

### 3. The DOB Cell is controlled by my Lock Script

When creating the Spore:

```ts
toLock: wallet.lock
```

is passed to `createSpore()`.

My understanding is that the created Spore Cell is locked by my account's Lock Script.

So the Cell still follows the same basic idea from previous exercises:

```text
Spore Cell
├── Lock → owner
└── Data → digital object
```

Spore also uses its own Type Script to define the Cell as a Spore object.

---

### 4. Transaction hash and output index can locate the Spore Cell

After creating the DOB, the function returns:

```ts
return { txHash, outputIndex };
```

Later, the application uses both values to read the Cell:

```ts
cccClient.getCellLive(
  {
    txHash,
    index: indexHex
  },
  true
);
```

My understanding is:

```text
Transaction Hash
       +
Output Index
       ↓
    OutPoint
       ↓
   Spore Cell
```

This is similar to the Store Data on Cell exercise.

---

### 5. The Spore data can be read back from the Cell

After finding the Cell:

```ts
const sporeData =
  unpackToRawSporeData(cell.outputData);
```

The application reads:

```ts
cell.outputData
```

and then uses:

```ts
unpackToRawSporeData()
```

to decode the Spore data.

The basic flow is:

```text
Spore Cell
    ↓
outputData
    ↓
unpackToRawSporeData()
    ↓
contentType + content
    ↓
Image
```

I was able to read the data back and display the same image that I uploaded.

---

## Result

| Item | Result |
| --- | --- |
| File Type | JPEG |
| DOB Creation | Successful |
| Transaction | Committed |
| Read Spore Content | Successful |
| Rendered Image | Same as original image |

---

## Main Takeaway

This exercise helped me understand that a CKB Cell can store more than a small text message.

In the previous exercise:

```text
Cell Data
   ↓
"Hello CKB from Thang!"
```

In this exercise:

```text
Spore Cell Data
   ↓
JPEG Image Data
```

Spore provides a structured way to store and identify digital objects on CKB.

The complete flow I understood is:

```text
Image File
    ↓
Binary Data
    ↓
Create Spore
    ↓
Transaction
    ↓
Spore Cell
    ↓
txHash + outputIndex
    ↓
Read Cell
    ↓
Unpack Spore Data
    ↓
Display Image
```