# Abyss — Local Password & Key Vault

## What This Is

Abyss is a local-first desktop password and cryptographic key vault for macOS and Windows. It lets a single user create and unlock an encrypted SQLite vault, store credentials and keys, generate strong passwords and AES/RSA/Ed25519 keys, and safely copy or export selected secrets. The product target is a learning-grade but functionally useful personal vault — not a commercial password-manager competitor.

## Core Value

A user can create an encrypted local vault with a master password, store and retrieve secrets, and never see plaintext sensitive data persisted to disk. If only one thing must work, it is: **vault confidentiality and tamper-evidence (DEK/KEK envelope encryption with AES-256-GCM AAD), end-to-end, on both macOS and Windows.**

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope. Building toward MVP v1.0 across 6 milestones (v0.1 → v1.0). -->

**Vault lifecycle**
- [ ] Create a new SQLite vault with `application_id`, `vault_id`, active DEK, master-password-wrapped DEK
- [ ] Unlock vault with master password (Argon2id KEK derivation, AES-256-GCM unwrap, generic failure)
- [ ] Lock vault manually
- [ ] Auto-lock vault after configurable idle timeout (default 10 min)
- [ ] Change master password (re-wrap DEK without re-encrypting records)
- [ ] Clear decrypted state and attempt clipboard clear on lock
- [ ] Reject all secret actions while locked

**Record management (5 record types)**
- [ ] Create / view / edit / delete credential records
- [ ] Create / view / edit / delete API key records
- [ ] Create / view / edit / delete secure note records
- [ ] Create / view / edit / delete symmetric (AES) key records
- [ ] Create / view / edit / delete asymmetric (RSA / Ed25519) key pair records
- [ ] Search records in memory after unlock
- [ ] Mask secrets by default; temporary reveal; copy selected fields
- [ ] Encrypted tags

**Generators**
- [ ] Password generator (configurable length / classes / ambiguous-exclusion, defaults length=24)
- [ ] AES key generator (128 / 192 / 256, base64 + hex display)
- [ ] RSA key pair generator (2048 / 3072 / 4096; default 3072)
- [ ] Ed25519 key pair generator
- [ ] Save generated material as a vault record

**Key export**
- [ ] Export public key in OpenSSH format
- [ ] Export public key in PEM format
- [ ] Export private key in PKCS#8 PEM format
- [ ] Private key copy/export gated by exposure-warning modal

**Clipboard**
- [ ] Copy secrets with countdown progress bar (10/30/60/120s; default 30s)
- [ ] Attempt clipboard clear on timeout, on manual clear, and on lock
- [ ] Reset countdown when a new secret is copied

**Cross-platform**
- [ ] App builds and runs on macOS
- [ ] App builds and runs on Windows
- [ ] Manual cross-platform test matrix passes

**Security infrastructure**
- [ ] Typed `AppErrorCode` DTO across Wails boundary; no string-matching errors in frontend
- [ ] Sanitized logging (no master passwords, KEKs, DEKs, plaintext records, generated secrets, clipboard contents)
- [ ] AAD enforcement for all AES-GCM operations (record metadata, record secret, wrapped DEK)
- [ ] AAD-tampering tests pass
- [ ] `golang-migrate` schema migrations embedded into the binary
- [ ] README documents setup, threat model, no-recovery warning, clipboard limitations, Go memory limitations

### Out of Scope

<!-- Explicit boundaries from PRD §3.3, §18. -->

- **Cloud sync** — local-first product; sync is a different threat model
- **Browser extension / autofill** — out of MVP scope; adds large attack surface
- **Mobile app** — desktop-only learning project
- **Team / shared / multi-user vaults** — single-user product
- **Biometric / hardware-key / TPM / Secure Enclave unlock** — master-password-only for MVP
- **Password breach monitoring** — requires network
- **TOTP authenticator** — explicit non-goal for MVP
- **SSH agent integration** — explicit non-goal for MVP
- **PGP key management** — explicit non-goal for MVP
- **Auto-update** — manual builds for MVP
- **Plugin system** — locked architecture, no extension points in MVP
- **Linux packaging** — macOS + Windows only for MVP (PRD §2.3)
- **Formal security audit** — learning-grade product; explicit non-goal
- **Next.js / SSR** — explicitly forbidden (PRD §2.2, §18.1)
- **WAL journal mode** — rollback journal preferred for vault portability (PRD §2.4)
- **SQLCipher** — record-level encryption preferred over whole-DB encryption (PRD §2.4)
- **Custom cryptographic algorithms** — use Go stdlib + `golang.org/x/crypto` only (PRD §18.1)
- **`math/rand` for any security-sensitive randomness** — `crypto/rand` only (PRD §18.1)
- **Custom password strength estimator** — use a vetted zxcvbn-style library if any (PRD §8.3)
- **ECDSA / X25519 / TOTP secret records** — deferred (PRD §5.1)

## Context

**Why this project exists.** Primary purpose is to learn Go + Wails v2 by building a real, functional, local-first product (PRD §1, §3.2). Learning goals span Wails method bindings, Go service-layer architecture, SQLite + migrations, authenticated encryption, Argon2id KDF, DEK/KEK envelope encryption, desktop clipboard handling, cross-platform packaging, React/Mantine UI, and frontend↔backend error boundary design.

**Architecture is locked.** PRD §2 ("Locked Architecture Decisions") and §18 ("Hard Rules") fix the stack, threat model, encryption model, journal mode, and forbidden technologies. The roadmap and plans must respect these — they are constraints, not options.

