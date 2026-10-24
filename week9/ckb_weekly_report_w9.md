# Week 9 --- CKB Credential Verification Infrastructure

**Scope:** Hardening, Credora differentiation, cross-family validation,
API/security review, developer extensibility, production QA, evidence
explainability, end-to-end validation, product differentiation, and
native wallet identity integration.

------------------------------------------------------------------------

## 1. Live Application

-   **Production Web App:**
    https://ckb-digital-credential-web.vercel.app/
-   **Public Verification Page:**
    https://ckb-digital-credential-web.vercel.app/verify
-   **Public Verification API:**
    https://ckb-digital-credential-service.onrender.com/api/verify
-   **Repository:** https://github.com/BTThang/ckbuilder

The application provides a public, sessionless credential-verification
flow. Verification is routed through a family-agnostic backend core and
ultimately relies on CKB chain state for the credential verdict.

------------------------------------------------------------------------

## 2. Week 9 Objective

Week 9 was an additive hardening and validation cycle rather than an
architectural rewrite.

Objectives: 1. Establish source-backed differentiation from Credora. 2.
Audit and harden the family-agnostic verification core. 3. Validate
Spore, Vellum Claim, and CKB-PoP. 4. Review the public API for security,
errors, abuse, and leakage. 5. Improve developer documentation and prove
extensibility. 6. Run production QA and end-to-end validation. 7.
Improve evidence and explainability. 8. Demonstrate multi-family
end-to-end behavior. 9. Document product differentiation. 10. Integrate
connected-wallet identity context natively without adding a wallet
dependency.

Rules: no unnecessary rewrite, no removed functionality, no fabricated
chain data, family-specific protocol logic stays in family
verifiers/readers, the public interface remains family-agnostic, and CKB
live state remains authoritative.

------------------------------------------------------------------------

## 3. Research & Credora Comparison --- RESEARCHED

Credora (`hiepthach/Credora_CKB`) was inspected across the Week 9
comparison dimensions.

Source-backed characterization: client-only Next.js app; no public API
routes; Spore-only verification; boolean `valid` plus `errors[]`;
localStorage/cache-first lookup; no `expectedOwner`; issuer/cluster
existence signal that does not determine validity; no transfer modeling
in the inspected verification path.

### Evidence-backed differentiators

1.  **Family-agnostic verification core:** Spore, Vellum `did:ckb`, and
    CKB-PoP through one interface and five-state result model.
2.  **Server-side, chain-authoritative verdict:** public HTTP
    verification reads live CKB state.
3.  **Five-state vocabulary:**
    `valid | invalid | not_found | unable_to_verify | unsupported`.
4.  **Live node as authority:** Vellum and CKB-PoP re-read live cells;
    indexer is discovery only.
5.  **Expected-owner verification:** owner comes from the live
    credential lock and is compared at script level.
6.  **Cryptographic issuer-proof signal:** issuer verification is
    modeled as a distinct evidence signal.
7.  **Public sessionless API:** `POST /api/verify` and
    `GET /api/verify/families`, with validation, rate limiting and
    timeout handling.
8.  **Extensibility without core rewrites:** additional families use the
    verifier/reader seam without new routes or response models.

Similarities are acknowledged: both use CKB/Spore concepts and an
on-chain credential object. Unsupported architectural claims about
Credora are not made.

------------------------------------------------------------------------

## 4. --- Verification Core Hardening --- IMPLEMENTED + TESTED

Two concrete defects were fixed.

### Defect 1 --- `hint` was dropped

`VerificationRequest.hint` was documented but not forwarded into the
core. The smallest additive fix forwards it through the controller,
verifier, and verification service. Regression tests cover present and
absent hints.

### Defect 2 --- malformed Spore IDs reached the chain

An explicit `family: "spore"` request with a malformed ID such as
`0xdead` could previously reach `findSpore`.

A shape guard now returns `invalid` before any provider call, matching
Vellum and CKB-PoP behavior.

Reviewed but deliberately unchanged: timeout fallback behavior, repeated
`sameScript` helper, existing rate-limit strategy, and other consistent
behavior without a concrete defect.

Observation retained: Spore's reader does not perform the same explicit
`getCellLive` re-read used by Vellum/CKB-PoP. This is documented as an
optional `SPORE-LIVE-CONFIRM` follow-up rather than an unnecessary
rewrite.

