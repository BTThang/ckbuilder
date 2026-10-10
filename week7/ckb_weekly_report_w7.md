# CKB Weekly Report --- Week 7

## 1. Overview

During Week 7, the project progressed from a Digital Credential DApp
toward **CKB Credential Verification Infrastructure**. The main focus
was to consolidate the credential verification flow, validate the full
credential lifecycle on CKB Testnet, and evaluate how the project could
be differentiated as a capstone.

The system follows one key principle: **the CKB blockchain is the source
of truth**. The backend provides indexing and verification services,
while credential owners sign transactions through their wallet. The
backend does not hold users' private keys or sign transactions on their
behalf.

The application is now deployed and available online:

**Live App:** https://ckb-digital-credential-web.vercel.app

## 2. System Architecture and Design

### 2.1 Architecture

The application consists of three main parts:

-   **Frontend:** React, Vite, TypeScript, CCC, and wallet integration.
    It provides the interface for issuing, viewing, transferring, and
    melting credentials, as well as public verification.
-   **Backend:** Express, TypeScript, and CCC. It exposes APIs for
    credential operations, transaction tracking, authentication, and
    public verification.
-   **CKB Testnet and SQLite:** CKB is the authoritative source for
    on-chain credential state. SQLite stores indexed application data
    and transaction records to support the user experience and history.

The frontend is deployed on Vercel, and the backend is deployed on
Render.

### 2.2 Source-of-Truth Model

Verification checks the credential against the blockchain rather than
trusting the database alone. SQLite is an index/cache, not the final
authority. If indexed data conflicts with live on-chain state, the chain
takes precedence.

This is important for transfers and melts. A transfer should be
reflected in the current on-chain owner, while a melted credential
should no longer be reported as an existing, active credential.

### 2.3 Public Verification and Roles

Public verification is available without requiring the verifier to sign
in or connect a wallet. The product model distinguishes:

-   **Issuer:** creates and issues a credential.
-   **Holder:** owns the credential and may transfer it or initiate a
    supported lifecycle operation.
-   **Verifier:** checks the credential's status and on-chain
    information through a public verification page or API.

The public verification route is separate from authenticated application
routes. User-specific actions remain protected.

## 3. Key Features Completed

### 3.1 Credential Lifecycle

The application supports the main credential lifecycle on CKB Testnet:

1.  **Issue:** create a Spore credential.
2.  **Verify:** check that the credential exists and inspect its
    on-chain state.
3.  **Transfer:** transfer the credential to a new owner.
4.  **Verify New Owner:** confirm that public verification reflects the
    new on-chain owner.
5.  **Melt:** melt the credential.
6.  **Verify Not Found:** confirm that a melted credential is no longer
    returned as an existing credential.

### 3.2 Public Verification API

A public verification endpoint is available at:

`GET /api/verify/:credentialId`

The endpoint validates the credential identifier and delegates to the
shared verification service. It uses the chain-authoritative
verification path and distinguishes credential states from
infrastructure or verification errors.

### 3.3 QR Code Verification

The application generates a public verification URL for a credential and
presents it as a QR code. Scanning the QR code opens the public
verification page, allowing a verifier to check a credential without
signing in.

### 3.4 Transaction Tracking and Lifecycle Reconciliation

The application tracks transaction progress and presents credential
lifecycle information. Verification compares indexed information with
live chain state, helping establish whether the cached/indexed record
agrees with the current on-chain state.

### 3.5 Authentication and Wallet Identity

The application includes JoyID authentication, wallet-to-identity
binding, nonce-based challenges, server-side sessions, profile
persistence, logout, and wallet switching. Public verification does not
depend on an authenticated session.

## 4. Research on Existing CKB Credential Projects and Capstone Positioning

### 4.1 Research Motivation

The credential and identity space already has multiple projects in the
Nervos CKB ecosystem. Reviewing related projects helps identify adjacent
problem areas and avoid treating basic credential issuance alone as the
project's main differentiator.

The following projects were identified during the initial ecosystem
review. Their names indicate related areas of work, but this report does
not claim a complete feature-by-feature comparison. Specific
capabilities and current status should be confirmed against each
project's latest repository documentation before making detailed
comparisons.

  -----------------------------------------------------------------------
  Project                 Related area identified Relevance to this
                          in the initial review   project
  ----------------------- ----------------------- -----------------------
  Credora                 CKB credential registry A reference point for
                                                  exploring credential
                                                  registration and
                                                  discovery.

  SkillPass               Skills and credentials  A related credential
                                                  use case.

  CKB-PoP                 Proof of personhood     Connects to identity
                                                  and proof-related use
                                                  cases beyond
                                                  conventional
                                                  credentials.

  Vellum (`did:ckb`)      Decentralized identity  A related
                                                  identity-oriented
                                                  direction to consider
                                                  when defining project
                                                  boundaries.
  -----------------------------------------------------------------------

These projects are ecosystem references, not evidence that the systems
have identical goals or implementations.

### 4.2 Proposed Positioning: CKB Credential Verification Infrastructure

