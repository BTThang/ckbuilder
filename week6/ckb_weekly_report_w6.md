# CKB Digital Credential — Week 6 Report

## Wallet Authentication & User Profile

### 1. Overview

Week 6 focused on adding wallet-based authentication and user profile management to the CKB Digital Credential DApp.

The main goal was to connect a CKB wallet with a user identity, authenticate users through JoyID signatures, maintain a secure server-side session, and allow authenticated users to manage their profile information.

The application is now deployed and available online:

**Live App:** https://ckb-digital-credential-web.vercel.app

---

## 2. Objectives

The main objectives for Week 6 were:

- Implement wallet-based sign-in using JoyID.
- Bind a wallet address to a verified wallet identity.
- Implement a nonce-based authentication challenge.
- Verify the signed challenge on the backend.
- Create and manage server-side sessions.
- Store sessions using secure HttpOnly cookies.
- Persist user profile information.
- Support logout and login persistence.
- Handle wallet switching correctly.
- Protect authenticated API endpoints.
- Verify that the complete credential lifecycle still works after authentication was introduced.

---

## 3. Authentication Flow

The authentication flow was designed around a signed challenge.

```text
User connects JoyID
        ↓
Frontend requests authentication nonce
        ↓
Backend creates a short-lived challenge
        ↓
User signs the challenge with JoyID
        ↓
Frontend sends signature + wallet identity to backend
        ↓
Backend verifies the signature
        ↓
Backend verifies that the wallet identity belongs to the claimed address
        ↓
User session is created
        ↓
Session ID is stored in an HttpOnly cookie
        ↓
Authenticated requests use the session
```

The backend does not trust the wallet address supplied by the frontend alone. It verifies the relationship between the wallet identity and the CKB address before creating a session.

---

## 4. JoyID Wallet Verification

The application supports JoyID authentication on CKB Testnet.

For JoyID wallets, the backend verifies the credential through the JoyID credential server. The production Testnet configuration uses:

```text
CKB_NETWORK=testnet
JOYID_CREDENTIAL_SERVER_URL=https://api.testnet.joyid.dev/api/v1
```

This allows the backend to verify that the public key associated with the JoyID identity is registered for the claimed CKB address.

The authentication flow was tested successfully on the deployed application.

---

## 5. Session Management

After successful wallet verification, the backend creates a server-side session.

The session architecture is:

```text
JoyID Signature
      ↓
Backend verification
      ↓
User record
      ↓
Server-side session
      ↓
HttpOnly session cookie
```

The session cookie is configured for production use with:

- `HttpOnly`
- `Secure`
- `SameSite=None`
- `Path=/`

The session ID is not stored in `localStorage`.

The production frontend and backend are deployed on separate domains, so the cookie configuration was adjusted to allow authenticated cross-origin requests between the Vercel frontend and Render backend.

---

## 6. User Persistence

A user record is associated with the authenticated wallet address.

The profile supports information such as:

- Display name
- Avatar URL
- Bio
- Organization
- Organization type
- Wallet address

Profile information is stored by the backend and can be retrieved again after refreshing the application.

---

## 7. Profile Management

An authenticated user can access the Profile page and update personal profile information.

The profile flow is:

```text
Sign in
   ↓
Open Profile
   ↓
Edit profile information
   ↓
Save changes
   ↓
Backend validates authenticated session
   ↓
Profile is persisted
```

The profile API is protected and requires a valid authenticated session.

The production deployment was tested successfully after resolving the session-cookie configuration for the Vercel-to-Render architecture.

---

## 8. Logout

Logout invalidates the current server-side session.

The expected flow is:

```text
Signed in
   ↓
Logout
   ↓
Session invalidated
   ↓
User becomes unauthenticated
```

After logout, authenticated profile operations are no longer available until the user signs in again.

---

## 9. Wallet Switching

Wallet switching was tested to make sure user data does not incorrectly carry over between different wallet identities.

The tested flow was:

```text
Wallet A
   ↓
Sign in
   ↓
Profile A

Switch wallet
   ↓
Wallet B
   ↓
Sign in
   ↓
Profile B
```

The application correctly associates the authenticated session with the currently signed-in wallet.

---

