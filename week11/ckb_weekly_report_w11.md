# CKB Credential Verification Infrastructure — Week 11 Report

**Report date:** 2026-11-07   
**Scope:** `ckb-digital-credential-web` (Vercel), `ckb-digital-credential-service` (Render), CKB Testnet  
**Report type:** Implementation summary, automated regression, production acceptance, and owner-confirmed manual validation  
**Final verdict:** **PASS**

---

## 1. Executive Summary

Week 11 completed four engineering workstreams:

1. Spore evidence enhancement: expose the live cell output index together with its transaction hash.
2. Automated browser testing: add Playwright/Chromium coverage for wallet and verification UI flows using a deterministic test wallet and stubbed API.
3. Build and deployment traceability: expose build identity from frontend and backend so production artifacts can be matched to repository commits.
4. Cross-family regression and extensibility: expand shared verification contract tests across Spore, Vellum Claim, and CKB-PoP.

All four workstreams are committed and present in production. The production frontend and backend report the same commit SHAs as their respective `main` branches. All automated quality gates pass:

- Frontend: **173/173 Vitest tests** and **11/11 Playwright tests**.
- Backend: **252/252 tests**.
- Frontend and backend typecheck, lint, and production build: PASS.
- Production API acceptance: **19/19 checks PASS**.

After the original release validation report was prepared, the project owner confirmed that real-wallet and UI checks passed on **both desktop and mobile**. These are recorded as **owner-confirmed manual results**, not automated Playwright coverage. Exact device models, browser versions, timestamps, and screenshots were not supplied and are not asserted here.

The 5–7 item visual evidence pack and provider dashboard deployment IDs/timestamps are deferred to **Week 12 evidence collection and handover**. Their absence does not change the Week 11 PASS verdict.

No credential was created, transferred, signed, or modified during read-only production acceptance.

---

## 2. Project and Environment

**Project:** CKB Credential Verification Infrastructure  
**Positioning:** Verification as infrastructure, not an app.

Core principles:
- CKB chain state remains the source of truth.
- Indexers support discovery/cache, not independent authority for credential validity.
- Verification core and API remain family-agnostic.
- Protocol-specific parsing and verification remain in family-specific verifiers/readers.
- Common states: `valid`, `invalid`, `not_found`, `unable_to_verify`, `unsupported`.
- Ownership comparison is independent of credential validity.
- A connected wallet address is context for ownership comparison, not proof of credential validity.
- Public verification does not require application login.
- Backend does not hold user keys or sign on behalf of users.

| Component | Repository / platform | Production URL |
|---|---|---|
| Frontend | `ckb-digital-credential-web` / Vercel | https://ckb-digital-credential-web.vercel.app/ |
| Verify page | Frontend | https://ckb-digital-credential-web.vercel.app/verify |
| Backend | `ckb-digital-credential-service` / Render | https://ckb-digital-credential-service.onrender.com/ |
| API base | Backend | https://ckb-digital-credential-service.onrender.com/api |

**Network:** CKB Testnet  
**Supported families:** `spore`, `vellum-claim`, `ckb-pop`

---

## 3. Week 11 Deliverables

| # | Workstream | Deliverable | Result |
|---:|---|---|---|
| 1 | Spore Evidence Enhancement | Live Spore outpoint exposes `txHash` and decimal `index`; index is never fabricated | PASS |
| 2 | Automated Browser Testing | Playwright/Chromium browser suite with deterministic wallet and stubbed API | 11/11 PASS |
| 3 | Build & Deployment Traceability | Frontend `/version.json`; backend `/api/version` and health build metadata | PASS |
| 4 | Cross-family Regression | Shared matrix across Spore, Vellum Claim, CKB-PoP; ownership independence and unsupported behavior | 15 new tests; full backend 252/252 PASS |
| 5 | Production Acceptance | Live production endpoint and verification checks | 19/19 PASS |
| 6 | Manual real-wallet smoke checks | Owner-confirmed real-wallet and desktop/mobile checks | PASS, manual |

---

## 4. Spore Evidence Enhancement

The backend reads `cell.outPoint.index` from the live Spore cell resolved through the Spore integration (`@ckb-ccc/spore` `findSpore`). The evidence response exposes the outpoint transaction hash and output index as a decimal string.

The implementation does not fabricate an index. If a live cell cannot be resolved, the index remains `null`.

