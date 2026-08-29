# Exercise 5 - Build a Simple Lock

> Date: 28/08/2026

## Goal

Learn how a custom Lock Script works on CKB by building a simple hash lock.

The Cell can only be spent when the correct preimage is provided.

---

## What I Did

- Built the `hash-lock` contract.
- Deployed the contract to the local devnet.
- Started the Simple Hash Lock frontend.
- Used `Hello World` as the preimage.
- Generated a hash-lock address from the preimage hash.
- Deposited 300 CKB into the generated hash-lock address.
- Used the preimage to unlock and spend the Cell.
- Checked the transaction result on the local devnet.

---

## What I Learned

### 1. A Lock Script can define custom unlocking rules

In the previous exercises, I mostly used the normal CKB lock based on a private key.

In this exercise, I used my own Lock Script:

```text
hash-lock
```

Instead of checking a signature, this Lock Script checks whether I know the correct preimage.

The basic idea is:

```text
Preimage
"Hello World"
      ↓
   hashCkb()
      ↓
Expected Hash
      ↓
Lock Script args
```

To spend the Cell, I need to provide the original preimage.

---

### 2. The expected hash is stored in Script args

The contract loads its own Script:

```ts
let expect_hash =
  new Uint8Array(HighLevel.loadScript().args).slice(35);
```

The expected hash is taken from the Lock Script `args`.

In my test:

```text
Hello World
     ↓
blake2b-256
     ↓
0x106911e4f83e...
```

and this hash can also be seen inside the Lock Script args.

So the Lock Script already knows the hash it expects.

It does not need to store the original `Hello World`.

---

### 3. The preimage is provided through the Witness

The contract reads the witness:

```ts
let witness_args =
  HighLevel.loadWitnessArgs(
    0,
    bindings.SOURCE_GROUP_INPUT
  );
```

Then it gets:

```ts
let preimage = witness_args.lock!;
```

So my basic understanding is:

```text
Script args
    ↓
Expected hash


Witness.lock
    ↓
Preimage
"Hello World"
```

The Script uses these two pieces of information to decide whether the Cell can be spent.

---

### 4. The Lock Script hashes the provided preimage

The contract calculates:

```ts
let hash = hashCkb(preimage);
```

So when I provide:

```text
Hello World
```

the Script does:

```text
Hello World
     ↓
hashCkb()
     ↓
Actual Hash
```

It then compares this result with the expected hash stored in the Script args.

---

### 5. The Script decides whether the transaction is valid

The main verification is:

```ts
if (!bytesEq(hash, expect_hash.buffer)) {
  return 11;
} else {
  return 0;
}
```

So:

```text
hash(preimage) == expected hash
            ?
       /         \
     Yes          No
      ↓            ↓
  return 0      return 11
      ↓            ↓
  Success        Failed
```

`return 0` means the Lock Script verification succeeds.

A non-zero return value means the Script verification fails.

This is how the custom Lock Script controls whether the Cell can be consumed.

---

### 6. Lock Script args and Witness have different purposes

This exercise helped me understand the difference between them.

```text
Lock Script args
      ↓
Condition that must be satisfied
      ↓
Expected hash


Witness
      ↓
Proof provided when spending
      ↓
Preimage
```

Then the Lock Script verifies:

```text
hash(Witness preimage)
        ==
expected hash from Script args
```

This was one of the most important things I learned from this exercise.

---

## Result

| Item | Result |
| --- | --- |
| Contract | hash-lock |
| Build | Successful |
| Deployment | Successful |
| Preimage | Hello World |
| Deposit | 300 CKB |
| Lock | Custom Hash Lock |
| Unlock Condition | Correct preimage |
| Script Success | Return 0 |

---

## Main Takeaway

Before this exercise, I mainly thought of a Lock Script as something related to an address and private key.

After this exercise, I understand that a Lock Script is actually a program that defines the conditions required to consume a Cell.

A normal lock may verify a signature.

My hash lock verifies a preimage:

```text
Create Cell
────────────────────────

"Hello World"
      ↓
    hash
      ↓
Lock Script args
      ↓
[ Cell: 300 CKB ]


Spend Cell
────────────────────────

Witness.lock
"Hello World"
      ↓
   hashCkb()
      ↓
Actual Hash

      ==

Expected Hash
from Script args

      ↓
   return 0
      ↓
Cell can be spent
```

The important idea is:

**Lock Script defines the rule, and Witness provides the proof needed to satisfy that rule.**