# Roadmap: Abyss — Local Password & Key Vault

## Overview

Abyss is a local-first Go + Wails v2 desktop password and key vault for macOS and Windows. The build is sequenced through six PRD-defined milestones (v0.1 Dogfoodable Vault → v1.0 MVP Complete) that map 1:1 to roadmap phases. Phase 1 carries the heaviest load: 14 of 23 identified pitfalls have their primary prevention window in v0.1, and security-critical scaffolding (AAD helper, typed-error DTO, sanitized logger, lock-state machine, fail-closed migrations, frontend api wrapper, CI lint gates) MUST exist before the first record is ever encrypted. Subsequent phases harden, expand record types, add cryptographic key generation and export, validate cross-platform behavior, and ship documentation. The authoritative source-of-truth document is `docs/PRD.md` v1.2; this roadmap is a derivative artifact.

**Granularity:** coarse (3–5 broad phases per GSD config; the 6 PRD milestones are the natural unit of work, kept at coarse plan-density of 1–3 plans per phase).

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work, one per PRD §15 milestone
- Decimal phases (e.g., 2.1): Reserved for urgent insertions via `/gsd-insert-phase`

- [ ] **Phase 1: v0.1 Dogfoodable Vault** - First usable local vault with credentials, password generator, clipboard countdown, and ALL security-critical scaffolding
- [ ] **Phase 2: v0.2 Security Foundation** - Auto-lock, master-password change, KDF benchmarking, full memory hygiene, AAD-tampering tests, full crypto test pass
- [ ] **Phase 3: v0.3 Expanded Record Types** - API key, secure note, and symmetric (AES) key record types built on the proven encryption pipeline
- [ ] **Phase 4: v0.4 Key Generation and Export** - AES/RSA/Ed25519 generators, asymmetric key pair records, OpenSSH/PEM/PKCS#8 export, private-key warning modal
- [ ] **Phase 5: v0.5 Cross-Platform Hardening** - Full macOS+Windows verification, lock-on-system-sleep, packaging notes, code-signing/SmartScreen documentation
- [ ] **Phase 6: v1.0 MVP Complete** - Frontend test suite, README threat model, Settings/About screens, password strength meter, final 22-scenario manual test matrix

## Phase Details

### Phase 1: v0.1 Dogfoodable Vault
**Goal**: User can create, unlock, and use an encrypted local vault for credentials and generated passwords on the primary dev OS, with all security-critical scaffolding in place.
**Depends on**: Nothing (first phase)
**Requirements**: VAULT-01, VAULT-02, VAULT-03, VAULT-04, VAULT-05, VAULT-06, VAULT-07, VAULT-10, VAULT-11, VAULT-12, REC-01, REC-06, REC-07, REC-08, REC-09, REC-10, REC-11, REC-12, REC-13, PASS-01, PASS-02, PASS-03, PASS-04, PASS-05, PASS-06, CLIP-01, CLIP-02, CLIP-03, CLIP-04, CLIP-05, API-01, API-02, API-03, API-04, API-06, API-08, API-09, UI-01, UI-02, UI-03, UI-04, UI-05, UI-07, UI-11, UI-13, SEC-01, SEC-03, SEC-04, SEC-05, SEC-06, SEC-07, SEC-08, SEC-09, STORE-01, STORE-02, STORE-03, STORE-04, STORE-05, STORE-06, STORE-07, STORE-08
**Success Criteria** (what must be TRUE):
  1. User can create a new vault file and unlock it with a master password (≥12 chars), receiving a generic failure message on any wrong-password attempt.
  2. User can create, view, edit, search, and delete credential records; secrets are masked by default and revealable on demand.
  3. SQLite vault file contains no plaintext sensitive material — title, username, password, notes, tags, URL never appear in plaintext columns (verified by automated grep test in CI on every PR, not only at milestone boundary).
  4. User can generate a configurable password (length 24 default, classes/exclude-ambiguous toggles), copy it with a visible countdown progress bar, and save it as a credential.
  5. Locking the vault clears the in-memory DEK, decrypted record state, and best-effort clipboard contents (verified by Go test); locked vault rejects all secret-touching API calls with `VAULT_LOCKED`.
  6. App builds and runs on the primary dev OS, AND a Windows build smoke-test succeeds at end of phase (per PRD §2.3 cross-OS verification).
