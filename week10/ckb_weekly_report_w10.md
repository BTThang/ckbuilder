# CKB Credential Verification Infrastructure --- Week 10 Report

**Scope:** Production readiness, live verification, dynamic family
discovery, wallet validation, mobile UX and DID Profile\
**Follow-up validation:** 2026-10-29 to 2026-10-31\
**Network:** CKB Testnet\
**Final verdict:** **PASS**

------------------------------------------------------------------------

## 1. Executive Summary

Week 10 validated the CKB Credential Verification Infrastructure against
CKB Testnet and the production deployment. Work included production API
regression, `evidenceSummary`, a real Spore positive path, dynamic
Family discovery, security checks, quality gates, wallet interaction,
mobile responsiveness and DID Profile behaviour.

The project owner subsequently confirmed that: - The Week 10 frontend
commit was deployed to Vercel Production. - `GET /api/verify/families`
returned the expected three families. - The deployed selector displayed
and allowed selection of Spore, Vellum Claim and CKB-PoP. - Production
credential verification flows passed. - Real-wallet production checks
passed for connection, MATCH, MISMATCH, account switching and
disconnect. - Final mobile/header UI fixes were deployed and passed: 1.
Wallet dropdown clipping on mobile was fixed. 2. Disconnect Wallet is
available while connected even when the user is not logged in. 3. The
redundant separate JoyID/address badge was removed; the clickable wallet
control remains the entry point for wallet information and disconnect.

Backend and frontend quality gates passed with **229 backend tests and
162 frontend tests (391 total)**. Interactive production wallet and
final UI results are owner-reported manual validations, not automated
browser test results.

**Final verdict: PASS.** Remaining evidence limitations are documented
in Section 14.

------------------------------------------------------------------------

## 2. Objectives

-   Confirm production deployment and behaviour.
-   Run a production API regression matrix.
-   Validate `evidenceSummary` in production.
-   Find and verify a real CKB Testnet Spore credential with a valid
    issuer proof.
-   Validate credential validity independently from ownership
    comparison.
-   Validate wallet-derived `expectedOwner`.
-   Regress read-only `did:ckb` profile behaviour.
-   Run production-safe security checks.
-   Make the frontend Family selector consume
    `GET /api/verify/families`.
-   Run backend and frontend quality gates.
-   Smoke-test deployed web routes and credential results.
-   Validate mobile UI and wallet interactions.
-   Record evidence and limitations without inventing chain or
    deployment data.

------------------------------------------------------------------------

## 3. Production Deployment and Repository State

### 3.1 Production endpoints

  -----------------------------------------------------------------------------------------------
  Item                                Value / result
  ----------------------------------- -----------------------------------------------------------
  Backend API                         `https://ckb-digital-credential-service.onrender.com/api`

  Frontend                            `https://ckb-digital-credential-web.vercel.app/`

  Verify page                         `https://ckb-digital-credential-web.vercel.app/verify`

  Network                             CKB Testnet

  `GET /api/verify/families`          HTTP 200; 3 families

  Families                            `spore`, `vellum-claim`, `ckb-pop`
  -----------------------------------------------------------------------------------------------

Family descriptions returned by the API: - `spore`: Spore credential
cells; CKB cell state plus Phase 1 cryptographic issuer proof. -
`vellum-claim`: Vellum claim cells verified against chain-authoritative
claim state. - `ckb-pop`: CKB-PoP participation badges verified against
chain-authoritative dob-badge cells.

### 3.2 Repository and deployment identity

  -------------------------------------------------------------------------------------------------------------------
  Repository                         Branch            Commit                                       Working tree
  ---------------------------------- ----------------- -------------------------------------------- -----------------
  Backend                            `main`            `a3cac5b01c9eace3e9af4eae552b32e0504822e4`   Clean
  `ckb-digital-credential-service`                     (`w9`)                                       

  Frontend                           `main`            `da67df4f6daeb6bb3160a692b7566825f20e2c00`   Clean at
  `ckb-digital-credential-web`                         (`w10`)                                      2026-10-29
                                                                                                    repository check
  -------------------------------------------------------------------------------------------------------------------