## 10. Authentication and Security Checks

Several security-related checks were performed during implementation.

### Wallet identity binding

The backend verifies that the identity that produced the signature is actually bound to the claimed CKB address.

### Nonce challenge

Authentication uses a short-lived nonce challenge instead of signing a static message.

### Server-side session

The server maintains the session instead of exposing a long-lived authentication token to frontend JavaScript.

### HttpOnly cookie

The session cookie is marked `HttpOnly`, reducing the ability of client-side JavaScript to access the session identifier.

### Secure cookie

Production sessions use secure cookies over HTTPS.

### Fail-closed wallet verification

If the JoyID credential server is unavailable or cannot verify the wallet identity, the backend refuses authentication instead of granting a session.

---

## 11. Production Deployment

The application is now deployed as a real web application.

```text
                    ┌──────────────────────┐
                    │       Vercel         │
                    │      Frontend        │
                    └──────────┬───────────┘
                               │
                               │ HTTPS API
                               ▼
                    ┌──────────────────────┐
                    │       Render         │
                    │       Backend        │
                    └───────┬───────┬──────┘
                            │       │
                            │       │
                            ▼       ▼
                     CKB Testnet   SQLite
                            │
                            ▼
                          JoyID
                     Credential Server
```

Live application:

**https://ckb-digital-credential-web.vercel.app**

The production environment successfully supports wallet connection, authentication, profile management, and communication between the frontend and backend.

---

## 12. End-to-End Regression Test

After authentication was implemented, the complete credential workflow was tested again to make sure authentication did not break the existing Spore functionality.

The tested flow was:

```text
Login
  ↓
Issue Credential
  ↓
Verify Credential
  ↓
Transfer Credential
  ↓
Verify new owner
  ↓
Check indexed state against blockchain
  ↓
Melt Credential
  ↓
Verify Credential
  ↓
not_found
```

The complete flow passed successfully.

This confirms that wallet authentication and user sessions can coexist with the existing CKB credential lifecycle.

---

## 13. Test Results

| Test | Result |
|---|---|
| Connect JoyID wallet | PASS |
| Sign in with wallet | PASS |
| Wallet identity verification | PASS |
| Server-side session creation | PASS |
| HttpOnly session cookie | PASS |
| Logout | PASS |
| Login persistence | PASS |
| Profile save | PASS |
| Profile persistence after refresh | PASS |
| Wallet switching | PASS |
| Authenticated API access | PASS |
| Issue credential after authentication | PASS |
| Verify credential | PASS |
| Transfer credential | PASS |
| Verify new owner | PASS |
| Index agrees with blockchain | PASS |
| Melt credential | PASS |
| Verify melted credential | PASS / `not_found` |
| Production frontend ↔ backend connection | PASS |

---

## 14. Key Learnings

### Wallet authentication is more than verifying a signature

A valid signature alone does not prove that the signer controls the CKB address being claimed.

The backend therefore needs to verify:

```text
Signature
+
Wallet identity
+
CKB address binding
```

before creating an authenticated session.

### Nonce-based authentication prevents replay of old challenges

Each login attempt uses a short-lived server-generated challenge. This prevents users from simply reusing an old signed authentication message.

### Authentication state should be managed by the backend

Using a server-side session with an HttpOnly cookie keeps the session identifier away from frontend JavaScript and provides a clearer security boundary.

### Production deployment can change authentication behavior

The local development environment used localhost for both frontend and backend, while production uses Vercel and Render on different domains.

This required production-aware cookie configuration so that authenticated API requests could carry the session correctly.

---

## 15. Week 6 Outcome

By the end of Week 6, the CKB Digital Credential DApp gained a complete wallet authentication and user profile layer.

The application now supports:

- JoyID wallet connection
- Wallet-based sign-in
- Nonce challenge authentication
- Wallet identity verification
- Server-side sessions
- Secure session cookies
- User profile management
- Logout
- Login persistence
- Wallet switching
- Protected profile APIs
- Production deployment
- Full credential lifecycle after authentication

The application is available online for testing:

**https://ckb-digital-credential-web.vercel.app**

Week 6 establishes the authentication and identity foundation required for building a multi-user digital credential application on Nervos CKB.