**Plans**: TBD
**UI hint**: yes

**Notes for plan-phase researcher:**
- This phase carries the SCAFFOLDING that cannot be retrofitted later. The plans must include: `internal/aad/` with golden-vector tests (AAD must ship in the FIRST `cipher.Seal` call — PRD §15 lists "AAD enforcement" as a v0.2 EXIT CRITERION but that is a verification checkpoint, not permission to defer); `internal/apperr/` with `AppError{Code,Message,Cause}` and `ToDTO()`; `internal/session/` with `RequireUnlocked()` and atomic DEK zero-on-lock; `internal/logger/` slog wrapper with redaction list; `frontend/src/api/` typed wrapper layer with `normalizeError`; migration fail-closed on dirty `schema_migrations`; `wails generate module` CI gate to catch DTO drift; `errcheck` and `forbidigo`/grep ban on `math/rand` in CI.
- Open research questions to resolve at this phase's planning: (a) confirm exact import path for the pure-Go SQLite database driver in `golang-migrate/v4` (`database/sqlite` vs `database/sqlite3`); after wiring, run `go mod why github.com/mattn/go-sqlite3` to confirm CGO has not crept in; (b) verify DSN params that set `journal_mode=DELETE` and `foreign_keys=ON`; (c) decide Mantine v8.3.18 vs v9 — recommendation is v8.3.18, re-evaluate at v1.0; (d) decide whether PASS-07 zxcvbn strength meter ships here or in Phase 6 — currently mapped to Phase 6.
- AAD must NOT include `schema_version` (PRD §5.4); AAD format is canonical per `internal/aad/Build()`. This phase establishes AAD; Phase 2 adds AAD-tampering exit-criterion verification tests.
- REC-13 (encrypted tags) is promoted from PRD §15 v0.3 to this phase per research finding: tags live inside the already-encrypted `metadata_blob`, so adding them costs almost nothing once metadata encryption is wired.
- CLIP-04 (manual "Clear now" button) is promoted from Should-have to this phase per research: it extends the same `UI-11` countdown component being built here.
- API-03 record DTOs are scoped to credential-only here; Phase 3 extends to api_key/secure_note/symmetric_key, Phase 4 extends to asymmetric. The DTO surface (`CreateRecord`, `GetRecord`, `UpdateRecord`, `DeleteRecord`, `ListRecords`, `SearchRecords`) ships now.
- UI-06 (full 5-type record forms) is intentionally NOT here — Phase 1 implements credential form as part of UI-04/UI-05 work. UI-06 lives in Phase 4 where all 5 types exist.
- Cross-OS smoke at phase end: at minimum a `wails build -platform windows/amd64` and a manual unlock-create-credential pass. Path bugs caught now are cheaper than path bugs caught at Phase 5.

### Phase 2: v0.2 Security Foundation
**Goal**: Vault security guarantees are verified by tests and operational mitigations: idle auto-lock, master-password change without record re-encryption, AAD-tampering detection, KDF tuned to the slowest dev machine, and best-effort memory hygiene.
**Depends on**: Phase 1
**Requirements**: VAULT-08, VAULT-09, CLIP-06, SEC-02, SEC-10, TEST-01
**Success Criteria** (what must be TRUE):
  1. Vault auto-locks after the configured idle timeout (default 10 min; allowed 1/5/10/15/30 min, plus Disabled-with-warning); idle timer survives system sleep (wall-clock poller, not single `time.AfterFunc`).
  2. User can change the master password; verified by Go test that the same DEK survives, no record ciphertext changes, and the wrapped-DEK row is replaced with a fresh salt and new KDF params.
  3. AAD tampering is detected: swapping the encrypted secret blob between two records, swapping `record_type`, or swapping `dek_id` causes decrypt to fail (verified by Go tests in `internal/aad` and `internal/crypto`).
  4. Argon2id KDF parameters are documented and benchmarked on the slowest supported dev machine; chosen profile keeps unlock under ~1 second; profile is recorded in README and per-wrapping `kdf_params_json`.
  5. Lock event clears in-memory DEK with `crypto.Zero` before nil assignment; clipboard clear is attempted; frontend stores are wiped via `vault:locked` event.
  6. Required Go unit tests (PRD §14.1) pass: KEK/DEK derivation and wrap/unwrap, wrong-password failure, AES-GCM encrypt/decrypt, AAD mismatch failure, nonce uniqueness, record encrypt/decrypt with tamper detection, password generator, application_id validation, migration apply, master-password-change re-wrap.