The frontend commit was confirmed as `origin/main` and contained: -
`src/pages/VerifyPage.tsx` --- dynamic Family discovery and non-Spore
deep-link restoration. - `src/test/verify-family-discovery.test.tsx` ---
discovery and deep-link tests.

### 3.3 Vercel deployment evidence

The deployed frontend commit was verified through public GitHub
Deployments metadata:

  Field                   Value
  ----------------------- --------------------------------------------
  Deployment ID           `7402725308`
  Ref / SHA               `da67df4f6daeb6bb3160a692b7566825f20e2c00`
  Environment             `Production`
  Deployment created      `2026-10-29T02:48:39Z`
  Vercel status           `success` --- Deployment has completed
  Commit status context   `Vercel` --- `success`

This SHA matched frontend repository `HEAD` and `origin/main`. Runtime
HTTP interfaces do not expose the commit SHA; deployment metadata is the
verification source. Backend health exposed `version: 1.0.0`.

### 3.4 Owner-reported deployment checks

After deploying the latest frontend commit, the project owner
confirmed: - `GET /api/verify/families` returned the expected three
families. - The deployed frontend displayed and allowed selection of all
three. - Production credential verification flows worked correctly.

These are owner-reported manual production checks.

------------------------------------------------------------------------

## 4. Production API Regression Matrix

The following 17 cases were tested against production
`POST /api/verify`, unless otherwise noted.

  -------------------------------------------------------------------------------------------------------------
              \# Case                             HTTP State           `valid`    Expected behaviour / result
  -------------- ---------------------- -------------- --------------- ---------- -----------------------------
               1 Vellum valid                      200 `valid`         `true`     Chain-derived owner and
                                                                                  outPoint returned

               2 Vellum with correct               200 `valid`         `true`     `ownership.matches: true`
                 `expectedOwner`                                                  

               3 Vellum with incorrect             200 `valid`         `true`     `ownership.matches: false`;
                 `expectedOwner`                                                  different-holder reason

               4 CKB-PoP valid                     200 `valid`         `true`     Chain-derived owner and
                                                                                  outPoint returned

               5 CKB-PoP with correct              200 `valid`         `true`     `ownership.matches: true`
                 `expectedOwner`                                                  

               6 CKB-PoP with incorrect            200 `valid`         `true`     `ownership.matches: false`
                 `expectedOwner`                                                  

               7 Spore not found                   200 `not_found`     `false`    No cell; no owner or outPoint
                 (`0xab` repeated 32                                              
                 bytes)                                                           

               8 Malformed Spore ID                200 `invalid`       `false`    Shape validation rejects
                 (`0xdead`)                                                       before RPC

               9 Malformed generic ID              200 `unsupported`   `false`    No registered verifier
                 (`not-a-credential`)                                             recognises identifier

              10 Unsupported family                200 `unsupported`   `false`    Unsupported family reported
                 (`skillpass`)                                                    safely

              11 Unknown Vellum ID                 200 `not_found`     `false`    No matching claim cell

              12 Malformed                         400 n/a             n/a        `VALIDATION_ERROR`; no RPC
                 `expectedOwner`                                                  

              13 Mainnet owner supplied            200 `valid`         `true`     Credential remains valid;
                 on Testnet                                                       ownership unresolved/no false
                                                                                  match

              14 Malformed JSON                    400 n/a             n/a        `BAD_REQUEST`

              15 Oversized request                 413 n/a             n/a        `PAYLOAD_TOO_LARGE`

              16 Hostile/unknown hint              200 `unsupported`   `false`    Hint not reflected; safe
                                                                                  response

              17 No family supplied                200 `unsupported`   `false`    No implicit family inference
                 with `did:ckb`                                                   
                 identifier                                                       
  -------------------------------------------------------------------------------------------------------------