------------------------------------------------------------------------

## 5. --- Cross-Family Validation --- TESTED

A matrix covered valid, malformed, not-found, unsupported, RPC failure,
owner match, owner mismatch, API routing, UI result, and evidence
behavior.

`cross-family-matrix.test.ts`: **14/14 PASS**.

Live positives: - Vellum: PASS. - CKB-PoP: PASS. - Spore live positive
with valid issuer proof: NOT TESTED because no suitable live credential
was available.

All-family owner match/mismatch and secret non-leakage were tested
successfully.

------------------------------------------------------------------------

## 6. --- API & Security Review --- IMPLEMENTED + TESTED

The public verification endpoint remains intentionally unauthenticated.

The review covered validation, unknown keys, malformed JSON, oversized
input, unknown families, hostile hints, RPC failures, error
normalization, information leakage, secrets, rate limiting and timeouts.

**12 hardening checks passed.**

Key results: - strict-object validation; - 64 KB body limit; - malformed
JSON → `400`; - unknown family → `unsupported`; - hint is
non-authoritative; - RPC failure → `unable_to_verify`; - raw node errors
and stack traces are not exposed; - no environment values/secrets in
responses; - `unsupported` is distinct from `not_found`; - malformed
Spore IDs fail before RPC; - no tested path produces a false `valid`.

Rate limiting remains per-process and keyed by `req.ip` behind a proxy.

------------------------------------------------------------------------

## 7. --- Developer Documentation & Extensibility --- IMPLEMENTED

The `FamilyVerifier` contract, five-state vocabulary, add-a-family
walkthrough, testing pattern, consumer-vs-family distinction, and source
reading order were documented.

A future family requires a family tag, verifier, export, and
registration; routes, schema, response envelope, and state vocabulary
remain unchanged.

SkillPass is a documented hypothetical example only. It is **not
implemented or integrated**.

------------------------------------------------------------------------

## 8. --- Production QA --- LIVE-VALIDATED

The initial production API smoke matrix recorded **13/14 PASS**, with
the malformed Spore case showing old `not_found` behavior because Render
was still running the pre-Week-9 commit `9ebfb89`. The source fix was
already proven locally.

Other live checks passed: health, family discovery, Vellum, CKB-PoP,
Spore not-found, unsupported family, validation errors, malformed JSON,
unknown route, request ID, and CORS.

The production Vercel app served `/`, `/verify`, assets, and the real
API wiring. Interactive browser checks were initially limited by the
execution environment.

------------------------------------------------------------------------

## 9. --- Extensibility Proof --- PASS

A test-only fourth family (`mock-demo`) was added through the existing
verifier seam.

-   `src/test/mock-family.adapter.ts`
-   `src/test/phase8-extensibility.test.ts`
-   **9/9 tests PASS**

The public HTTP API was exercised after registration. No controller,
route, schema, response envelope, or common state vocabulary change was
required.

------------------------------------------------------------------------

## 10. --- Evidence & Explainability --- PASS

Added a family-neutral `evidenceSummary` as an additive response field.

Rules: - copy the verifier verdict; - derive only common facts; - never
guess owner, network, or output index; - unsupported/no-cell paths
retain null ownership/outpoint information.

The frontend renders an `EvidenceSummaryCard`. Family-specific evidence
remains authoritative.

------------------------------------------------------------------------

## 11. --- End-to-End Demo --- PASS

Validated multi-family scenarios including: - Vellum positive; - CKB-PoP
positive; - Spore not-found; - unsupported family; - owner match; -
owner mismatch; - malformed input; - public API routing; - SPA deep
links.

Local source was also run against real CKB Testnet.

------------------------------------------------------------------------

## 12. --- Product Differentiation --- PASS

Positioning:

> **Verification as infrastructure, not an app.**

Differentiators are: - HTTP-first integration; - demonstrated
extensibility; - explainable verdicts; - owner-scoped checks; -
five-state fail-closed verdicts; - chain-authoritative evidence.

No third-party Credora or SkillPass integration is claimed.

------------------------------------------------------------------------

## 13. --- Wallet Integration Research --- RESEARCHED

`ckb-connect-wallet-kit` was evaluated and **not integrated**.