**Plans**: TBD

**Notes for plan-phase researcher:**
- Idle timer topology per ARCHITECTURE.md §6: frontend debounces user-input pings to backend (≥1s); backend owns the actual countdown and `Lock()` call; backend emits `vault:locked` event via `runtime.EventsEmit`; frontend subscribes once at App root.
- Open research question to resolve at this phase's planning: should clipboard / no-recovery warnings be surfaced inside the app (Settings or About screen) and not only in README? PRD §19 puts them in README only. Decide here so Phase 6 documentation aligns.
- Lock-on-system-sleep (VAULT-13) is NOT in this phase; deferred to Phase 5 because IOKit / WM_WTSSESSION_CHANGE hooks are platform-specific and best researched alongside cross-platform hardening.
- TEST-01 is the consolidated Go-unit-test deliverable. Initial subset tests are written in Phase 1 implementation; this phase ensures the full PRD §14.1 list is green.

### Phase 3: v0.3 Expanded Record Types
**Goal**: User can store and manage all non-asymmetric record types (API keys, secure notes, symmetric keys) using the same encryption pipeline that ships in Phase 1.
**Depends on**: Phase 2
**Requirements**: REC-02, REC-03, REC-04
**Success Criteria** (what must be TRUE):
  1. User can create, view, edit, and delete API key records (key value encrypted; service name, environment, expires_at inside encrypted metadata blob).
  2. User can create, view, edit, and delete secure note records (note body encrypted; title inside encrypted metadata blob).
  3. User can create, view, edit, and delete symmetric (AES) key records by pasting an existing key value (in-app generation arrives in Phase 4); key bytes encrypted; algorithm/key_length inside encrypted metadata blob.
  4. SQLite grep test still passes: title, key value, note body, API key, environment, tags never appear in plaintext.
  5. App builds and runs on macOS AND Windows at end of phase (cross-OS verification per PRD §2.3).
**Plans**: TBD

**Notes for plan-phase researcher:**
- The metadata blob JSON format gets new optional fields per record type (PRD §7.2/§7.3/§7.4). Per Pitfall 21, all new fields MUST be optional (`omitempty`); type changes get new field names. This is the first migration of the metadata-blob schema — verify the Phase 1 JSON marshal/unmarshal helpers tolerate forward-and-backward compatibility.
- REC-04 (symmetric key record type) lands here without a generator; user pastes an existing key value. AES-* generators arrive in Phase 4 and call into the symmetric-record CRUD established here. This split follows PRD §15 v0.3 vs v0.4 scope exactly.
- REC-09 (in-memory search) and REC-13 (encrypted tags) were promoted to Phase 1; this phase only delivers types 2/3/4. UI-06 (full 5-type record forms) lives in Phase 4 where the asymmetric type rounds out the set.
- Cross-OS smoke at phase end: rebuild on the secondary OS, verify all four record types CRUD-roundtrip on both.