### 4.1 Security notes

-   A bounded `<script>alert(1)</script>` hint was not reflected in the
    response.
-   A separate more hostile SQL/path-traversal-like input was blocked by
    the Render edge WAF with HTML `403` before reaching the application.
-   Checked responses contained no stack traces, raw RPC errors,
    environment variables, secrets, internal filesystem paths or false
    `valid` results.
-   No destructive load testing or intentional production rate-limit
    exhaustion was performed.
-   Rate-limit `429` and recovery behaviour are covered by offline
    tests.

------------------------------------------------------------------------

## 5. Evidence Summary Validation

Production returned `evidenceSummary` on applicable inspected responses,
including `valid`, `not_found`, `invalid` and `unsupported`.

Verified properties: - `evidenceSummary.family` matches the routed
verifier family; unsupported routing uses `null`. -
`evidenceSummary.identifier` reflects the request identifier. -
`evidenceSummary.state` matches the verifier verdict. - Valid Vellum and
CKB-PoP responses include Testnet, chain-derived owner and outPoint
transaction hash plus index. - Spore evidence includes network and
chain-derived owner. Its `outPoint.txHash` is the creation transaction
hash and `outPoint.index` is `null`; the Spore verifier reports the
creation transaction hash, not the live cell index. - For `not_found`,
`invalid` and `unsupported`, no owner or outPoint is invented. - The
summary does not change the verdict. A wrong `expectedOwner` leaves
`valid: true` and sets `ownership.matches: false`.

------------------------------------------------------------------------

## 6. Spore Live Positive-Path Validation

**Result: PASS**

A real CKB Testnet Spore credential was found and verified against the
production API. No credential, transaction, owner or proof was
fabricated.

### 6.1 Credential discovery

The installed `@ckb-ccc/spore` `findSpores` helper was unusable because
`t.findCells(...)` was not a function or its return value was not async
iterable. The search therefore used the underlying CCC client
directly: - Scanned 14,970 live Spore cells on CKB Testnet. - Decoded
each cell as a Spore molecule table. - Selected the live Spore cell
carrying both `proof` and `issuerAddress`.

### 6.2 Verified credential

  --------------------------------------------------------------------------------------------------------------------------------------------
  Field                               Value
  ----------------------------------- --------------------------------------------------------------------------------------------------------
  Spore ID                            `0x59d6ca2b82015c7fd182c5247be055fa7f644a133a105dc2b8d1b9c725e00b1a`

  Creation transaction                `0x49e23ac81b24744f9fc65aa1c545d194d1c0f861d15e9cd6e3968d250ca55ad1`

  Chain owner                         `ckt1qrfrwcdnvssswdwpn3s9v8fp87emat306ctjwsm3nmlkjg8qyza2cqgqq80ke04nnu6txrs2cln39hkn2y23kr64u5lr62qg`

  Title                               `test 1`

  Credential type                     `COURSE_COMPLETION`

  Issuer                              `Nervos Academy`

  Issue date                          `2026-10-16`

  Content type                        `application/json;version=1`

  Proof algorithm / sign type         `secp256k1` / `JoyId`
  --------------------------------------------------------------------------------------------------------------------------------------------

### 6.3 Production result

Production `POST /api/verify` returned: - HTTP 200. -
`state: "valid"`. - `valid: true`. - Chain status `active`. - Chain
owner equal to the value above. - Cryptographic status `verified`. -
`structure: true`, `signature: true`, `issuer: true`. -
`combined.valid: true`.

Supplying `expectedOwner` equal to the chain owner produced
`ownership.matches: true`.

------------------------------------------------------------------------

## 7. Wallet Identity and Ownership Validation

The core invariant is:

**Credential validity is independent from ownership comparison.**

