# Requirements: Abyss — Local Password & Key Vault

**Defined:** 2026-04-29
**Source:** Derived from `docs/PRD.md` v1.2 (Architecture-Locked MVP Build Contract). The PRD is authoritative; this file mirrors it in GSD's REQ-ID format and groups items for phase mapping.
**Core Value:** A user can create an encrypted local vault with a master password, store and retrieve secrets, and never see plaintext sensitive data persisted to disk. Vault confidentiality and tamper-evidence (DEK/KEK envelope encryption with AES-256-GCM AAD), end-to-end, on macOS and Windows.

## v1 Requirements

Requirements for v1.0 MVP (PRD §15 milestones v0.1 → v1.0). Each maps to a roadmap phase.

### Vault Lifecycle (VAULT)

PRD §8.1, §2.6, §5.

- [ ] **VAULT-01**: Create a new SQLite vault file with project-specific `application_id` set (PRD §6.1, FR-001, FR-003)
- [ ] **VAULT-02**: Initialize schema via `golang-migrate` migrations embedded into the Go binary (PRD §2.5, FR-002)
- [ ] **VAULT-03**: Generate a fresh `vault_id` UUID at vault creation (FR-004)
- [ ] **VAULT-04**: Generate an active DEK using `crypto/rand` 32 bytes at vault creation (FR-005)
- [ ] **VAULT-05**: Wrap the DEK using a master-password-derived KEK (Argon2id) and store in `key_wrappings` (FR-006, PRD §5.2)
- [ ] **VAULT-06**: Unlock vault with master password — derive KEK, unwrap DEK, generic failure on any error (FR-007, PRD §9.4)
- [ ] **VAULT-07**: Manually lock vault (FR-008)
- [ ] **VAULT-08**: Auto-lock vault after configurable idle timeout (default 10 min; allowed: 1 / 5 / 10 / 15 / 30 min, Disabled-with-warning) (FR-009, PRD §13)
- [ ] **VAULT-09**: Change master password — derive new KEK with new salt, re-wrap the same DEK; do not re-encrypt records (FR-010, PRD §5.2)
- [ ] **VAULT-10**: Clear decrypted frontend and backend state on lock (FR-011)
- [ ] **VAULT-11**: Attempt clipboard clear on lock (FR-012)
- [ ] **VAULT-12**: Reject all secret actions while locked (FR-013)
- [ ] **VAULT-13**: Lock-on-system-sleep / lid-close (PRD §13.4 Should-have for MVP hardening; required for v1.0)

### Record Management (REC)

PRD §6.3, §7, §8.2.

- [ ] **REC-01**: Create credential records (encrypted metadata + encrypted secret blob per PRD §7.1) (FR-014)
- [ ] **REC-02**: Create API key records (PRD §7.2) (FR-015)
- [ ] **REC-03**: Create secure note records (PRD §7.3) (FR-016)
- [ ] **REC-04**: Create symmetric (AES) key records (PRD §7.4) (FR-017)
- [ ] **REC-05**: Create asymmetric key pair records (RSA / Ed25519) with plaintext public-key columns and encrypted private-key blob (PRD §7.5) (FR-018)
- [ ] **REC-06**: View records after unlock (FR-019)
- [ ] **REC-07**: Edit records (re-encrypt metadata and/or secret on change) (FR-020)
- [ ] **REC-08**: Delete records after explicit confirmation (FR-021)
- [ ] **REC-09**: Search records in memory after unlock (FR-022)
- [ ] **REC-10**: Mask secret values by default (FR-023)
- [ ] **REC-11**: Temporarily reveal secrets on user action (FR-024)
- [ ] **REC-12**: Copy selected fields to clipboard (FR-025)
- [ ] **REC-13**: Store tags inside the encrypted metadata blob (NOT plaintext columns) (FR-026, Should-have promoted to v1)

### Password Generator (PASS)

PRD §8.3.

- [ ] **PASS-01**: Generate passwords using `crypto/rand` (FR-027)
- [ ] **PASS-02**: Configurable length (default 24, min 8, max 128) (FR-028, PRD §11.2)
- [ ] **PASS-03**: Toggleable character classes: uppercase, lowercase, numbers, symbols (FR-029)
- [ ] **PASS-04**: Toggleable exclude-ambiguous-characters option (FR-030, Should promoted to v1)
- [ ] **PASS-05**: Copy generated password (FR-031)
- [ ] **PASS-06**: Save generated password as a credential record (FR-032)
- [ ] **PASS-07**: Show password strength using a vetted zxcvbn-style library only (FR-033, Should promoted to v1)

