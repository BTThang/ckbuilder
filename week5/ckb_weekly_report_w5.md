# CKB Weekly Report — Week 5

------------------------------------------------------------------------

## Project Links

- **Frontend Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp
- **Backend Service Repository:** https://github.com/BTThang/btt-ckb-learning-dapp/tree/main/btt-ckb-learning-dapp-service

------------------------------------------------------------------------

## Topic: Transaction Lifecycle and Spore Operations

### 1. Overview

This week focused on understanding and implementing the transaction lifecycle in a Nervos CKB dApp, with particular attention to Spore operations.

The main objectives were:

- Create a Spore.
- Transfer a Spore.
- Melt a Spore.
- Broadcast transactions through the connected JoyID wallet.
- Track transaction status from `pending` to `committed`.
- Display application-level transaction history.
- Persist transaction history with `localStorage`.
- Keep the application functional when the backend service is unavailable.
- Verify the complete Spore lifecycle through an end-to-end test.

---

## 2. Transaction Lifecycle

A CKB transaction does not become committed immediately after it is broadcast.

The implemented flow is:

```text
Build Transaction
       ↓
Complete Inputs / Fee
       ↓
Sign Transaction
       ↓
Broadcast Transaction
       ↓
Pending
       ↓
Committed
```

The application now polls the CKB RPC endpoint after broadcasting a transaction.

The polling logic checks the transaction status repeatedly. When the status becomes `committed`, the application updates the local transaction history and refreshes blockchain data.

The polling loop performs up to 20 checks with a 3-second interval between checks.

### Key implementation

```tsx
async function waitForTransactionStatus(txHash: string) {
  if (!signer) return;

  for (let i = 0; i < 20; i++) {
    try {
      const txInfo = await signer.client.getTransaction(txHash);
      const status = txInfo?.status;

      console.log(`Transaction check ${i + 1}:`, status);

      if (status === "committed") {
        setAppTxHistory((prev) =>
          prev.map((item) =>
            item.txHash === txHash
              ? { ...item, status: "committed" }
              : item,
          ),
        );

        await refreshBlockchainData();
        return;
      }
    } catch (error) {
      console.error(`Transaction check ${i + 1} failed:`, error);
    }

    await new Promise((resolve) => setTimeout(resolve, 3000));
  }

  setAppTxHistory((prev) =>
    prev.map((item) =>
      item.txHash === txHash
        ? { ...item, status: "pending" }
        : item,
    ),
  );
}
```

This demonstrates the difference between transaction submission and blockchain confirmation.

---

## 3. Create Spore

The Spore creation flow was extended to record the transaction in the application history.

The process is:

1. Create the Spore transaction.
2. Complete transaction inputs by capacity.
3. Complete the transaction fee.
4. Broadcast the transaction through the connected signer.
5. Add the transaction to application history with `submitted` status.
6. Query the transaction status.
7. Poll until the transaction becomes `committed`.
8. Refresh blockchain data.

### Evidence

### Figure 1 — App Transaction History

![Figure 1 — App Transaction History](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%201%20%E2%80%94%20App%20Transaction%20History.png)

This screenshot shows the application-level transaction history containing Spore operations, including Create, Transfer, and Melt transactions.

### Figure 2 — Console – Create Spore

![Figure 2 — Console – Create Spore](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%202%20%E2%80%94%20Console%20%E2%80%93%20Create%20Spore.png)

The console shows the Spore creation process, including the Spore ID and transaction information.

---

## 4. Transfer Spore

A transfer operation was implemented using the Spore ID and a receiver CKB address.

The receiver address is converted into a CKB lock script before being passed to the Spore SDK.

### Implementation flow

```tsx
const { script: receiverLock } =
  await ccc.Address.fromString(
    sporeReceiver,
    signer.client,
  );

const { tx } = await spore.transferSpore({
  signer,
  id: sporeId,
  to: receiverLock,
});

await tx.completeInputsByCapacity(signer);
await tx.completeFeeBy(signer);

const txHash = await signer.sendTransaction(tx);
```

After broadcasting, the transaction is added to the application history and monitored until it reaches `committed`.

### Evidence

### Figure 3 — Console – Transfer Spore

![Figure 3 — Console – Transfer Spore](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%203%20%E2%80%94%20Console%20%E2%80%93%20Transfer%20Spore.png)

The console demonstrates the transaction lifecycle changing from `pending` to `committed`.

### Figure 5 — Blockchain Explorer – Transfer

![Figure 5 — Blockchain Explorer – Transfer](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%205%20%E2%80%94%20Blockchain%20Explorer%20%E2%80%93%20Transfer.png)

The blockchain explorer provides external evidence of the transfer transaction, including the transaction hash and block information.

---

## 5. Melt Spore

The Melt operation was implemented using the Spore ID.

The successful implementation uses explicit transaction signing and submission:

```tsx
const { tx } = await spore.meltSpore({
  signer,
  id: sporeId,
});

await tx.completeFeeBy(signer);

const signedTx = await signer.signTransaction(tx);

const txHash = await signer.client.sendTransaction(signedTx);
```

The transaction is then monitored using the same pending-to-committed polling mechanism.

During testing, ownership was also verified as an important part of the Spore lifecycle. A Spore can only be melted by the wallet that owns the Spore. After transferring a Spore to another wallet, attempting to melt it from the original wallet resulted in a JoyID witness/signing error. The successful end-to-end test therefore used the currently connected wallet as the Spore owner.

### Evidence

### Figure 4 — Console – Melt Spore

![Figure 4 — Console – Melt Spore](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%204%20%E2%80%94%20Console%20%E2%80%93%20Melt%20Spore.png)