### 7.1 Production API evidence

  ----------------------------------------------------------------------------------
  Family            Credential        Correct owner               Wrong owner
  ----------------- ----------------- --------------------------- ------------------
  Vellum            `valid: true`     `ownership.matches: true`   Still
                                                                  `valid: true`;
                                                                  `matches: false`

  CKB-PoP           `valid: true`     `ownership.matches: true`   Still
                                                                  `valid: true`;
                                                                  `matches: false`

  Spore             `valid: true`     `ownership.matches: true`   Wrong-owner API
                                                                  case was not
                                                                  separately rerun
                                                                  in the original
                                                                  matrix
  ----------------------------------------------------------------------------------

### 7.2 Automated frontend coverage

`verify-connected-wallet.test.tsx` contains 17 tests covering: -
Offering the connected address before it is used. - Filling
`expectedOwner` from the connected wallet. - Keeping credential verdict
separate from owner comparison. - Following a changed wallet address. -
Not overwriting a manually entered owner when the wallet changes. -
Clearing wallet-derived owner on disconnect.

The full frontend suite passed with 162 tests.

### 7.3 Owner-reported interactive production wallet validation

The project owner confirmed successful manual production tests for:

  -----------------------------------------------------------------------
  Test                                Result
  ----------------------------------- -----------------------------------
  Connect a real supported wallet     PASS

  MATCH --- connected wallet is the   PASS
  credential owner                    

  MISMATCH --- credential remains     PASS
  valid while ownership does not      
  match                               

  Switch wallet account/address       PASS

  Disconnect wallet and clear         PASS
  wallet-derived state                

  Verification without signing a      PASS
  transaction                         
  -----------------------------------------------------------------------

These are owner-reported manual production results. No wallet name,
browser version, transaction hash or more precise test timestamp is
asserted where it was not supplied.

------------------------------------------------------------------------

## 8. DID Profile Validation

The DID Profile path is read-only and informational: - Resolves the
address on the configured network. - Uses `ccc.didCkb.listDidCkbsByLock`
with the address's own lock script and a limit of 10. - Does not sign,
send a transaction, or expose private keys or secrets. - Returns an
empty list on malformed/unresolvable addresses or chain errors. -
Profile data does not participate in the credential verdict or ownership
comparison. - Wallet-derived state is cleared on disconnect.

Automated test results from the frontend suite:

  Test file                     Tests passed
  --------------------------- --------------
  `did-profile.test.ts`                    7
  `profile.test.tsx`                       4
  `wallet-restore.test.tsx`               10
  `user-menu.test.tsx`                    14

The project owner reported that the final DID Profile production checks
passed, including the checks performed as part of the final mobile/DID
validation. Specific browser, wallet, DID record and device details were
not supplied and are not asserted.

------------------------------------------------------------------------

## 9. Dynamic Family Discovery

**Result: PASS --- deployed and manually verified on production.**

The frontend calls `api.listFamilies()` on mount and consumes
`GET /api/verify/families`.

Behaviour: - Falls back to built-in `spore`, `vellum-claim` and
`ckb-pop` when the API is absent, rejects or returns an empty list. -
Keeps Spore as the default when no family is requested. - Preserves
tailored labels, placeholders, hints and validation for the three known
families. - Renders a discovered family as a selectable option with
generic label and hint. - Includes the current family in options even if
discovery does not mention it. - Routes the submitted identifier and
family to the backend. - Restores non-Spore deep links by reading `id`
or `sporeId`.

### 9.1 Automated tests

`verify-family-discovery.test.tsx`: 4 passed: - Discovered family is
selectable and routed. - Discovery rejection falls back to built-in
families. - Empty discovery falls back. - Non-Spore deep link is
restored and routed.

The existing `VerifyPage.test.tsx` (15 tests) continued to pass.

### 9.2 Owner-reported production checks

The project owner confirmed: - Production `GET /api/verify/families`
returned all three expected families. - The deployed frontend displayed
and allowed selection of all three. - Credential verification flows
worked correctly after deployment.

------------------------------------------------------------------------