**Cryptographic surface is fully specified.** PRD §5 specifies AES-256-GCM for records, Argon2id (target 256MiB / 3 iters / 4 parallelism, with documented fallback profiles), per-wrapping salt, AAD strings for record metadata / record secret / wrapped DEK. AAD must NOT include `schema_version` so ordinary migrations don't force re-encryption.

**SQLite shape is fully specified.** PRD §6 specifies tables (`vault_metadata`, `deks`, `key_wrappings`, `records`, `settings`), `application_id` = 0x4E56544E, rollback journal mode, and which fields stay plaintext (public key material) vs encrypted (everything else).

**Backend↔Frontend contract is locked.** PRD §10 specifies all Wails method signatures and DTOs; PRD §10.1 defines the `AppErrorCode` enum used at the boundary.

**Threat model is explicit.** PRD §4 spells out what is and isn't protected against. Notably out of scope: malware, keyloggers, screen capture, memory scraping, third-party clipboard managers. The unlocked session is the primary memory-exposure window; auto-lock is the mitigation. Go memory limitations (immutable strings, Wails binding boundary) are acknowledged in the README requirement.

**Build order is sequenced.** PRD §20 specifies a 20-step implementation sequence and PRD §15 groups it into 6 milestones (v0.1 Dogfoodable Vault → v1.0 MVP Complete). The roadmap should align with these milestones.

## Constraints

- **Tech stack — Desktop framework**: Go + Wails v2 stable. Wails v3 alpha is forbidden. (PRD §2.1)
- **Tech stack — Frontend**: Vite + React + TypeScript + Mantine. Next.js / SSR is forbidden. (PRD §2.2, §18.1)
- **Tech stack — Storage**: SQLite with rollback journal. WAL mode and SQLCipher are forbidden for MVP. (PRD §2.4)
- **Tech stack — Migrations**: `golang-migrate`, embedded into the Go binary. (PRD §2.5)
- **Tech stack — Crypto**: AES-256-GCM, Argon2id, `crypto/rand`, `golang.org/x/crypto/ssh` for OpenSSH marshaling. No custom algorithms. No `math/rand`. (PRD §5, §18)
- **Platform**: macOS + Windows. Linux is out of scope for MVP. Cross-OS verification at milestone boundaries, not deferred to the end. (PRD §2.3)
- **Security — encryption model**: DEK/KEK envelope encryption. DEK random; KEK = Argon2id(master_password, salt). Records encrypted with DEK. DEK wrapped by KEK. Wrapped DEK is the password verifier. AAD on all AES-GCM operations per PRD §5.4. (PRD §2.6, §5)
- **Security — error and logging policy**: Typed `AppErrorCode` DTO at the Wails boundary. Sanitized logs — never log master passwords, KEKs, DEKs, plaintext records, generated secrets, clipboard contents. No string-matching errors in the frontend. (PRD §10.1, §12, §18)
- **Security — clipboard**: 30s default timeout, configurable {10, 30, 60, 120}. Frontend owns countdown UI; backend owns clipboard write/clear. Best-effort only; document clipboard-manager limitations. (PRD §2.7, §8.7)
- **Security — private keys**: Copy/export only when vault unlocked AND user confirms exposure-warning modal. (PRD §2.8)
- **UX**: 10 primary screens enumerated in PRD §9.1. Unlock failure shows generic error (no oracle). Vault lock state must be visually unambiguous.
- **Testing**: Required Go unit tests, frontend tests, and manual cross-platform test matrix per PRD §14. SQLite must contain no plaintext sensitive material.

## Key Decisions

<!-- Frozen by PRD §2 and §18. Documenting them here so they are visible to planning. -->

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Go + Wails v2 (not v3 alpha, not Tauri/Electron) | Wails v2 is the explicit learning target; v2 is the stable line | — Pending |
| Vite + React + TS + Mantine (not Next.js) | Inside Wails the frontend is a static SPA; SSR / server components add complexity with no benefit | — Pending |
| SQLite + record-level encryption (not SQLCipher / whole-DB) | Vault portability matters more than throughput; users may copy/zip/move vault files | — Pending |
| Rollback journal mode (not WAL) | Avoids sidecar checkpoint state; safer default for a portable file | — Pending |
| `golang-migrate` (not hand-rolled) | Versioned, repeatable, embeddable; only deviate if Wails packaging breaks | — Pending |
| DEK/KEK envelope encryption with AES-256-GCM + AAD | Industry-standard envelope model; AAD prevents record-swap tampering; supports master-password change without re-encrypting records | — Pending |
| Argon2id with documented profile + fallback | Memory-hard KDF; per-wrapping params allow tuning per machine | — Pending |
| AAD excludes SQLite `schema_version` | Schema migrations must NOT force record re-encryption unless the record envelope itself changes | — Pending |
| Public keys stored plaintext, but record interaction gated by unlock | UX simplification; not a security requirement; revisit in a future version | — Pending |
| Clipboard countdown owned by frontend; clipboard read/write owned by Go | Clean separation; backend does the OS-sensitive work | — Pending |
| MVP excludes biometric / TPM / Secure Enclave unlock | Master-password-only is sufficient for the learning-grade scope | — Pending |
| Source-of-truth document is `docs/PRD.md` v1.2 (Architecture-Locked MVP Build Contract) | Build contract was authored before GSD initialization; planning derives from it | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-29 after initialization (auto mode from docs/PRD.md v1.2)*