**Live Testnet fixture:**
- Credential ID: `0x59d6ca2b82015c7fd182c5247be055fa7f644a133a105dc2b8d1b9c725e00b1a`
- Outpoint transaction hash: `0x49e23ac81b24744f9fc65aa1c545d194d1c0f861d15e9cd6e3968d250ca55ad1`
- Output index: `"0"`

Production returned a valid result with both transaction hash and output index.

**Result: PASS**

---

## 5. Automated Browser Testing

Playwright with Chromium was added because jsdom does not provide full browser layout, CSS, and navigation behavior. The E2E entry mounts the real app while using a deterministic test-only wallet provider, stubbed verification API responses, and external-host blocking for deterministic isolation.

Coverage includes:
- Wallet connection and sign-in UI behavior.
- Credential ownership `MATCH` and `MISMATCH`.
- Account switching.
- Wallet disconnect.
- Disconnect behavior independent of application login/logout state.
- Mobile navigation and wallet dropdown at a narrow viewport.
- Verification without unnecessary transaction signing.
- Build/version metadata behavior.

| Suite | Result |
|---|---:|
| Wallet/verification Playwright tests | 9/9 PASS |
| Version metadata Playwright tests | 2/2 PASS |
| **Total** | **11/11 PASS** |

**Boundary:** Playwright uses a deterministic test wallet and stubbed APIs. It is not a live JoyID/CCC provider integration test.

**Result: PASS**

---

## 6. Build and Deployment Traceability

### Frontend
The frontend build emits `/version.json` and HTML metadata tags for app name, app version, Git commit, build time, deployment environment, and network. Vercel routing prevents `/version.json` from being masked by the SPA fallback; the endpoint uses `no-store`.

### Backend
The backend exposes `GET /api/version` and an additive `build` object in `GET /api/health`. Build metadata is generated at build time and contains public traceability fields only.

### Production metadata

| Field | Frontend / Vercel | Backend / Render |
|---|---|---|
| Version | `1.0.0` | `1.0.0` |
| Commit | `c5db2d71e134e3a96b59f7ca7583650f199126c1` | `303b0fd87b22e095d16561a85c12ed8875cbcccc` |
| Environment | `production` | `production` |
| Network | `testnet` | `testnet` |
| Build timestamp | `2026-11-04T14:29:28.483Z` | `2026-11-04T14:29:32.607Z` |

Both production services report the same commits as their respective local `HEAD` and `origin/main`.

Provider deployment IDs and dashboard deployment-event timestamps were not available during validation. Live build metadata is the evidence used to match production artifacts to pushed source commits.

**Result: PASS**

---

## 7. Cross-family Regression and Extensibility

The regression suite exercises the shared verification contract across `spore`, `vellum-claim`, and `ckb-pop`.

Coverage includes valid/invalid credentials, malformed identifiers, not-found results, data-source failures, correct/incorrect `expectedOwner`, explicit/missing family, unsupported-family envelopes, common result/evidence fields, and registration of a test-only `mock-demo` family.

Assertions confirm:
- An explicitly supplied family is authoritative.
- Vellum Claim and CKB-PoP identifiers are not implicitly inferred when `family` is omitted.
- Ownership `MATCH` or `MISMATCH` does not change the credential verification state.
- Unsupported inputs return the common unsupported response envelope.
- A test-only family can be registered without changing the three production families.

Results:
- New Week 11 cross-family tests: **15/15 PASS**.
- Full backend suite: **252/252 PASS**.
- Frontend regression guard: **173/173 PASS**.

The cross-family matrix is an offline/fake-client regression suite. Live-chain behavior is assessed separately.

**Result: PASS**

---

## 8. Git and Code Audit

| Repository | Branch | Commit | HEAD = `origin/main` | Working tree |
|---|---|---|---|---|
| `ckb-digital-credential-web` | `main` | `c5db2d71e134e3a96b59f7ca7583650f199126c1` | Yes | Clean |
| `ckb-digital-credential-service` | `main` | `303b0fd87b22e095d16561a85c12ed8875cbcccc` | Yes | Clean |

The Week 11 commits were already pushed. No uncommitted or unpushed changes were present during the release audit.

**Frontend change summary:** 18 files, +1055 / -11. Included Playwright configuration, test-only E2E entry/provider, browser tests, build-info resolver, `version.json` Vite plugin, Vercel routing hardening, and wallet provider export.

**Backend change summary:** 19 files, +998 / -19. Included Spore output-index evidence, build-info manifest generation/loading, `/api/version`, additive health build metadata, and the 15-test cross-family regression suite.

