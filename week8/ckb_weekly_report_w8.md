# Week 8 — CKB Credential Verification Infrastructure
## 1. Live Application

Production application:

https://ckb-digital-credential-web.vercel.app/

Public verification:

https://ckb-digital-credential-web.vercel.app/verify

---

## 2. Week 8 Objective

Week 8 focused on completing and validating the credential verification infrastructure across the supported credential families, with emphasis on:

- CKB-PoP live verification against CKB Testnet.
- Vellum Claim (`did:ckb`) verification using a real identifier.
- Production/public verification flow validation.
- Owner verification and negative-case handling.
- Regression safety across the existing verification core.
- Final documentation and hardening.

The implementation continued the existing family-agnostic verification architecture rather than introducing a separate verification system for each credential family.

---

## 2. Research — 4 Related CKB Projects

Week 8 included a focused research review of four related projects to understand how different CKB credential/identity models can be represented and verified through a common verification infrastructure.

### 2.1 Vellum

**Project:** Vellum / Vellum Claim

Key research findings:

- Vellum Claim uses a `did:ckb` identifier.
- Verification resolves the DID to its associated live CKB DID Metadata Cell.
- The chain state is the authoritative source for the current credential/identity state.
- The verification flow therefore follows an identifier → DID metadata → live chain state model.
- The existing implementation treats the Vellum path as a dedicated family-specific verifier behind the common verification core.

**Integration relevance:** Vellum demonstrates a credential family where the identifier itself is a DID reference and verification requires resolving associated on-chain metadata rather than treating the identifier as a standalone credential record.

### 2.2 SkillPass

**Project:** SkillPass

The SkillPass repository was reviewed to understand its credential issuance and verification model, identifier structure, on-chain representation, and how its credential data can be interpreted independently from the application UI.

The research focused on:

- credential structure and claims;
- issuer/holder relationship;
- identifier and verification inputs;
- CKB/on-chain components involved in credential state;
- what information can be treated as authoritative during verification;
- implications for supporting SkillPass as another credential family.

**Integration relevance:** SkillPass reinforces the need to keep protocol-specific parsing and chain-resolution logic inside a dedicated family verifier rather than coupling the public verification API to one credential format.

### 2.3 CKB-PoP

**Project:** CKB-PoP

Sources inspected included the official CKB-PoP repository, the locally available `ckb-pop-kit@0.2.0`, protocol documentation, and live CKB Testnet state.

Key findings:

- The `dob-badge` is represented by a CKB type script.
- The badge identifier is 60 bytes / 120 hexadecimal characters.
- The identifier consists of:
  - `type_id` — 20 bytes
  - `scope_id_hash` — 20 bytes
  - `recipient_address_hash` — 20 bytes
- Indexer discovery can locate candidate cells, but live chain state is used for the final verification decision.
- Cell data/metadata is not authoritative for the credential verdict.
- The verifier should fail closed when the required deployment configuration is unavailable.

A real Testnet badge was subsequently used to validate the implementation against live chain state.

**Integration relevance:** CKB-PoP is a strong example of why the verification infrastructure must separate discovery from authoritative chain verification.

### 2.4 Credora

**Project:** Credora CKB

The Credora repository was reviewed to understand its credential/data model and its relationship to CKB-based verification.

The research focused on:

- credential structure;
- issuer and holder concepts;
- identifier representation;
- on-chain state/data involved in verification;
- verification evidence that can be independently checked;
- compatibility with a family-agnostic verification architecture.

**Integration relevance:** Credora provides another protocol-specific credential model that can be isolated behind the same common verification interface without requiring changes to the public verification API for every new credential family.

### 2.5 Cross-project comparison

| Project | Credential / identity model | Verification focus | Main architectural implication |
|---|---|---|---|
| Vellum | `did:ckb` Claim | Resolve DID Metadata Cell and verify live state | Identifier resolution must be family-specific |
| SkillPass | Skill/credential model | Claims, issuer/holder and CKB representation | Protocol parsing should remain isolated |
| CKB-PoP | `dob-badge` type-script credential | Type script + live cell state | Indexer is discovery; live chain is authoritative |
| Credora | CKB-based credential/data model | Credential evidence and on-chain state | New families should plug into the same core |

### 2.6 Research conclusion

The four projects do not share one identical credential representation or verification procedure. The common requirement is that verification must translate a credential-specific identifier into authoritative evidence.

This supports the Week 8 architecture:

```text
Public Verification API
        ↓
Family Router
        ↓
Family-specific Verifier
        ↓
Family-specific Reader / Resolver
        ↓
CKB authoritative state
```