The project already had CCC, CCC Connector, JoyID, `WalletProvider`,
connected-address state, and existing `expectedOwner` support. The
desired capability could therefore be implemented natively without
adding another wallet abstraction or runtime dependency.

Decision: retain the existing wallet architecture and implement the
required identity-to-owner flow directly.

------------------------------------------------------------------------

## 14. --- Native Wallet Identity Integration --- PASS

The connected wallet now feeds the existing `expectedOwner` path:

``` text
Connected Wallet
      ↓
Connected CKB Address
      ↓
Use connected wallet
      ↓
expectedOwner
      ↓
Existing Verification Core
      ↓
Owner MATCH / Owner MISMATCH
```

No backend, public API, verification-core, or family-verifier change was
required.

### Wallet behavior

-   Connected valid CKB address → `Use connected wallet` is available.
-   Wallet-derived owner follows wallet changes.
-   Disconnect clears wallet-derived owner.
-   Manual owner input is never silently overwritten.
-   Malformed wallet addresses are rejected before the API request.
-   No wallet signature or transaction is requested.

### DID profile

A read-only `did:ckb` profile was added to `VerifyPage`.

It: - resolves the connected address's own lock; - displays
informational profile data; - never affects verification or ownership; -
never enters the verification request; - does not sign or mutate
anything; - deliberately does not render the avatar.

### Automated tests

Frontend: - `verify-connected-wallet.test.tsx`: **17/17 PASS** -
`did-profile.test.ts`: **7/7 PASS** - `VerifyPage.test.tsx`: **15/15
PASS** - **158/158 frontend tests PASS across 19 files**

Backend: - **229/229 PASS across 16 files**

Typecheck, lint and build passed in both projects.

------------------------------------------------------------------------

## 15.  --- Live Testnet Validation --- PASS

Real CKB Testnet credentials were used.

  --------------------------------------------------------------------------------------------
  Family                  Identifier                                   Live owner
  ----------------------- -------------------------------------------- -----------------------
  Vellum                  `did:ckb:fn7u37m7vwerr4ojysgdwwp4mescjtrp`   `ckt1qzda0…kpmy`

  CKB-PoP                 `0xb5b0a980…73735`                           `ckt1qrfrw…crre`
  --------------------------------------------------------------------------------------------

Live API cases all passed:

  Case                            Result
  ------------------------------- ---------------------------
  Vellum without expected owner   `valid`
  Vellum + real owner             `valid` + `matches=true`
  Vellum + different owner        `valid` + `matches=false`
  CKB-PoP + real owner            `valid` + `matches=true`
  CKB-PoP + different owner       `valid` + `matches=false`
  Malformed expected owner        `400` validation error
  Mainnet owner on testnet        fails closed
  Unknown Vellum ID               `not_found`
  Unsupported family              `unsupported`

Critical invariant:

``` text
credential validity != ownership comparison
```

An owner mismatch does not make an otherwise valid credential invalid.

------------------------------------------------------------------------

## 16. Production Browser Validation --- PASS

After the production build was available, the verification UI was
manually tested.

The observed production flow confirmed:

1.  Vellum family selection.
2.  Real Vellum credential identifier.
3.  Connected wallet address shown.
4.  `Use connected wallet` fills the expected owner.
5.  Credential verifies as `VALID`.
6.  Evidence summary displays the on-chain owner.
7.  When the expected owner differs from the credential owner, the UI
    shows `MISMATCH` while the credential remains `VALID`.

Observed mismatch behavior:

``` text
Result: VALID

Current-owner check: MISMATCH

Held by a different address
The credential is held by a different address than the expected holder.
```

This confirms the intended separation between credential validity and
owner comparison in the browser UI.

------------------------------------------------------------------------

## 17. Final Quality Gate

### Backend --- `ckb-digital-credential-service`

  Check                 Result
  --------------------- -------------------------------
  `npm run typecheck`   PASS
  `npm run lint`        PASS
  `npm test`            **229/229 PASS --- 16 files**
  `npm run build`       PASS

### Frontend --- `ckb-digital-credential-web`

  Check                 Result
  --------------------- -------------------------------
  `npm run typecheck`   PASS
  `npm run lint`        PASS
  `npm test`            **158/158 PASS --- 19 files**
  `npm run build`       PASS