Repository hygiene:
- Commit-diff secret scan found documentation/test references such as `private key`, `token`, and `RENDER_GIT_COMMIT`, but no credentials.
- `.env` remains ignored/untracked; only `.env.example` is tracked.
- `dist/`, `test-results/`, `node_modules/`, `data/`, and `.env` are ignored and were not accidentally committed.

**Result: PASS**

---

## 9. Automated Quality Gates

### Frontend — `ckb-digital-credential-web`

| Command | Result |
|---|---|
| `npm test` (Vitest) | **173/173 passed**, 22 files |
| `npm run test:e2e` (Playwright / Chromium) | **11/11 passed** |
| `npm run typecheck` (`tsc -b`) | PASS |
| `npm run lint` (`eslint .`) | PASS |
| `npm run build` (`tsc -b && vite build`) | PASS; production bundle and `dist/version.json` emitted |

### Backend — `ckb-digital-credential-service`

| Command | Result |
|---|---|
| `npm test` (Vitest) | **252/252 passed**, 18 files |
| `npm run typecheck` (`tsc --noEmit`) | PASS |
| `npm run lint` (`eslint .`) | PASS |
| `npm run build` (`tsc -p tsconfig.build.json && generate-build-info.mjs`) | PASS; `dist/build-info.json` generated |

**Non-blocking observation:** the frontend suite prints React `act(...)` warnings for some asynchronous `VerifyPage` state updates. Tests still pass; these are non-blocking test-hygiene warnings.

**Overall automated quality gate: PASS**

---

## 10. Production Deployment and Endpoint Verification

### Frontend

| Check | Observed | Result |
|---|---|---|
| `GET /version.json` | HTTP 200 | PASS |
| Content type | `application/json; charset=utf-8` | PASS |
| Cache control | `no-store` | PASS |
| Commit metadata | Matches frontend `main` commit | PASS |
| Environment / network | `production` / `testnet` | PASS |
| `GET /` | HTTP 200, HTML | PASS |
| HTML metadata tags | Present and consistent with `version.json` | PASS |

### Backend

| Check | Observed | Result |
|---|---|---|
| `GET /api/version` | HTTP 200 | PASS |
| Commit metadata | Matches backend `main` commit | PASS |
| Environment / network | `production` / `testnet` | PASS |
| `GET /api/health` | HTTP 200, `status: ok` | PASS |
| Chain connectivity | `chain.reachable: true`, `chain.network: testnet` | PASS |
| Health build metadata | Commit matches `/api/version` | PASS |
| `GET /api/verify/families` | HTTP 200, three families | PASS |

Observed CKB Testnet tip at the time of original validation: `22974136`.

**Deployment traceability limitation:** provider deployment IDs and exact dashboard event timestamps were not retrievable because Vercel/Render dashboard credentials or CLI access were unavailable. This is a Week 12 evidence item, not a deployment failure. Production endpoints report build metadata matching the pushed commits.

---

## 11. Live Production Acceptance Matrix

All checks below were read-only `POST /api/verify` requests against the live Render service on CKB Testnet. No credential lifecycle action was performed.

| # | Scenario | Expected / observed behavior | Result |
|---:|---|---|---|
| 1 | Real Spore credential | `state: valid`; live outpoint includes transaction hash and `index: "0"` | PASS |
| 2 | Missing Spore identifier | `state: not_found`, `valid: false`, no outpoint | PASS |
| 3 | Malformed Spore ID — non-hex | `state: invalid` | PASS |
| 4 | Malformed Spore ID — too short | `state: invalid` | PASS |
| 5 | Known valid Vellum Claim | `state: valid`; outpoint returned | PASS |
| 6 | Known valid CKB-PoP badge | `state: valid`; outpoint returned | PASS |
| 7 | Spore ownership MATCH | `matches: true`; credential remains `valid` | PASS |
| 8 | Spore ownership MISMATCH | `matches: false`; credential remains `valid` | PASS |
| 9 | Vellum ownership MATCH | `matches: true`; credential remains `valid` | PASS |
| 10 | Vellum ownership MISMATCH | `matches: false`; credential remains `valid` | PASS |
| 11 | CKB-PoP ownership MATCH | `matches: true`; credential remains `valid` | PASS |
| 12 | CKB-PoP ownership MISMATCH | `matches: false`; credential remains `valid` | PASS |
| 13 | Unsupported family | Common unsupported response; no family routing | PASS |
| 14 | Bare Vellum ID without family | `unsupported`; no implicit family inference | PASS |
| 15 | Bare CKB-PoP ID without family | `unsupported`; no implicit family inference | PASS |
| 16 | Unknown API route | HTTP 404 JSON error envelope; no stack trace | PASS |
| 17 | Malformed JSON body | HTTP 400 JSON error envelope; no stack trace | PASS |
| 18 | Build metadata exposure | Only allow-listed public build fields returned | PASS |
| 19 | Sensitive-data exposure check | No secrets or credential material found in checked responses | PASS |