### AES Key Generator (AES)

PRD §8.4, §11.3.

- [ ] **AES-01**: Generate AES-128 keys using `crypto/rand` (FR-034)
- [ ] **AES-02**: Generate AES-192 keys (FR-035)
- [ ] **AES-03**: Generate AES-256 keys (FR-036)
- [ ] **AES-04**: Display generated key as Base64 (FR-037)
- [ ] **AES-05**: Display generated key as hex (FR-038)
- [ ] **AES-06**: Save generated AES key to vault as a symmetric key record (FR-039)
- [ ] **AES-07**: Copy generated AES key while unlocked (FR-040)

### RSA Key Generator (RSA)

PRD §8.5, §11.4. Default size 3072.

- [ ] **RSA-01**: Generate 2048-bit RSA key pairs (FR-041)
- [ ] **RSA-02**: Generate 3072-bit RSA key pairs (FR-042)
- [ ] **RSA-03**: Generate 4096-bit RSA key pairs (FR-043)
- [ ] **RSA-04**: Export public key in OpenSSH format (FR-044)
- [ ] **RSA-05**: Export public key in PEM format (FR-045, Should promoted to v1)
- [ ] **RSA-06**: Export private key in PKCS#8 PEM format using `x509.MarshalPKCS8PrivateKey` (FR-046)
- [ ] **RSA-07**: Warn before private key copy/export via the exposure-warning modal (FR-047)
- [ ] **RSA-08**: Save generated RSA key pair to vault as an asymmetric key pair record (FR-048)

### Ed25519 Key Generator (ED)

PRD §8.6.

- [ ] **ED-01**: Generate Ed25519 key pairs (FR-049)
- [ ] **ED-02**: Label Ed25519 as a signing-key algorithm in the UI (FR-050)
- [ ] **ED-03**: Export public key in OpenSSH format (FR-051)
- [ ] **ED-04**: Export public key in PEM format (FR-052, Should promoted to v1)
- [ ] **ED-05**: Export private key in PKCS#8 PEM format (FR-053)
- [ ] **ED-06**: Export private key in OpenSSH format (FR-054, Should promoted to v1)
- [ ] **ED-07**: Warn before private key copy/export via the exposure-warning modal (FR-055)
- [ ] **ED-08**: Save generated Ed25519 key pair to vault as an asymmetric key pair record (FR-056)

### Clipboard (CLIP)

PRD §2.7, §8.7.

- [ ] **CLIP-01**: Copy selected secrets to OS clipboard via Go backend (FR-057)
- [ ] **CLIP-02**: Display countdown progress bar in frontend (FR-058)
- [ ] **CLIP-03**: Attempt clipboard clear after configured timeout (allowed 10 / 30 / 60 / 120 s; default 30 s) (FR-059, PRD §2.7)
- [ ] **CLIP-04**: Allow manual clipboard clear ("Clear now" button) (FR-060, Should promoted to v1)
- [ ] **CLIP-05**: Reset countdown when a new secret is copied (FR-061)
- [ ] **CLIP-06**: Attempt clipboard clear when vault locks (FR-062)
- [ ] **CLIP-07**: Document clipboard manager limitations in the README (FR-063)

### Backend Bindings & DTOs (API)

PRD §10. Locked Wails-binding contract — Go backend exposes these to the React frontend.