The research therefore informed the decision to keep the verification core family-agnostic while isolating protocol-specific rules in individual verifiers/readers.

---

## 3. Credential Verification Research

### 2.1 CKB-PoP

The CKB-PoP implementation was based on investigation of the CKB-PoP project and its available source/package material.

Implementation-relevant facts established:

- The CKB-PoP `dob-badge` is identified through a type script.
- The badge identifier is 60 bytes / 120 hexadecimal characters:
  - `type_id` — 20 bytes
  - `scope_id_hash` — 20 bytes
  - `recipient_address_hash` — 20 bytes
- Indexer discovery is used to locate candidate cells; live node state is used for the final verification decision.
- Cell data/metadata is not treated as authoritative for the credential verdict.
- The implementation fails closed when required deployment configuration is unavailable.
- A real CKB Testnet `dob-badge` deployment was successfully used for live validation.

### 2.2 Vellum Claim

Vellum Claim verification was investigated as a `did:ckb`-based credential flow.

The implemented flow is:

```text
Vellum Claim (did:ckb)
        ↓
did:ckb identifier
        ↓
DID Metadata Cell
        ↓
Live CKB chain state
        ↓
Verification result
```

The Vellum flow was tested using a real `did:ckb` identifier rather than a fabricated credential identifier.

---

## 4. Implementation

### 3.1 Verification architecture

The project retains a common verification core with family-specific verifiers/readers:

```text
Public UI
   ↓
Public Verification API
   ↓
Family Router
   ↓
Credential-family verifier
   ↓
Chain reader
   ↓
CKB live state
```

This keeps credential-family-specific protocol logic isolated while allowing the public verification API and UI to remain family-agnostic.

### 3.2 CKB-PoP

CKB-PoP verification was implemented behind the existing verification-core architecture.

Implemented capabilities include:

- CKB-PoP family registration.
- 60-byte identifier validation.
- CKB-PoP reader.
- Type-script construction.
- Indexer discovery.
- Live-cell verification.
- Owner derivation from the live cell lock.
- Mapping of verification outcomes to common verification states.
- Public API integration.
- Public UI family selection and verification.
- Expected-owner comparison.

Fail-closed behavior:

| Condition | Result |
|---|---|
| Malformed identifier | `invalid` |
| Credential not found | `not_found` |
| Missing required deployment configuration | `unsupported` |
| Chain/RPC failure | unavailable/error state |
| Matching live credential | `valid` |

### 3.3 Vellum Claim

Vellum Claim remains integrated into the same public verification infrastructure.

The public UI exposes:

```text
Vellum claim (did:ckb)
did:ckb identifier
Expected owner address (optional)
```

The expected-owner field is an optional comparison. The authoritative owner is read from chain state.

---

## 5. Live Testnet / Production Validation

### 4.1 CKB-PoP live validation

A real CKB-PoP badge was validated against CKB Testnet.

Observed deployment:

```text
Network: CKB Testnet
hash_type: type
```

Observed `dob-badge` code hash:

```text
0xb36ed7616c4c87c0779a6c1238e78a84ea68a2638173f25ed140650e0454fbb9
```

Real badge identifier used:

```text
0xb5b0a980df2698c1bd3298d3625960566c2c6aa5659cd06bc027c893783eadf6c72f02083a18ba7872f60b7ae510c86b1cf0e0d3c184e96644f73735
```

The live badge resolved successfully and the public verification API returned:

```text
supported = true
routed    = true
state     = valid
valid     = true
```

The evidence included the live owner, badge arguments, outpoint, network and cell data.

### 4.2 CKB-PoP negative cases

| Case | Expected result | Result |
|---|---|---|
| Real badge identifier | `valid` | PASS |
| Random valid-format identifier | `not_found` | PASS |
| Malformed identifier | `invalid` | PASS |
| Missing/unsupported deployment configuration | `unsupported` | PASS |

### 4.3 Vellum Claim live validation

A real Vellum `did:ckb` identifier was tested through the production/public verification interface.

The test used:

```text
Credential family: Vellum claim (did:ckb)
```

The identifier was accepted by the UI and the verification flow completed successfully.

The optional expected-owner comparison was also exercised.

Result:

```text
Vellum live verification: PASS
```

This demonstrates that the Vellum Claim path was exercised through the deployed public verification interface using a real `did:ckb` identifier.

---

## 6. Public Verification UI

The public verification page supports multiple credential families through one verification interface.

Flow:

```text
Credential family
        ↓
Credential identifier
        ↓
Optional expected owner
        ↓
Verify
        ↓
Result
        ↓
Evidence
```

The result view exposes:

- verification state
- identifier
- family
- verification timestamp
- routing status
- support status
- evidence
- current-owner comparison where applicable

Identifier validation occurs before requests are sent to the verification API.

---

## 7. Owner Verification

The optional owner-check flow was validated separately from credential existence.

Model:

```text
Credential verification
        +
Expected owner
        ↓
Chain owner vs expected owner
        ↓
MATCH / MISMATCH
```

A credential can therefore remain valid while an optional expected-owner comparison reports a mismatch.

The owner comparison does not replace the authoritative chain owner.

---

## 8. Regression and Quality Gates

### Backend

Latest Week 8 validation:

- TypeScript typecheck: PASS
- ESLint: PASS
- Vitest: PASS
- Production build: PASS

Backend:

```text
146 tests passed
12 test files
0 failed
```

### Frontend

Latest Week 8 validation:

- TypeScript typecheck: PASS
- ESLint: PASS
- Vite production build: PASS
- Vitest: PASS

Frontend:

```text
133 tests passed
17 test files
0 failed
```

### Combined

```text
Backend:  146 passed
Frontend: 133 passed
----------------------
Total:    279 passed
```

Existing Spore/Vellum behavior was preserved while CKB-PoP functionality was integrated.

---

## 9. Production Deployment / Runtime Validation

The deployed public application was used to validate the verification flows.

Validated areas included:

- public verification routing
- credential-family selection
- CKB-PoP verification
- Vellum Claim verification
- expected-owner comparison
- verification result rendering
- evidence rendering
- negative verification states

A production configuration issue was identified during validation: CKB-PoP production verification requires the testnet `CKB_POP_DOB_BADGE_CODE_HASH`.

After the configuration was corrected, the CKB-PoP live verification path was successfully validated against the real Testnet badge.

---

## 10. Security / Hardening

The verification infrastructure follows these principles:

- Public verification endpoints do not require a user session.
- Malformed identifiers are rejected before unnecessary chain access.
- Verification decisions use live chain state.
- Indexer data is discovery information, not final authority.
- Missing protocol configuration fails closed.
- Optional owner comparison does not replace the authoritative chain owner.
- Protocol-specific logic remains isolated in dedicated verifiers/readers.
- Existing authenticated issuer/holder workflows remain separate from public verification.
- Private credentials and secrets are not committed.

---

## 11. Known Limitations

### 10.1 CKB-PoP deployment metadata

The CKB-PoP testnet code hash used for live validation was observed directly from a real Testnet cell.

No authoritative deployment manifest containing the same value was located in the inspected project sources.

Therefore the report distinguishes between:

```text
Observed live deployment
```

and:

```text
Official published deployment manifest
```

The observed deployment was sufficient for live verification, while the official-manifest/source limitation remains documented.

### 10.2 Family inference

The current public API requires the credential family for routing when family inference is unavailable.

Therefore an identifier supplied without a family hint may return:

```text
unsupported
```

rather than automatically identifying its credential family.

This is a current design limitation rather than a failure of the family-specific verifier.

### 10.3 Metadata authority

For CKB-PoP, cell metadata/data is returned as evidence/context but is not treated as authoritative for the verification verdict.

Live cell state and protocol-specific script/identifier conditions remain authoritative.

---

## 12. Final Week 8 Result

### Overall status: PASS

Week 8 completed the intended verification-infrastructure work and validated the important paths through the deployed public application.

| Validation | Result |
|---|---|
| CKB-PoP live Testnet verification | PASS |
| CKB-PoP `valid` result | PASS |
| CKB-PoP `not_found` result | PASS |
| CKB-PoP malformed identifier | PASS |
| Vellum Claim (`did:ckb`) live verification | PASS |
| Expected-owner comparison | PASS |
| Public verification UI | PASS |
| Public verification API | PASS |
| Backend automated tests | 146/146 PASS |
| Frontend automated tests | 133/133 PASS |
| Typecheck | PASS |
| Lint | PASS |
| Production build | PASS |

The resulting system provides a common public verification infrastructure capable of routing and verifying supported CKB credential families while keeping protocol-specific verification logic isolated and chain-authoritative.

---


## 14. Week 8 Conclusion

The main objective of Week 8 was not only to implement verifier code, but to demonstrate that the verification infrastructure can operate against real CKB chain state through the deployed public application.

The CKB-PoP path was validated against a real CKB Testnet badge, while the Vellum Claim path was validated using a real `did:ckb` identifier.

The implementation therefore moves beyond a purely local/mock verification architecture toward a practical, public, chain-authoritative credential verification infrastructure.

The remaining limitations are primarily related to deployment metadata, family inference and protocol-specific source availability rather than failures in the core verification flow.