### Phase 4: v0.4 Key Generation and Export
**Goal**: User can generate AES symmetric keys and RSA/Ed25519 asymmetric key pairs in-app, save them to the vault as records, and export public keys (OpenSSH, PEM) and private keys (PKCS#8 PEM) — with private-key export gated by an exposure-warning modal.
**Depends on**: Phase 3
**Requirements**: REC-05, AES-01, AES-02, AES-03, AES-04, AES-05, AES-06, AES-07, RSA-01, RSA-02, RSA-03, RSA-04, RSA-05, RSA-06, RSA-07, RSA-08, ED-01, ED-02, ED-03, ED-04, ED-05, ED-06, ED-07, ED-08, API-05, API-07, UI-06, UI-08, UI-12
**Success Criteria** (what must be TRUE):
  1. User can generate AES-128/192/256 keys via `crypto/rand`, see them as Base64 and hex, copy them, and save them as a symmetric key record.
  2. User can generate RSA-2048/3072/4096 key pairs (default 3072) and Ed25519 key pairs and save them as asymmetric key pair records (public key plaintext column; private key encrypted blob).
  3. User can export the public key in OpenSSH format (`ssh.MarshalAuthorizedKey`) or PEM format; round-trip parse test passes for both formats and both algorithms.
  4. User can export the private key in PKCS#8 PEM format (`x509.MarshalPKCS8PrivateKey`, PEM type `"PRIVATE KEY"`) ONLY after confirming the private-key exposure modal; same gate applies for "Copy private key."
  5. App builds and runs on macOS AND Windows at end of phase (cross-OS verification per PRD §2.3); exported keys load correctly in `~/.ssh/` and via `openssl pkcs8` on both OSes.
**Plans**: TBD
**UI hint**: yes

**Notes for plan-phase researcher:**
- Open research question to resolve at this phase's planning: verify `golang.org/x/crypto/ssh.MarshalPrivateKey` availability at the chosen `x/crypto` module version BEFORE planning ED-06 (Ed25519 OpenSSH private-key export). If unavailable, defer ED-06 to post-v1.0 and document in README. **Do not hand-roll the OpenSSH private-key format under any circumstances.**
- RSA-4096 keygen takes 1–5 seconds and MUST run in a goroutine off the Wails main thread (UI freeze otherwise). Surface progress in UI-08.
- Use `x509.MarshalPKCS8PrivateKey` for ALL private-key PEM export (RSA, Ed25519). PEM block type is always `"PRIVATE KEY"` — never `"RSA PRIVATE KEY"` (that is PKCS#1 and a common confusion source per Pitfall 17). Note: `ed25519.PrivateKey` is a value type, NOT a pointer.
- Public-key fingerprint: single helper `FingerprintSHA256` returning `SHA256:<base64-no-padding>` (Pitfall 19).
- UI-12 (private-key warning modal) has copy and export variants per PRD §9.8; both require explicit "I Understand…" confirmation. Keep modal text exactly per PRD.
- UI-06 (all 5 record-type forms) lands here once the asymmetric type schema is finalized; covers credential, api_key, secure_note, symmetric_key, asymmetric_key_pair.
- API-05 (`GenerateAESKey`, `GenerateRSAKeyPair`, `GenerateEd25519KeyPair`) and API-07 (`ExportPublicKey`, `ExportPrivateKey`) arrive together. Backend MUST verify `session.RequireUnlocked()` before any export call.

### Phase 5: v0.5 Cross-Platform Hardening
**Goal**: App is verified on both macOS and Windows end-to-end, with platform-specific lock-on-sleep, documented packaging/code-signing workflows, and a passing manual test matrix.
**Depends on**: Phase 4
**Requirements**: VAULT-13, XPLAT-01, XPLAT-02, XPLAT-03, XPLAT-04, XPLAT-05
**Success Criteria** (what must be TRUE):
  1. App builds and runs end-to-end on macOS (universal or arm64) and Windows (amd64); manual cross-platform test matrix per PRD §14.3 (22 scenarios) passes on both.
  2. Vault locks automatically when the system sleeps or the laptop lid closes (IOKit hook on macOS; `WM_WTSSESSION_CHANGE` / equivalent on Windows); verified manually on both OSes.
  3. README documents macOS and Windows setup, build, and packaging steps including the macOS Gatekeeper `xattr -d com.apple.quarantine` workaround for ad-hoc-signed builds and the Windows SmartScreen behavior for unsigned binaries.
  4. Vault file paths, export paths, clipboard, and file dialogs are verified on both OSes (path handling uses `path/filepath` only; `filepath.Clean(filepath.FromSlash(input))`; no Unicode normalization assumptions).
  5. Cross-OS verification at this phase boundary is the FULL pass; the smoke tests in earlier phases were rehearsals.
**Plans**: TBD

**Notes for plan-phase researcher:**
- Open research question to resolve at this phase's planning: System sleep detection requires platform-specific APIs (IOKit on macOS, `WM_WTSSESSION_CHANGE` on Windows). Research the exact Wails/OS hook approach at this phase's planning. If both prove infeasible in MVP timeframe, document and defer with a Settings warning.
- Code signing is DEFERRED for MVP. Document in README: macOS uses ad-hoc sign (`codesign --force --deep --sign - ./build/bin/abyss.app`); Windows ships unsigned with a SmartScreen note. SHA-256 checksums of release binaries belong in release notes (manual integrity check).
- The full PRD §14.3 22-scenario matrix runs again in Phase 6 as the final pass. This phase establishes that it CAN pass; Phase 6 ensures it DOES pass after polish.
- XPLAT-05 ("Cross-OS verification at milestone boundaries — do not defer to the end") is a policy that has been enforced at every phase's success criteria; the requirement-as-deliverable lives here as the formal pass.

### Phase 6: v1.0 MVP Complete
**Goal**: Final polish — Settings/About screens, frontend tests, full README threat-model documentation, password strength feedback — and a green final cross-platform manual test pass. No new features.
**Depends on**: Phase 5
**Requirements**: PASS-07, CLIP-07, UI-09, UI-10, TEST-02, DOC-01, DOC-02, DOC-03, DOC-04, DOC-05
**Success Criteria** (what must be TRUE):
  1. Settings screen lets the user change clipboard timeout, idle auto-lock timeout, theme preference, and last-selected record-type filter; About screen displays version and the security model summary.
  2. Master-password and generated-password screens show advisory strength feedback via `@zxcvbn-ts/core` (frontend-only; never crosses the Wails boundary); strength is informational, not a form gate.
  3. README covers project purpose, macOS+Windows dev/build/test setup, vault security model and threat model (PRD §4), required no-recovery and clipboard-limitations warnings, Go memory limitations statement, private-key export warning, known limitations and non-goals, learning-grade disclaimer.
  4. Required frontend tests (PRD §14.2) pass: Create/Unlock/Change-Master forms, record forms, password generator options, key generator options, clipboard countdown component, private-key warning modal, locked-state action blocking, auto-lock state transition.
  5. Final 22-scenario manual cross-platform test matrix (PRD §14.3) passes on macOS AND Windows; release-build smoke-test runs on a clean install of each OS.
**Plans**: TBD
**UI hint**: yes

**Notes for plan-phase researcher:**
- PASS-07 final placement: per FEATURES.md priority matrix this is P2 (v1.x candidate), but PRD §15 requires it in v1.0 if shipping the strength meter at all. Use `@zxcvbn-ts/core` v3 + a single language pack, frontend-only, advisory-only. Verify package currency/maintenance status when planning this phase; if abandoned, document and skip.
- Open question: surface clipboard / no-recovery warnings inside the app (Settings or About) in addition to README. PRD §19 puts them in README only. Recommendation: surface in About at minimum so users who never read the README still see them.
- This is the polish phase; resist adding any feature not already in v1 requirements. The roadmap deliberately excludes: cloud sync, browser extension, autofill, mobile, biometric/TPM, breach monitoring, TOTP, SSH agent, PGP, auto-update, plugins, Linux packaging, formal audit, telemetry, "remember last unlock," recent-vaults auto-unlock, CSV import/export, vault hint, lockout/rate-limiting (per PROJECT.md Out of Scope and PRD §3.3, §18).
- After v1.0 ships, the roadmap reorganizes into the milestone-grouped format per the GSD template; v1.x candidates (CSV import, encrypted-zip export, RSA OpenSSH private-key export, locked-state public-key ring, system-sleep lock if deferred from Phase 5, harmless-overwrite-then-clear clipboard) become explicit Active items in PROJECT.md only after this milestone closes.

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. v0.1 Dogfoodable Vault | 0/TBD | Not started | - |
| 2. v0.2 Security Foundation | 0/TBD | Not started | - |
| 3. v0.3 Expanded Record Types | 0/TBD | Not started | - |
| 4. v0.4 Key Generation and Export | 0/TBD | Not started | - |
| 5. v0.5 Cross-Platform Hardening | 0/TBD | Not started | - |
| 6. v1.0 MVP Complete | 0/TBD | Not started | - |

---

*Roadmap created: 2026-04-29*
*Source: derived from `docs/PRD.md` v1.2 §15 milestones and §20 build sequence; validated against research findings in `.planning/research/`.*