### Live fixtures
- Spore credential: `0x59d6ca2b82015c7fd182c5247be055fa7f644a133a105dc2b8d1b9c725e00b1a`
- Spore outpoint transaction hash: `0x49e23ac81b24744f9fc65aa1c545d194d1c0f861d15e9cd6e3968d250ca55ad1`
- Spore output index: `0`
- Vellum Claim: `did:ckb:fn7u37m7vwerr4ojysgdwwp4mescjtrp`
- CKB-PoP badge: `0xb5b0a980df2698c1bd3298d3625960566c2c6aa5659cd06bc027c893783eadf6c72f02083a18ba7872f60b7ae510c86b1cf0e0d3c184e96644f73735`

**Production acceptance: 19/19 PASS**

---

## 12. Manual Real-wallet and Desktop/Mobile Validation

### Owner confirmation

After the original release validation report, the project owner explicitly confirmed that manual testing passed on **desktop and mobile**.

This is recorded as a manual, owner-reported validation result, separate from Playwright, which uses a deterministic test wallet and stubbed API.

| Area | Result | Evidence classification |
|---|---|---|
| Real wallet connection on desktop | PASS — owner confirmed | Manual owner confirmation |
| Wallet and credential verification interaction | PASS — owner confirmed | Manual owner confirmation |
| Ownership MATCH / MISMATCH interaction | PASS — owner confirmed | Manual owner confirmation |
| Wallet switching and disconnect interaction | PASS — owner confirmed | Manual owner confirmation |
| Desktop verification experience | PASS — owner confirmed | Manual owner confirmation |
| Mobile verification experience | PASS — owner confirmed | Manual owner confirmation |
| Mobile wallet menu / disconnect experience | PASS — owner confirmed | Manual owner confirmation |

The owner confirmed that desktop and mobile testing passed. Exact device models, operating-system versions, browser versions, viewport dimensions, test timestamps, and individual screenshots were not provided and are intentionally omitted.

**Boundary:** Playwright is automated with a deterministic test wallet and stubbed API. Real-wallet desktop/mobile checks were performed manually and confirmed by the owner; they are not represented as automated JoyID/CCC integration tests.

**Manual desktop/mobile status: PASS (owner-confirmed)**

---

## 13. Security and Information Exposure

- Error responses use a stable JSON envelope with error code, message, and request ID.
- No stack traces, file paths, or framework internals were observed in checked error responses.
- `/api/version` returns allow-listed public build fields and does not expose environment variables, secrets, or connection strings.
- `/api/health` exposes the public CKB Testnet RPC endpoint `wss://testnet.ckb.dev/ws`. This is public infrastructure information, not a credential.
- Platform headers reveal hosting information already evident from the public service URL.
- Both production services use HTTPS.
- No API keys, private keys, session tokens, passwords, or cookies were found in checked responses or commit diffs.

**Result: PASS**

---

## 14. Limitations and Known Issues

These are documented boundaries, not Week 11 release blockers:

1. **Provider deployment IDs/timestamps:** Vercel and Render deployment IDs and dashboard event timestamps were not captured. Production build metadata matches the pushed commits. Collect provider-side details in Week 12 if dashboard access is available.
2. **Screenshot evidence pack:** manual desktop/mobile checks have been confirmed PASS, but supporting screenshots have not yet been collected. Collect a concise evidence pack in Week 12.
3. **Manual test metadata:** exact device models, browser versions, viewport sizes, and test timestamps were not provided. They are omitted rather than guessed.
4. **Automated wallet boundary:** Playwright uses a deterministic wallet and stubbed API; it does not replace live JoyID/CCC provider testing.
5. **Explicit family requirement:** Vellum Claim and CKB-PoP require an explicit `family`; bare identifiers return `unsupported`. This is intentional.
6. **Issuer field:** `issuer` is source-backed for Spore only. Vellum Claim and CKB-PoP return `null` rather than invented issuer data.
7. **Live acceptance is point-in-time:** Testnet fixtures represent live cells at validation time. A later transfer or melt may change current outpoint/status.
8. **React `act(...)` warnings:** non-failing frontend test warnings remain and do not affect passing test/build gates.