The project is being positioned as **CKB Credential Verification
Infrastructure**: a blockchain-native system for issuing, tracking, and
independently verifying digital credentials on Nervos CKB.

The intended emphasis is on verification and reliable credential state,
rather than issuance alone:

-   **Independent public verification:** third parties can verify
    credentials without logging in or connecting a wallet.
-   **Reusable verification interface:** a public API makes verification
    available beyond the main application UI.
-   **QR-based access:** a verifier can reach the public verification
    page by scanning a credential's QR code.
-   **Lifecycle-aware verification:** verification reflects changes
    caused by issue, transfer, and melt operations.
-   **Chain-authoritative status:** current on-chain state takes
    precedence over cached database records.
-   **Clear role separation:** issuer, holder, and verifier have
    different responsibilities and access requirements.

These points describe the project's intended differentiation. They are
not a claim that no other CKB project offers any of these capabilities.

### 4.3 Capstone Considerations

The next stage should validate the proposed differentiation rather than
add features without a clear purpose. Before finalizing the capstone
scope, related repositories can be compared using a consistent
checklist: target users, credential model, on-chain data model,
verification method, lifecycle support, privacy assumptions, API
availability, and deployment status.

Potential future directions should be selected after this comparison and
after the core verification workflow has been demonstrated reliably.
Advanced privacy or zero-knowledge features may be considered later, but
they are not presented as completed work in this report.

## 5. Testing and Production Validation

### 5.1 Automated Quality Checks

The latest recorded audit reported:

  Check                           Result
  ------------------------- ------------
  Web automated tests         121 passed
  Service automated tests      69 passed
  Total automated tests       190 passed
  TypeScript typecheck              PASS
  Lint                              PASS
  Production build                  PASS

These are the recorded results from the project audit. They should be
re-run if code changes after that audit.

### 5.2 Production End-to-End Test

The full production flow was manually tested on CKB Testnet. All nine
checks were confirmed as passed:

    \# Test case                                    Result
  ---- -------------------------------------------- --------
     1 Login                                        PASS
     2 Issue credential                             PASS
     3 Public verification                          PASS
     4 Transfer credential                          PASS
     5 Verify new owner                             PASS
     6 Index agrees with chain                      PASS
     7 Melt credential                              PASS
     8 Verify melted credential returns Not Found   PASS
     9 Scan QR code and open public verification    PASS

The end-to-end scenario was:

`Login → Issue → Public Verify → Transfer → Verify New Owner → Index/Chain Reconciliation → Melt → Verify Not Found → QR Scan`

This test provides practical evidence that the main lifecycle and public
verification flow work together in the deployed environment. It applies
to the tested CKB Testnet scenario and does not imply a
production-mainnet audit.

## 6. Technical Learnings

### 6.1 Blockchain State vs. Indexed State

An indexer or database can improve query performance and user
experience, but its data can become stale. Verification should consult
authoritative on-chain state when establishing whether a credential
exists and who currently owns it.

### 6.2 Credential Lifecycle Is More Than Issuance

Issuing a credential is only one part of its lifecycle. Transfers change
ownership, and melting changes whether the credential exists as a live
on-chain object. A useful verification system must reflect those
transitions accurately.

### 6.3 Public Verification Has Different Requirements from User Actions

A verifier generally needs to inspect a credential, not act as its
owner. Separating public verification from authenticated actions makes
the system easier for third parties to use while keeping
wallet-dependent operations protected.

### 6.4 QR Codes and APIs Extend the Verification Experience

A QR code is a convenient entry point, while a public API makes
verification reusable by other interfaces or services. Both should use
consistent verification logic so that the result remains aligned.

### 6.5 Production Testing Complements Automated Tests

Automated tests, typechecking, linting, and builds help catch code-level
issues. End-to-end testing checks that the deployed frontend, backend,
wallet interaction, transaction lifecycle, and public verification work
together. Both levels of validation are necessary.

## 7. Current Status and Next Steps

### 7.1 Current Status

The core credential verification workflow is implemented and has passed
the recorded automated checks and the manually confirmed production E2E
scenario on CKB Testnet. Public verification, QR access, lifecycle
handling, and chain/index reconciliation were exercised in the tested
flow.

### 7.2 Next Steps

1.  Re-run automated checks if code changes are made after the recorded
    audit.
2.  Complete a repository-level comparison of related CKB projects,
    confirming their current documentation and capabilities.
3.  Use that comparison to define a clear capstone problem statement and
    scope.
4.  Prioritize improvements that strengthen verification reliability,
    usability, or interoperability instead of adding unrelated features.
5.  Document known deployment constraints and operational limitations
    separately from tested functionality.

## Conclusion

Week 7 focused on validating the project as a credential verification
system, not merely as a credential issuance interface. The recorded
automated checks and manually confirmed production E2E flow demonstrate
the core Issue → Verify → Transfer → Verify New Owner → Melt → Not Found
sequence, including QR-based public verification.

The proposed capstone direction is **CKB Credential Verification
Infrastructure**. The next step is to validate its differentiation
through a more detailed review of related CKB projects and define a
focused capstone scope.