- [ ] **API-01**: All security-sensitive operations live in Go; frontend has no direct SQLite / crypto / filesystem / clipboard access (PRD §10)
- [ ] **API-02**: Implement vault DTOs and methods: `CreateVault`, `OpenVault`, `UnlockVault`, `LockVault`, `ChangeMasterPassword`, `GetVaultStatus` (PRD §10.2)
- [ ] **API-03**: Implement record DTOs and methods: `ListRecords`, `GetRecord`, `CreateRecord`, `UpdateRecord`, `DeleteRecord`, `SearchRecords` (PRD §10.3)
- [ ] **API-04**: Implement password generator DTOs and `GeneratePassword` method (PRD §10.4)
- [ ] **API-05**: Implement key generator DTOs and methods: `GenerateAESKey`, `GenerateRSAKeyPair`, `GenerateEd25519KeyPair` (PRD §10.5)
- [ ] **API-06**: Implement clipboard DTOs and methods: `CopyToClipboard`, `ClearClipboard` (PRD §10.6)
- [ ] **API-07**: Implement export DTOs and methods: `ExportPublicKey`, `ExportPrivateKey`; backend verifies vault unlock state before export (PRD §10.7)
- [ ] **API-08**: Define `AppErrorCode` typed enum DTO; map all internal errors to safe public errors at the Wails boundary (PRD §10.1, §12)
- [ ] **API-09**: Frontend uses `code` field, not string matching, for error handling (PRD §10.1, §18.1 hard rule #14)

### UI (UI)

PRD §9.

- [ ] **UI-01**: Welcome / Vault Selection screen with Create-vault, Open-existing-vault, View-security-limitations actions (PRD §9.2)
- [ ] **UI-02**: Create Vault screen with vault path, master password, confirm-master-password, no-recovery warning, and ≥12-char password validation (PRD §9.3)
- [ ] **UI-03**: Unlock Vault screen — generic failure message; do not reveal failure source (PRD §9.4)
- [ ] **UI-04**: Vault Home screen — header with vault status, lock button, search input, record-type filter, record list, New-Record / Generate-Password / Generate-Key / Settings buttons (PRD §9.5)
- [ ] **UI-05**: Record Detail screen with metadata display, masked-by-default secret fields, reveal/copy/edit/delete actions, and private-key warning gate (PRD §9.6)
- [ ] **UI-06**: New / Edit Record screen for each of the 5 record types (PRD §7)
- [ ] **UI-07**: Password Generator screen
- [ ] **UI-08**: Key Generator screen (AES / RSA / Ed25519)
- [ ] **UI-09**: Settings screen — clipboard timeout, idle auto-lock timeout, theme preference, last-selected record-type filter (PRD §6.3 settings)
- [ ] **UI-10**: About / Security Model screen
- [ ] **UI-11**: Clipboard countdown component — full → empty progress bar, ≥1 Hz updates, "Clear now" button, resets on new copy, clears on lock (PRD §9.7)
- [ ] **UI-12**: Private Key Exposure Warning modal — copy and export variants with explicit "I Understand…" confirmation buttons (PRD §9.8)
- [ ] **UI-13**: Locked vs unlocked state is visually unambiguous (PRD §9.5)

### Security Infrastructure (SEC)

PRD §4, §5, §12.

- [ ] **SEC-01**: Argon2id KDF with documented profile (target 256MiB / 3 iters / 4 parallelism, fallback 64MiB / 3 / 1, minimum 19MiB / 2 / 1); per-wrapping params and salt (PRD §5.3)
- [ ] **SEC-02**: KDF parameter benchmark / tuning helper run on slowest supported dev machine (PRD §5.3)
- [ ] **SEC-03**: AAD on every AES-GCM encrypt/decrypt operation; AAD strings exactly per PRD §5.4 (record metadata, record secret, wrapped DEK)
- [ ] **SEC-04**: AAD does NOT include SQLite `schema_version` (PRD §5.4); ordinary migrations must not force record re-encryption
- [ ] **SEC-05**: Unique `crypto/rand` nonce per AES-GCM encryption; never reuse with the same key; nonces stored alongside ciphertext (PRD §5.5)
- [ ] **SEC-06**: Typed `AppErrorCode` enum used at the Wails boundary (PRD §10.1, §12)
- [ ] **SEC-07**: Sanitized logging at service boundaries — never log master passwords, KEKs, DEKs, plaintext records, generated passwords/keys, clipboard contents (PRD §12.2)
- [ ] **SEC-08**: User-facing errors are clear, safe, non-leaky; never expose raw crypto errors, SQL internals, stack traces, master-password details, KDF internals, or decrypted payloads (PRD §12.1)
- [ ] **SEC-09**: Fail-closed on dirty migration state (PRD §2.5)
- [ ] **SEC-10**: Best-effort byte-slice zeroing of sensitive material; clear session state on lock; no unnecessary copies; document Go memory limitations in README (PRD §4.4)

### Storage (STORE)

PRD §6.

- [ ] **STORE-01**: SQLite `application_id` = `0x4E56544E` set at vault creation; checked at every open (FR-003, PRD §6.1)
- [ ] **STORE-02**: Rollback journal mode (`PRAGMA journal_mode = DELETE`); `PRAGMA foreign_keys = ON` (PRD §6.2)
- [ ] **STORE-03**: `vault_metadata` table (single-row, holds `vault_id`, `schema_version`, `record_envelope_version`) (PRD §6.3)
- [ ] **STORE-04**: `deks` table (DEK identity + lifecycle metadata; no plaintext DEK) (PRD §6.3)
- [ ] **STORE-05**: `key_wrappings` table (KDF params, salt, wrapped DEK, wrapped DEK nonce, FK to `deks`) (PRD §6.3)
- [ ] **STORE-06**: `records` table — encrypted metadata blob + encrypted secret blob, optional plaintext public key columns, FK to `deks` (PRD §6.3)
- [ ] **STORE-07**: `settings` table — non-sensitive UI preferences only (PRD §6.3)
- [ ] **STORE-08**: SQLite vault file contains no plaintext sensitive material — title, notes, tags, username, URL, API keys, passwords, AES keys, private keys never in plaintext columns (PRD §6.3, §14.3)

### Cross-Platform (XPLAT)

PRD §2.3, §14.3.

- [ ] **XPLAT-01**: App builds and runs on macOS
- [ ] **XPLAT-02**: App builds and runs on Windows
- [ ] **XPLAT-03**: Manual macOS + Windows test matrix passes (PRD §14.3 — 22-row matrix covering vault create / open / unlock / wrong-password / change-master / record CRUD / generators / clipboard / auto-lock / export / lock / restart / no-plaintext)
- [ ] **XPLAT-04**: Documented macOS and Windows setup, build, and packaging notes in README (PRD §19)
- [ ] **XPLAT-05**: Cross-OS verification at milestone boundaries — do not defer to the end (PRD §2.3)

### Testing (TEST)

PRD §14.

- [ ] **TEST-01**: Go unit tests covering Argon2id parameter handling, KEK derivation, DEK generation, DEK wrap/unwrap, wrong-password unwrap failure, AES-GCM encrypt/decrypt, AAD mismatch failure, nonce generation, record encrypt/decrypt, record tamper detection, password generator, AES/RSA/Ed25519 generators, OpenSSH public key export, PKCS#8 private key export, SQLite `application_id` validation, migration application, master-password-change re-wrap (PRD §14.1)
- [ ] **TEST-02**: Frontend tests covering Create/Unlock/Change-Master forms, record forms, password generator options, key generator options, clipboard countdown component, private-key warning modal, locked-state action blocking, auto-lock state transition (PRD §14.2)

### Documentation (DOC)

PRD §19.

- [ ] **DOC-01**: README — project purpose, dev setup (macOS + Windows), build instructions, test instructions
- [ ] **DOC-02**: README — vault security model and threat model (PRD §4)
- [ ] **DOC-03**: README — required no-recovery warning, clipboard limitations warning, Go memory limitations statement (PRD §4.4, §19)
- [ ] **DOC-04**: README — private key export warning, known limitations, non-goals (PRD §19)
- [ ] **DOC-05**: Required learning-grade disclaimer in README (PRD §19)

## v2 Requirements

Deferred beyond v1.0. Acknowledged but not in current roadmap.

### Clipboard

- **CLIP-V2-01**: Optional harmless overwrite of clipboard before clearing (PRD FR-064 Could-level)

### Records / UX

- **REC-V2-01**: Recent vaults list on Welcome screen (PRD §9.2 deferred unless implementation is trivial)
- **REC-V2-02**: Locked-state public key ring — view/copy public keys without unlocking (PRD §2.9 future improvement)

### Cryptography

- **CRYPTO-V2-01**: ECDSA key generation (PRD §5.1 deferred)
- **CRYPTO-V2-02**: X25519 key generation (PRD §5.1 deferred)
- **CRYPTO-V2-03**: TOTP secret records (PRD §5.1 deferred)
- **CRYPTO-V2-04**: DEK rotation workflow — `deks` table already supports it (PRD §6.3)

## Out of Scope

Explicitly excluded. PRD §3.3, §18 Hard Rules, plus PRD §2 architectural locks.

| Feature | Reason |
|---------|--------|
| Cloud sync | Local-first product; sync is a different threat model (PRD §3.3) |
| Browser extension | Out of MVP scope; large attack surface (PRD §3.3) |
| Browser autofill | Hard rule #2 — explicitly forbidden (PRD §18) |
| Mobile app | Desktop-only learning project (PRD §3.3) |
| Team / shared / multi-user vaults | Single-user product; hard rules #3, #4 (PRD §18) |
| Biometric unlock | Master-password-only for MVP (PRD §3.3) |
| Hardware security key unlock | Out of MVP scope (PRD §3.3) |
| TPM / Secure Enclave integration | Out of MVP scope (PRD §3.3) |
| Password breach monitoring | Requires network; out of scope (PRD §3.3) |
| TOTP authenticator app | Out of MVP scope (PRD §3.3) |
| SSH agent integration | Out of MVP scope (PRD §3.3) |
| PGP key management | Out of MVP scope (PRD §3.3) |
| Auto-update | Manual builds for MVP (PRD §3.3) |
| Enterprise audit logs | Single-user, learning-grade product (PRD §3.3) |
| Plugin system | Locked architecture; hard rule equivalent (PRD §3.3) |
| Linux packaging | macOS + Windows only for MVP; hard rule #5 (PRD §2.3, §18) |
| Formal security audit | Learning-grade positioning (PRD §3.3, §4.1) |
| Next.js / SSR / server-side frontend | Hard rules #10, #11 (PRD §2.2, §18) |
| WAL journal mode | Hard rule #12 — rollback journal preferred for vault portability (PRD §2.4, §18) |
| SQLCipher | Record-level encryption preferred over whole-DB encryption (PRD §2.4) |
| Custom cryptographic algorithms | Hard rule #8 — use Go stdlib + `golang.org/x/crypto` (PRD §18) |
| `math/rand` for any security material | Hard rule #13 — `crypto/rand` only (PRD §18) |
| Custom password strength estimator | Use a vetted zxcvbn-style library if any (PRD §8.3, §18 preferred) |
| Plaintext secrets in SQLite | Hard rule #6 — never (PRD §18) |
| Logging secrets | Hard rule #7 — never (PRD §18) |
| String-matching errors in frontend | Hard rule #14 — use typed codes (PRD §18) |
| Coupling schema migrations to record decryption | Hard rule #15 — AAD excludes `schema_version` (PRD §18) |
| Exposing private keys without warning | Hard rule #9 — exposure modal required (PRD §18) |

## Traceability

Each v1 REQ-ID maps to exactly one phase. Phases derive from PRD §15 milestones (v0.1 → v1.0). Status reflects current execution state.

| Requirement | Phase | Status |
|-------------|-------|--------|
| VAULT-01 | Phase 1 | Pending |
| VAULT-02 | Phase 1 | Pending |
| VAULT-03 | Phase 1 | Pending |
| VAULT-04 | Phase 1 | Pending |
| VAULT-05 | Phase 1 | Pending |
| VAULT-06 | Phase 1 | Pending |
| VAULT-07 | Phase 1 | Pending |
| VAULT-08 | Phase 2 | Pending |
| VAULT-09 | Phase 2 | Pending |
| VAULT-10 | Phase 1 | Pending |
| VAULT-11 | Phase 1 | Pending |
| VAULT-12 | Phase 1 | Pending |
| VAULT-13 | Phase 5 | Pending |
| REC-01 | Phase 1 | Pending |
| REC-02 | Phase 3 | Pending |
| REC-03 | Phase 3 | Pending |
| REC-04 | Phase 3 | Pending |
| REC-05 | Phase 4 | Pending |
| REC-06 | Phase 1 | Pending |
| REC-07 | Phase 1 | Pending |
| REC-08 | Phase 1 | Pending |
| REC-09 | Phase 1 | Pending |
| REC-10 | Phase 1 | Pending |
| REC-11 | Phase 1 | Pending |
| REC-12 | Phase 1 | Pending |
| REC-13 | Phase 1 | Pending |
| PASS-01 | Phase 1 | Pending |
| PASS-02 | Phase 1 | Pending |
| PASS-03 | Phase 1 | Pending |
| PASS-04 | Phase 1 | Pending |
| PASS-05 | Phase 1 | Pending |
| PASS-06 | Phase 1 | Pending |
| PASS-07 | Phase 6 | Pending |
| AES-01 | Phase 4 | Pending |
| AES-02 | Phase 4 | Pending |
| AES-03 | Phase 4 | Pending |
| AES-04 | Phase 4 | Pending |
| AES-05 | Phase 4 | Pending |
| AES-06 | Phase 4 | Pending |
| AES-07 | Phase 4 | Pending |
| RSA-01 | Phase 4 | Pending |
| RSA-02 | Phase 4 | Pending |
| RSA-03 | Phase 4 | Pending |
| RSA-04 | Phase 4 | Pending |
| RSA-05 | Phase 4 | Pending |
| RSA-06 | Phase 4 | Pending |
| RSA-07 | Phase 4 | Pending |
| RSA-08 | Phase 4 | Pending |
| ED-01 | Phase 4 | Pending |
| ED-02 | Phase 4 | Pending |
| ED-03 | Phase 4 | Pending |
| ED-04 | Phase 4 | Pending |
| ED-05 | Phase 4 | Pending |
| ED-06 | Phase 4 | Pending |
| ED-07 | Phase 4 | Pending |
| ED-08 | Phase 4 | Pending |
| CLIP-01 | Phase 1 | Pending |
| CLIP-02 | Phase 1 | Pending |
| CLIP-03 | Phase 1 | Pending |
| CLIP-04 | Phase 1 | Pending |
| CLIP-05 | Phase 1 | Pending |
| CLIP-06 | Phase 2 | Pending |
| CLIP-07 | Phase 6 | Pending |
| API-01 | Phase 1 | Pending |
| API-02 | Phase 1 | Pending |
| API-03 | Phase 1 | Pending |
| API-04 | Phase 1 | Pending |
| API-05 | Phase 4 | Pending |
| API-06 | Phase 1 | Pending |
| API-07 | Phase 4 | Pending |
| API-08 | Phase 1 | Pending |
| API-09 | Phase 1 | Pending |
| UI-01 | Phase 1 | Pending |
| UI-02 | Phase 1 | Pending |
| UI-03 | Phase 1 | Pending |
| UI-04 | Phase 1 | Pending |
| UI-05 | Phase 1 | Pending |
| UI-06 | Phase 4 | Pending |
| UI-07 | Phase 1 | Pending |
| UI-08 | Phase 4 | Pending |
| UI-09 | Phase 6 | Pending |
| UI-10 | Phase 6 | Pending |
| UI-11 | Phase 1 | Pending |
| UI-12 | Phase 4 | Pending |
| UI-13 | Phase 1 | Pending |
| SEC-01 | Phase 1 | Pending |
| SEC-02 | Phase 2 | Pending |
| SEC-03 | Phase 1 | Pending |
| SEC-04 | Phase 1 | Pending |
| SEC-05 | Phase 1 | Pending |
| SEC-06 | Phase 1 | Pending |
| SEC-07 | Phase 1 | Pending |
| SEC-08 | Phase 1 | Pending |
| SEC-09 | Phase 1 | Pending |
| SEC-10 | Phase 2 | Pending |
| STORE-01 | Phase 1 | Pending |
| STORE-02 | Phase 1 | Pending |
| STORE-03 | Phase 1 | Pending |
| STORE-04 | Phase 1 | Pending |
| STORE-05 | Phase 1 | Pending |
| STORE-06 | Phase 1 | Pending |
| STORE-07 | Phase 1 | Pending |
| STORE-08 | Phase 1 | Pending |
| XPLAT-01 | Phase 5 | Pending |
| XPLAT-02 | Phase 5 | Pending |
| XPLAT-03 | Phase 5 | Pending |
| XPLAT-04 | Phase 5 | Pending |
| XPLAT-05 | Phase 5 | Pending |
| TEST-01 | Phase 2 | Pending |
| TEST-02 | Phase 6 | Pending |
| DOC-01 | Phase 6 | Pending |
| DOC-02 | Phase 6 | Pending |
| DOC-03 | Phase 6 | Pending |
| DOC-04 | Phase 6 | Pending |
| DOC-05 | Phase 6 | Pending |

**Coverage:**
- v1 requirements: 115 total (corrected from preliminary 113 count during roadmap creation; PASS-07 and CLIP-07 were undercounted)
- v2 requirements: 6 total
- Out of scope: 28 categorical exclusions
- Mapped to phases: 115 / 115 (100%)

**Per-phase requirement count:**

| Phase | Count | Theme |
|-------|-------|-------|
| Phase 1 — v0.1 Dogfoodable Vault | 61 | Vault core, credential records, password generator, clipboard countdown, ALL security scaffolding |
| Phase 2 — v0.2 Security Foundation | 6 | Auto-lock, change master password, KDF benchmark, memory hygiene, full Go test pass |
| Phase 3 — v0.3 Expanded Record Types | 3 | API key, secure note, symmetric key record types |
| Phase 4 — v0.4 Key Generation and Export | 29 | AES/RSA/Ed25519 generators, asymmetric records, public/private key export, warning modal |
| Phase 5 — v0.5 Cross-Platform Hardening | 6 | macOS+Windows verification, lock-on-system-sleep, packaging notes |
| Phase 6 — v1.0 MVP Complete | 10 | Frontend tests, README threat model, Settings/About screens, password strength meter |
| **Total** | **115** | |

---
*Requirements defined: 2026-04-29*
*Last updated: 2026-04-29 after roadmap creation; traceability populated by `gsd-roadmapper`.*