## 10. Mobile Viewport and Final Wallet UX Validation

### 10.1 Mobile viewport checks

The project owner reported successful final mobile viewport checks for
the Verify experience, including: - Verify page rendering and
navigation. - Family selector and credential identifier input. -
Expected-owner input and connected-wallet action. - Verify button and
result presentation. - Evidence and issuer information. - Current-owner
MATCH/MISMATCH presentation. - Page scrolling and deep-link behaviour.

The original mobile check exposed a wallet dropdown clipping issue. This
was addressed in a subsequent UI refinement and production deployment.

Exact device models, browser versions and measured viewport sizes were
not supplied; this report does not claim a specific device/browser
matrix.

### 10.2 Wallet dropdown clipping fix

A production mobile screenshot showed that the wallet dropdown extended
beyond the visible viewport and some menu content was clipped.

The project owner confirmed that the fix was deployed and the mobile
menu test passed. The wallet dropdown was fully usable on the tested
mobile production interface, and the menu content was no longer clipped
as in the original screenshot.

### 10.3 Disconnect available before application login

The project owner identified that Disconnect Wallet was unavailable when
a wallet was connected but the user had not logged into the application.

The UX was corrected so wallet connection state and application
authentication are independent.

The project owner confirmed the production test passed: - A connected
but unauthenticated user can access Disconnect Wallet. - Disconnect is
not gated by application login. - Logout and Disconnect remain distinct
actions.

### 10.4 Remove duplicate wallet identity display

The header previously showed both a separate JoyID/address badge and a
clickable wallet-address control that opened wallet details and
Disconnect.

The redundant separate JoyID/address badge was removed. The project
owner confirmed: - The network status remains visible. - The clickable
wallet control remains the entry point for wallet information and
Disconnect. - The wallet identity is no longer redundantly displayed in
a separate badge.

### 10.5 Final owner confirmation

The project owner explicitly confirmed that all three final production
UI changes passed: 1. Mobile wallet dropdown clipping fix. 2. Disconnect
available while connected but not logged in. 3. Removal of the duplicate
JoyID/address badge.

These are owner-reported manual production checks.

------------------------------------------------------------------------

## 11. Full Quality Gates

All commands below were reported as actually run and passed.

### 11.1 Backend

  Gate                  Result
  --------------------- ------------------------------
  `npm run typecheck`   PASS
  `npm run lint`        PASS
  `npm test`            PASS --- 16 files, 229 tests
  `npm run build`       PASS

### 11.2 Frontend

  Gate                  Result
  --------------------- ------------------------------
  `npm run typecheck`   PASS
  `npm run lint`        PASS
  `npm test -- --run`   PASS --- 20 files, 162 tests
  `npm run build`       PASS

### 11.3 Test totals

  Area            Tests
  ----------- ---------
  Backend           229
  Frontend          162
  **Total**     **391**

The backend was not changed during the original Week 10 implementation.
The frontend suite increased from the Week 9 baseline of 158 to 162 due
to four new family discovery/deep-link tests.

The later mobile/header UX refinements were reported as deployed and
manually tested by the project owner. This report does not claim a new
automated test count for those later refinements unless separately
recorded by the implementation run.

------------------------------------------------------------------------

## 12. Deployed Browser Smoke

The original headless Chrome smoke test verified: - `/verify` rendered
the heading, Family selector, three known options, Spore hint and Verify
button. - The Spore deep link rendered `CREDENTIAL VERIFIED`, active
badge, title `test 1`, family `Spore`, issuer `Nervos Academy`, holder,
creation transaction, CKB Testnet and no error state.

This was a render smoke test, not an interactive browser suite.

The later real-wallet and final mobile/header checks were performed
manually by the project owner and are recorded separately in Sections 7
and 10.

------------------------------------------------------------------------

## 13. Bugs Found and Resolved

### 13.1 Non-Spore deep links did not restore their identifier