The console demonstrates the Melt transaction progressing from `pending` to `committed`.

### Figure 6 — Blockchain Explorer – Melt

![Figure 6 — Blockchain Explorer – Melt](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%206%20%E2%80%94%20Blockchain%20Explorer%20%E2%80%93%20Melt.png)

The blockchain explorer provides external evidence of the Melt transaction, including the transaction hash and block information.

---

## 6. Application Transaction History

An application-level transaction history was added to make Spore operations easier to track from the dApp UI.

Each record contains:

```tsx
{
  txHash: string;
  type: string;
  timestamp: string;
  status: string;
  detail?: string;
}
```

Example records include:

- `Spore` — `Create Spore`
- `Spore` — `Transfer Spore`
- `Spore` — `Melt Spore`

The history also prevents duplicate transaction records by checking whether the transaction hash already exists.

```tsx
const exists = prev.some(
  (item) => item.txHash === txHash
);

if (exists) return prev;
```

Blockchain transaction history was also updated to collect transaction records, sort them by block number, and display the newest transactions first.

---

## 7. LocalStorage Persistence

The application transaction history is persisted using browser `localStorage`.

The initial state loads previously saved records:

```tsx
const [appTxHistory, setAppTxHistory] = useState(() => {
  const savedHistory = localStorage.getItem("appTxHistory");
  return savedHistory ? JSON.parse(savedHistory) : [];
});
```

Whenever the history changes, it is stored again:

```tsx
useEffect(() => {
  localStorage.setItem(
    "appTxHistory",
    JSON.stringify(appTxHistory),
  );
}, [appTxHistory]);
```

This means the application history remains available after refreshing the browser.

### Evidence

### Figure 7 — App after Refresh

![Figure 7 — App after Refresh](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%207%20%E2%80%94%20App%20after%20Refresh.png)

The application history remains visible after refreshing the browser, demonstrating that the transaction records were persisted locally.

---

## 8. Backend Failure Isolation

The application was also tested with the backend service stopped.

The frontend remained usable and blockchain-related operations continued to work because transaction status and blockchain data are obtained directly through the CKB client/RPC where required.

The backend request produced a fetch error in the console, but this did not prevent the main application interface from functioning.

### Evidence

### Figure 8 — Backend Stopped + App Still Works

![Figure 8 — Backend Stopped + App Still Works](https://raw.githubusercontent.com/BTThang/ckbuilder/blob/main/week5/images/Figure%208%20%E2%80%94%20Backend%20Stopped%20%2B%20App%20Still%20Works.png

This demonstrates that the frontend can remain functional even when the backend service is unavailable.

---

## 9. End-to-End Spore Test

A complete Spore lifecycle was tested:

```text
Create Spore
    ↓
Transfer Spore
    ↓
Wait for committed
    ↓
Melt Spore
    ↓
Wait for committed
```

The successful test used the same connected wallet as the owner for the final Melt operation.

### Transfer transaction

- Spore ID:
  `0x8171f6cc4a7e177c93893be1698513a48c8c7f53c309478c6eaff38a9f4de146`
- Transfer TX:
  `0x0029a806148535bebe0e24a0657df9e72643dabc6413d00313511ad48562c24c`
- Final observed status: `committed`

### Melt transaction

- Melt TX:
  `0xa4fda5d63646c54990ca5ccf752b082f8ec790a57bf40be75990b7880c518288`
- Final observed status: `committed`

The transfer transaction reached `committed` during polling, and the Melt transaction also reached `committed`.

---

## 10. Week 5 Checklist

| Task | Status |
|---|---|
| Create Spore | ✅ Completed |
| Transfer Spore | ✅ Completed |
| Melt Spore | ✅ Completed |
| Broadcast transactions | ✅ Completed |
| Track `pending → committed` | ✅ Completed |
| Application transaction history | ✅ Completed |
| LocalStorage persistence | ✅ Completed |
| Duplicate transaction protection | ✅ Completed |
| Newest blockchain transactions first | ✅ Completed |
| RPC polling error handling | ✅ Completed |
| Backend failure isolation | ✅ Completed |
| JoyID transaction signing | ✅ Completed |
| Create → Transfer → Melt E2E test | ✅ Completed |
| Browser refresh persistence test | ✅ Completed |

---

## 11. Key Learnings

### 11.1 Transaction submission is different from confirmation

Calling `sendTransaction()` only broadcasts a transaction. The transaction still needs to be confirmed by the CKB network.

Therefore, the application should distinguish states such as:

```text
submitted
pending
committed
```

### 11.2 Transaction hashes are useful application identifiers

The transaction hash is used to:

- Identify a transaction.
- Prevent duplicate history records.
- Poll transaction status.
- Display transaction information to the user.

### 11.3 Spore ownership matters

Spore operations are tied to the owner of the Spore. A wallet that does not control the current Spore ownership cannot successfully perform a Melt operation.

### 11.4 Frontend state and blockchain state are different

The application maintains its own transaction history for user experience, while the blockchain remains the source of truth for transaction status.

The application therefore synchronizes its UI state with CKB RPC responses.

### 11.5 LocalStorage improves the user experience

Persisting the application history locally prevents the UI history from disappearing after a browser refresh.

---

## 12. Conclusion

Week 5 focused on connecting application-level transaction handling with actual CKB transaction confirmation.

The dApp now supports the main Spore lifecycle:

```text
Create → Transfer → Melt
```

It also provides transaction status tracking, application transaction history, local persistence, duplicate protection, and basic resilience when the backend service is unavailable.

The end-to-end test confirmed that the implemented Spore operations can be broadcast and monitored until they reach the `committed` state on the CKB network.