**Total automated regression tests: 387/387 PASS.**

------------------------------------------------------------------------

## 18. Final Architecture

``` text
                    Public Verify UI
                 React + Vite / Vercel
                          │
                          │ POST /api/verify
                          │ GET /api/verify/families
                          ▼
                  Verification API
                    Express / Render
                          │
                 validation / timeout
                 rate limit / errors
                          │
                          ▼
                 Verification Core
              family identification/routing
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
       Spore           Vellum           CKB-PoP
      Verifier         Verifier         Verifier
          │               │                │
        Reader          Reader           Reader
          └───────────────┼────────────────┘
                          ▼
                    CKB Testnet
                    Live Chain State
                          │
                          ▼
                 Common Verification
                 valid / invalid /
                 not_found /
                 unable_to_verify /
                 unsupported
```

Wallet identity is an additional context path:

``` text
Connected Wallet
      ↓
CKB Address
      ↓
expectedOwner
      ↓
Ownership comparison
      ↓
MATCH / MISMATCH
```

DID profile is deliberately outside the verification path:

``` text
Connected Address
      ↓
did:ckb lookup
      ↓
Read-only profile
      ↓
Informational only
```

------------------------------------------------------------------------

## 19. Trust & Security Model

-   Credential validity is decided by family-specific verification
    against authoritative CKB state.
-   Ownership is derived from the credential's live lock and compared
    against `expectedOwner`.
-   `expectedOwner` is a comparison target, not proof of control.
-   Wallet integration reads only the public CKB address.
-   No private key, seed, session secret, or signing capability is
    exposed.
-   No transaction is created, signed, broadcast, issued, transferred,
    or melted by Phase 9.6.
-   DID profile data is self-asserted informational data and cannot
    influence a verdict.
-   Index/cache data is not treated as the final authority for live
    credential validity.
-   Public API remains sessionless by design.

------------------------------------------------------------------------

## 20. Remaining Limitations

1.  No live Spore credential with a valid issuer proof is currently
    available for a positive production test.
2.  Automated browser-wallet/extension testing is unavailable; the Phase
    9.6 browser flow was manually validated after the production build.
3.  Mainnet is not a deployment target for the current project.
4.  Rate limiting remains per-process and proxy/IP based.
5.  SQLite remains a single-node limitation.
6.  Spore live-cell re-read asymmetry remains a documented optional
    hardening item.
7.  There is no issuer registry/trust anchor; the infrastructure proves
    chain state and ownership, not issuer reputation.
8.  DID profile parsing currently focuses on the document-root fields
    used by the real fixtures.
9.  DID avatars are deliberately not rendered.
10. DID profile is currently shown on `VerifyPage` only.
11. Frontend/backend network parity is configuration-dependent and
    currently fails closed if mismatched.

------------------------------------------------------------------------

## 21. Evidence

Production: - https://ckb-digital-credential-web.vercel.app/ -
https://ckb-digital-credential-web.vercel.app/verify -
https://ckb-digital-credential-service.onrender.com/api/verify -
https://github.com/BTThang/ckbuilder

------------------------------------------------------------------------

## 22. Final Verdict

# PASS --- WITH DOCUMENTED LIMITATIONS

Week 9 successfully strengthened the project from a multi-family
credential verifier into a more defensible **CKB Credential Verification
Infrastructure**.

The key achievements are:

-   source-backed differentiation from Credora;
-   two concrete verification-core fixes;
-   three-family verification through one common interface;
-   CKB chain-authoritative verification;
-   fail-closed five-state results;
-   public API security hardening;
-   extensibility proven without API rewrites;
-   family-neutral evidence and explainability;
-   end-to-end multi-family validation;
-   native connected-wallet → `expectedOwner` integration;
-   owner MATCH and MISMATCH validated against real CKB Testnet
    credentials;
-   read-only `did:ckb` identity context;
-   production browser validation of the wallet/owner flow;
-   no new wallet dependency;
-   no signing, transaction, secret, or credential mutation introduced.

Final automated gate:

``` text
Backend:   229 / 229 PASS
Frontend:  158 / 158 PASS
-------------------------
Total:     387 / 387 PASS
```

**WEEK 9 STATUS: PASS**