**Problem:** Vellum and CKB-PoP verification links used
`?id=<identifier>&family=<family>`, but the page initially read only the
legacy `sporeId` query key.

**Fix:** Initial state now reads `sporeId` or `id` before falling back
to an empty string.

**Regression test:** Added to `verify-family-discovery.test.tsx`.

**Status:** Fixed, tested, committed and deployed.

### 13.2 Mobile wallet dropdown clipped by viewport

**Problem:** The wallet dropdown was partially outside the mobile
viewport and menu content was clipped.

**Status:** The project owner confirmed the UI fix was deployed and
passed production mobile testing.

### 13.3 Disconnect unavailable when not logged in

**Problem:** Disconnect Wallet was not available in the
connected-but-unauthenticated state.

**Status:** The project owner confirmed the production UX now allows
Disconnect independently of application login.

### 13.4 Duplicate wallet identity in header

**Problem:** A separate JoyID/address badge duplicated information
already available through the clickable wallet control.

**Status:** The project owner confirmed the redundant badge was removed
and the production test passed.

------------------------------------------------------------------------

## 14. Remaining Limitations and Evidence Boundaries

The following are not recorded as failures, but remain evidence
boundaries: - Exact device models, browser versions and measured mobile
viewport sizes were not supplied. - The specific wallet name and browser
environment used for owner-reported interactive production tests were
not supplied. - No transaction was created as part of verification;
verification is intended to be read-only. - Runtime HTTP interfaces do
not expose the frontend commit SHA; deployment metadata was used to
verify deployment identity. - Spore `evidenceSummary.outPoint.index`
remains `null`; the reported hash is the creation transaction hash, not
the live cell index. A technical/product decision is still needed if the
live cell index is required. - The installed `findSpores` helper was
unusable during discovery; the underlying CCC cell query was used
instead. - Production rate-limit exhaustion was not intentionally
performed; 429 and recovery behaviour is covered by offline tests. -
Some stronger hostile inputs were blocked by the Render edge WAF before
reaching the application. - The raw screenshots and detailed
device/browser metadata for the latest owner-reported UI refinements
were not included in the report material. Their PASS status is based on
the project owner's explicit confirmation.

------------------------------------------------------------------------

## 15. Final Verdict

# PASS

Week 10 is considered complete based on the recorded production and test
evidence, plus the project owner's subsequent manual production
validations.

### Justification

-   Production backend and frontend were reachable.
-   The production API regression matrix passed all 17 recorded cases.
-   `evidenceSummary` was validated in production and did not alter
    verifier verdicts.
-   A real CKB Testnet Spore credential was verified by the production
    API with `valid: true` and a verified JoyID/secp256k1 issuer proof.
-   Credential validity and ownership comparison remained separate.
-   Production security checks produced no observed information leaks or
    false valid results.
-   Dynamic Family Discovery was committed, deployed and manually
    verified on production.
-   The deployed frontend commit identity was verified through
    deployment metadata.
-   Backend and frontend typecheck, lint, tests and builds passed (229
    backend tests and 162 frontend tests).
-   The project owner confirmed successful real-wallet production tests
    for connection, MATCH, MISMATCH, account switching and disconnect.
-   The project owner confirmed successful final mobile/header
    production checks for wallet dropdown clipping, unauthenticated
    disconnect and removal of duplicate wallet identity display.
-   DID Profile behaviour was reported as passing in final production
    validation.

Non-blocking evidence limitations remain documented in Section 14. They
do not invalidate the verified Week 10 outcomes.

------------------------------------------------------------------------

## 16. Suggested Carry-Forward to Week 11

1.  Decide whether Spore evidence summary should expose a live cell
    index in addition to the creation transaction hash.
2.  Consider adding an automated interactive browser smoke suite for
    wallet and mobile flows if ongoing regression coverage is required.
3.  Consider exposing build/deployment identity in runtime health
    metadata if operational auditability requires it.

No Week 11 implementation work is included in this Week 10 report.