None of these items invalidates the Week 11 release result.

---

## 15. Evidence Register

| Evidence | Status | Notes |
|---|---|---|
| Frontend automated test output | Available | 173/173 |
| Playwright browser test output | Available | 11/11 |
| Backend automated test output | Available | 252/252 |
| Typecheck, lint, and build results | Available | Both repositories pass |
| Git commit and clean working-tree audit | Available | Both repositories on `main`; clean |
| Production build/version metadata | Available | Commit SHAs match repository `main` |
| Live production API acceptance | Available | 19/19 checks pass |
| Manual desktop/mobile test result | Owner-confirmed | PASS; screenshots not yet collected |
| Vercel/Render deployment IDs and dashboard timestamps | Pending | Not retrievable during Week 11 validation |
| Manual screenshot evidence pack | Deferred to Week 12 | Target approximately 5–7 consolidated screenshots |

### Planned Week 12 screenshot pack

These screenshots are supplementary evidence for checks already completed in Week 11. They are not a requirement to rerun or reopen Week 11 validation.

| # | Planned screenshot | What it demonstrates |
|---:|---|---|
| 1 | JoyID connected on desktop | Real wallet connection state |
| 2 | Credential verification — MATCH | Ownership match UI |
| 3 | Credential verification — MISMATCH | Ownership mismatch UI |
| 4 | Wallet switching and Disconnect on desktop | Account switching and disconnect controls |
| 5 | Verify page on mobile | Mobile verification layout |
| 6 | Wallet menu and Disconnect on mobile | Mobile wallet controls |
| 7 | Vercel/Render deployment details, if accessible | Provider-side deployment record |

Before sharing screenshots, redact full wallet addresses or unrelated personal information that does not need to appear in the evidence.

---

## 16. Week 12 Carry-forward

Week 12 should focus on **final evidence collection, documentation, and handover**, not reopening completed Week 11 feature work.

Recommended tasks:
1. Capture and organize the 5–7 consolidated screenshots listed above.
2. If dashboard access is available, record Vercel and Render deployment IDs and deployment-event timestamps.
3. Add final evidence references to the project release/closure documentation.
4. Review and finalize README, architecture overview, API reference, operational notes, and handover instructions.
5. Prepare the final project closure report and sign-off.

Week 12 should label screenshots and deployment details as supplementary evidence for Week 11 results. The Week 11 PASS verdict remains unchanged.

---

## 17. Final Verdict

# PASS

Week 11 is complete and release-ready.

Basis for PASS:
- All four Week 11 engineering workstreams are committed and included in production.
- Frontend and backend production report the exact commits present on their respective `main` branches.
- Frontend automated tests: **173/173 PASS**.
- Playwright browser tests: **11/11 PASS**.
- Backend automated tests: **252/252 PASS**.
- Frontend and backend typecheck, lint, and production build: **PASS**.
- Production endpoint and build metadata checks: **PASS**.
- Live production acceptance: **19/19 PASS** across Spore, Vellum Claim, CKB-PoP, ownership comparison, unsupported inputs, error envelopes, and information-exposure checks.
- Project owner confirmed manual real-wallet and desktop/mobile checks passed.

The remaining screenshot pack and provider dashboard deployment details are evidence/handover follow-ups for Week 12. They are not failing release gates and do not require changing the Week 11 verdict.

**Final status: WEEK 11 — PASS / CLOSED FOR RELEASE VALIDATION**

---

## Appendix A — Key Production Endpoints

| Purpose | Endpoint |
|---|---|
| Frontend | https://ckb-digital-credential-web.vercel.app/ |
| Verify UI | https://ckb-digital-credential-web.vercel.app/verify |
| Frontend build metadata | https://ckb-digital-credential-web.vercel.app/version.json |
| Backend API base | https://ckb-digital-credential-service.onrender.com/api |
| Backend version | https://ckb-digital-credential-service.onrender.com/api/version |
| Backend health | https://ckb-digital-credential-service.onrender.com/api/health |
| Supported verification families | https://ckb-digital-credential-service.onrender.com/api/verify/families |

## Appendix B — Repository Commits

- Frontend: `c5db2d71e134e3a96b59f7ca7583650f199126c1`
- Backend: `303b0fd87b22e095d16561a85c12ed8875cbcccc`
