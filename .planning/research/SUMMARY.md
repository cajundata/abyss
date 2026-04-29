# Research Summary — Abyss

**Project:** Abyss — Local-first desktop password & cryptographic key vault
**Domain:** Desktop secrets manager (Go + Wails v2, macOS + Windows)
**Researched:** 2026-04-29
**Confidence:** HIGH (PRD §2/§5/§6/§10/§18 are frozen; all four research files draw from official docs and verified sources)

---

## Executive Summary

Abyss is a local-first single-user vault that stores credentials, API keys, secure notes, and — its genuine differentiator — natively generated and exportable AES/RSA/Ed25519 keys. The correct comparator is KeePassXC with first-class crypto-key record types. Every meaningful stack and architectural decision has already been frozen in `docs/PRD.md` v1.2. Research validates that those decisions are correct and adds the implementation specifics (exact versions, package paths, pitfall mitigations) that planning agents will need.

The build sequence has a heavy front-loading of security-critical work. Fourteen of 23 identified pitfalls have their primary prevention window in v0.1 ("Dogfoodable Vault"). AAD enforcement, typed error infrastructure, sanitized logging, nonce uniqueness, and the ban on `math/rand` must all exist before any record is ever written. Attempting to ship v0.1 as a "skeleton" and bolt security on later is the single largest risk to the project — records written with wrong AAD cannot be repaired without re-encryption, and the grep test for SQLite plaintext leakage should run in CI on every PR from day one, not only at milestone boundaries.

The four research files are internally consistent. The stack is fully resolved with version pins. The architecture has precise package-boundary recommendations. The pitfall catalog maps every risk to its primary prevention milestone. The main open questions are operational (exact import path for the pure-Go golang-migrate SQLite driver, whether `golang.org/x/crypto/ssh.MarshalPrivateKey` is available at the chosen module version) and should be validated on the first day of v0.1 implementation, not deferred.

---

## Locked Stack — Do Not Re-Debate

These are frozen by PRD §2 and §18. Downstream planning agents must not relitigate them.

| Component | Locked Version | Why This Matters to Planning |
|-----------|---------------|------------------------------|
| Wails | **v2.12.0** | Includes macOS 26 (Tahoe) WebView crash fix and macOS clipboard mojibake fix (LANG env). Do not use v3 alpha. |
| Go | **1.25.9** (preferred) or **1.26.2** | Go 1.24 EOL'd 2026-02-11. Go 1.23.3+ required for macOS 15+. |
| TypeScript | **5.9.x** (5.9.2+) | TS 6 has deprecation warnings; TS 7 is beta (Go-native). Stay on 5.9 for MVP. |
| Vite | **8.0.x** (8.0.10) | Rolldown (Rust) bundler; 10–30x faster. Wails dev proxies to it. |
| React | **19.2.5** | Required by Mantine v9; compatible with Mantine v8. |
| Mantine | **v8.3.18 (pin `^8.3.18`)** | v9.0 shipped 2026-03-31 — only ~1 month old. v8.3.18 is "last 8.x," stable, has uncontrolled-form perf and built-in Zod Standard Schema resolver. Re-evaluate at v1.0. |
| SQLite driver | **`modernc.org/sqlite` v1.36.0+** | Pure-Go; eliminates CGO cross-compile pain for macOS->Windows. Avoid retracted versions (e.g., v1.34.3). |
| `golang-migrate` | **v4 (API frozen)** | Use `iofs` source driver with `embed.FS`. Use `database/sqlite` subpackage (NOT `database/sqlite3` which pulls CGO). |
| Zustand | **v5.x (5.0.x)** | Client-side store for vault session + clipboard state. Selectors prevent re-renders from 1-second clipboard countdown. |
| Zod | **v3.x (3.23+)** | Mantine v8's built-in `schemaResolver` works natively with Zod v3. Zod v4 requires a separate resolver package. |
| `react-router-dom` | **v6.x (6.30+)** | 10 PRD screens. v7 is fine too but v6 has smaller learning curve. |
| `golang.org/x/crypto/argon2` | **latest** (track Go release) | `argon2.IDKey(password, salt, time, memory, threads, keyLen)` — wrap in typed `Params` struct, never call directly. |
| `golang.org/x/crypto/ssh` | **latest** | `ssh.MarshalAuthorizedKey`, `ssh.FingerprintSHA256`, `ssh.NewPublicKey`. Public-key format for all exported keys. |
| `crypto/x509` | stdlib | `x509.MarshalPKCS8PrivateKey` — accepts `*rsa.PrivateKey`, `*ecdsa.PrivateKey`, `ed25519.PrivateKey` (NOT a pointer). PEM type = `"PRIVATE KEY"`, never `"RSA PRIVATE KEY"`. |
| Node.js | **22 LTS** | Current active LTS. |

**Clipboard implementation:** Wails runtime `ClipboardSetText`/`ClipboardGetText` (already in dep tree, PRD §2.7 lock, v2.12 has the macOS LANG fix). Do not use `atotto/clipboard` (spawns external processes) or `golang.design/x/clipboard` (adds CGO on Windows, image support not needed).

---

## Architectural Seams That Must Exist From v0.1

These are not negotiable and must not be deferred. Adding them after records are written is a breaking change or a security regression.

| Package / Seam | What It Does | Risk If Deferred |
|----------------|-------------|------------------|
| `internal/aad/` | Single `Build(kind, vaultID, recordID, recordType, dekID, wrappingType, envVersion)` helper for all three AAD shapes (record metadata, record secret, wrapped DEK). No inline AAD construction anywhere. | Records written with inconsistent or wrong AAD cannot be decrypted later. Retrofitting requires re-encrypting every record. |
| `internal/apperr/` | `AppError{Code, Message, Cause}` + `ToDTO()`. Single point of error sanitization. Typed `AppErrorCode` constants mirroring the TS enum. | Every Wails method must be refactored if this comes late; error contracts get hand-patched and drift. |
| `internal/session/` | Lock-state machine. `RequireUnlocked()` gating on every secret-touching method. Atomic session pointer swap on lock. DEK zeroing (`crypto.Zero(session.DEK)`) before nil assignment. | Without this, lock is advisory-only. Backend methods may serve secrets after lock. |
| `frontend/src/api/` | Typed wrappers around generated Wails bindings. All components route through this layer. `normalizeError(unknown) -> AppErrorDTO`. No component ever imports from `/wailsjs/go/main/App` directly. | Error-handling grows expecting raw `error.message` strings; painful to refactor when AppErrorCode is introduced. |
| `internal/logger/` | `slog` wrapper with redaction list for fields named `password`, `dek`, `kek`, `value`, `secret`, `private_key`, `api_key`. | Secrets logged during v0.1 dogfooding are unrecoverable from log files. |
| Migration fail-closed | On dirty `schema_migrations` state: return `MIGRATION_FAILED`, never auto-force. | Half-migrated vault schema leads to undefined behavior; auto-force risks data corruption. |
| `wails generate module` in CI | Generates TS bindings; CI fails if committed bindings differ. Catches DTO drift between Go structs and TS types. | Silent field rename in Go -> `undefined` in TS -> security-critical field silently missing (e.g., `isUnlocked`, `code`). |
| `errcheck` lint + ban `math/rand` | `errcheck` in golangci-lint catches dropped `crypto/rand.Read` errors. A `grep`/`forbidigo` rule bans `math/rand` import in all Go files. | Dropped `rand.Read` error -> zero-filled DEK or nonce. `math/rand` in a security path -> predictable key material. |

---

## AAD Must Ship in the First Encrypt Call

**PRD §15 lists "AAD enforcement" as a v0.2 exit criterion.** This is a milestone *verification* checkpoint, not permission to defer implementation. The research is unambiguous: AAD must be implemented in the very first `cipher.Seal` call written, before any record is saved to any vault. The reason:

- AAD is part of the ciphertext's authentication tag. A record encrypted without AAD, or with a different AAD shape, cannot be decrypted after the AAD format is added or changed.
- Retrofitting AAD after records exist requires re-encrypting every record — equivalent to a data migration that needs the DEK, which is wrong layering.
- The v0.1 "Looks Done But Isn't" checklist explicitly includes: "AAD-tampering test passes (swap secret blob between two records -> decrypt fails). Even though PRD lists this in v0.2 exit criteria, AAD must be present from v0.1."

**The AAD format (PRD §5.4):**
- Record metadata: `{vaultID}|{envelopeVersion}|{recordID}|{recordType}|{dekID}|metadata`
- Record secret: `{vaultID}|{envelopeVersion}|{recordID}|{recordType}|{dekID}|secret`
- Wrapped DEK: `{vaultID}|{dekID}|{wrappingType}|wrapped_dek`
- `schema_version` is explicitly excluded — schema migrations must not force record re-encryption.

---

## Key Findings by Research File

### Stack (from STACK.md)

The stack is fully resolved with high confidence. The two highest-impact unlocked decisions are:

1. **`modernc.org/sqlite` over `mattn/go-sqlite3`** — pure-Go eliminates CGO cross-compile pain. Wails issue #4112 tracks CGO cross-compile regressions in Wails 2.10+. At personal-vault scale (hundreds to a few thousand records), the 2x INSERT penalty is microseconds.

2. **Mantine v8.3.18 over v9** — v9 shipped 2026-03-31, ~6 weeks before MVP planning. v8.3.18 has all needed features (uncontrolled form mode, Zod resolver, `useIdle`, `useInterval`). Migrate at v1.0 or post-MVP.

**Migration driver gotcha (MEDIUM confidence — verify in v0.1):** The `golang-migrate/v4` package has two SQLite subpackages: `database/sqlite` (pure-Go, use this) and `database/sqlite3` (CGO, do not use). After wiring, run `go mod why github.com/mattn/go-sqlite3` — if it appears, the wrong driver was imported.

**Test isolation:** Per-test temp file via `t.TempDir()` for all storage/migration tests. Not `:memory:` — migration tests and `application_id` PRAGMA semantics differ between memory and file modes.

### Features (from FEATURES.md)

Every table-stakes feature is in PRD scope. Defaults are at or above industry norms (clipboard 30s, auto-lock 10min, password length 24, RSA 3072, Argon2id 256MiB).

**Should-have promotion candidates** — these are cheap and high-value; the roadmapper should treat them as v1.0 unless scope is severely constrained:

| REQ-ID | Feature | Why Promote |
|--------|---------|-------------|
| REC-13 | Encrypted tags in metadata blob | "Free" — tags live inside the already-encrypted metadata blob; no separate tag table needed |
| RSA-05 | RSA public-key PEM export | LOW effort; HIGH interop (TLS, JWT signing) |
| ED-06 | Ed25519 OpenSSH private-key export | PRIMARY SSH interop path; users drop this directly into `~/.ssh/` |
| VAULT-13 | Lock-on-system-sleep / lid-close | Closes real exposure window; platform-specific hooks (IOKit on macOS) |
| CLIP-04 | Manual clipboard clear button | LOW effort; aligns with security posture |

**Anti-features to explicitly exclude** (not named in PRD §3.3 but surfaced by research):

- **"Remember last unlock" / "stay unlocked across launches"** — subverts the lock model entirely; the DEK would survive a relaunch. Must be explicitly named as an anti-feature in PROJECT.md Out of Scope so future contributors do not add it as a "UX improvement."
- **Telemetry / analytics** — a vault must not phone home. Not in PRD §3.3 currently. Should be an explicit "we will never add this" exclusion.

### Architecture (from ARCHITECTURE.md)

**Module dependency order (build in this direction):**

```
apperr, logger -> crypto -> aad -> storage -> session -> records -> vault -> app.go -> main.go
                                                       /
                          generator -> keyexport -----+
                          clipboard -------------------+
```

Anything left of `app.go` has zero imports from anything to the right. `internal/` convention enforces external isolation.

**State residency rules:**
- DEK: in-memory only (`Session.dek []byte`); zeroed and nilled on lock; never logged, never persisted
- Records: decrypted on demand, never cached server-side (widens memory exposure, complicates lock semantics)
- Frontend: `sessionStore` for lock status; `clipboardStore` for countdown; `recordsById` cleared on lock via single `vault:locked` Wails event

**Lock-state machine:** `closed -> opening -> locked <-> unlocking -> unlocked -> locking -> locked`. Backend `session.RequireUnlocked()` is the authoritative check — every secret-touching method begins with it. Frontend `lockStatus` is advisory only.

**Idle tracking:** Frontend debounces user input pings to backend (>=1s). Backend owns the countdown timer and calls `Lock()`. Backend emits `vault:locked` event; frontend subscribes once at App root. Use a 1-second wall-clock poller (not `time.AfterFunc`) so the timer survives system sleep automatically.

### Pitfalls (from PITFALLS.md)

**23 pitfalls identified. 14 have primary prevention in v0.1.** The roadmapper must not treat v0.1 as "just the skeleton."

**Pitfall-to-milestone mapping:**

| # | Pitfall | Primary Prevention | Phase |
|---|---------|-------------------|-------|
| 1 | AES-GCM nonce reuse | `crypto.NewNonce()` always calls `crypto/rand.Read(12 bytes)`; nonce lives only for one `Seal` call | v0.1 Crypto/AEAD |
| 2 | AAD construction errors | `internal/aad/` single helper, golden-vector tests | v0.1 Crypto/AEAD |
| 3 | Argon2id param regression | `kdf.DefaultParams()` vs `kdf.ParamsForTesting()` in separate build-tag file | v0.1 KDF |
| 4 | Master password heap residency | Copy to `[]byte`, zero after KDF call; document Go memory limit in README | v0.1 Vault Lifecycle |
| 5 | DEK retention after lock | `crypto.Zero(dek)` before nil; atomic session swap; context cancel | v0.1 Vault Lifecycle |
| 6 | Frontend caches decrypted records past lock | Single `vault:locked` event -> clear all stores -> navigate to unlock screen | v0.1 Records |
| 7 | SQLite plaintext leakage | Automated grep test (sentinel strings); schema review at every migration | v0.1 Records |
| 8 | golang-migrate dirty state | Explicit dirty check -> `MIGRATION_FAILED`; refuse to unlock; document recovery | v0.1 Storage |
| 9 | AAD coupled to schema_version | `aad.Build()` signature has no `schemaVersion` param; code comment "DO NOT add schema_version" | v0.1 Crypto/AEAD |
| 10 | crypto/rand error swallowing | `errcheck` lint in CI; `RandomBytes(n int) ([]byte, error)` wrapper | v0.1 (lint at skeleton) |
| 11 | math/rand in security path | `grep`/`forbidigo` lint ban; CI gate | v0.1 (lint at skeleton) |
| 12 | Unlock oracle via differentiated errors | Single `mapUnlockError` collapses all internal errors to `UNLOCK_FAILED` | v0.1 Vault Lifecycle |
| 13 | Wails TS type drift | `wails generate module` in CI; Go DTO structs have `json:` tags; TS `strict: true` | v0.1 Skeleton |
| 14 | Cross-platform path handling | `path/filepath` only; `filepath.Clean(filepath.FromSlash(input))`; `t.TempDir()` in tests | v0.1 Storage |
| 15 | macOS code-signing friction | README: `xattr -d com.apple.quarantine` workaround and SmartScreen bypass | v0.5/v1.0 Docs |
| 16 | Idle timer skewed by sleep | Wall-clock poller (1s tick comparing `time.Since(lastActivityAt)`), not single `time.AfterFunc` | v0.2 Auto-Lock |
| 17 | PKCS#8 vs PKCS#1 confusion | Always `x509.MarshalPKCS8PrivateKey`; PEM type `"PRIVATE KEY"`; round-trip test | v0.4 Key Export |
| 18 | OpenSSH private key format (Ed25519) | Verify `ssh.MarshalPrivateKey` availability at chosen module version; defer if unavailable | v0.4 Key Export |
| 19 | Public key fingerprint format | Single helper `FingerprintSHA256`; output `SHA256:<base64-no-padding>` only | v0.4 Key Gen |
| 20 | Clipboard clear silently fails | Treat as best-effort; UI says "attempted"; README documents clipboard manager limitation | v0.1 Clipboard |
| 21 | JSON metadata blob schema drift | Append-only fields; all new fields optional (`omitempty`); type changes -> new field name | v0.1 Records |
| 22 | Recent-vaults UX subverts lock | No recent-vaults in v0.1–v1.0; welcome screen = "Create" and "Open" only | v0.1 Vault Lifecycle |
| 23 | zxcvbn library selection | `@zxcvbn-ts/core` v3 + language pack, frontend only; advisory only; not a form gate | v0.1 Password Gen |

---

## The Grep Test Must Run in CI, Not Only at Milestones

The grep test (Pitfall 7) should not wait for milestone boundaries. The acceptance criterion: create a fixture vault with sentinel strings in every record field that should be encrypted, close the DB, scan the raw `.sqlite` file and any `.sqlite-journal` temp files for those sentinels. Any hit = CI failure. This should be an automated test in `internal/storage` from v0.1 onward, running on every PR.

---

## Cross-OS Verification at Every Milestone Boundary

PRD §2.3 requires macOS + Windows. Deferring Windows smoke-tests to v0.5 is a known technical-debt shortcut that costs more than it saves: path-handling bugs (`path` vs `path/filepath`, NFD vs NFC Unicode), Wails-binding regressions, and CGO-related issues discovered late force rewrites of earlier code. At minimum, do one Windows build smoke-test at the end of every milestone, not only at v0.5 Cross-Platform Hardening.

---

## Implications for Roadmap

### Suggested Phase Structure

The PRD §15 milestones and §20 build steps are the authoritative sequence. Research validates and refines this structure:

**v0.1 — Dogfoodable Vault (heaviest security load: 14 of 23 pitfalls)**

REQ coverage: VAULT-01–12, REC-01 (credential only), PASS-01–06, CLIP-01–07, API-01–09, SEC-01–10, STORE-01–08, TEST-01 (crypto/KDF/nonce/AAD/migration subset), UI-01–07 (Welcome, Create, Unlock, Vault Home, Record Detail/Edit, Password Generator)

Must exist by end of v0.1:
- `internal/aad/` with golden-vector tests
- `internal/apperr/` + `frontend/src/errors/`
- `internal/session/` with `RequireUnlocked()` and DEK zeroing on lock
- `internal/logger/` with redaction list
- `frontend/src/api/` wrapper layer
- Migration fail-closed on dirty state
- `wails generate module` in CI
- `errcheck` + `math/rand` ban in CI
- Automated grep test in storage test suite
- One Windows build smoke-test

**v0.2 — Security Foundation**

REQ coverage: VAULT-08 (auto-lock), VAULT-09 (change master password), SEC-* hardening, TEST-02 (frontend tests), full typed-error enum, AAD-tampering tests as exit-criterion verification (implementation was in v0.1)

Pitfalls primarily addressed: 16 (idle timer sleep skew), 12 (unlock oracle hardening), 8 (migration dirty state verification)

**v0.3 — Expanded Records**

REQ coverage: REC-02 (API key), REC-03 (secure note), REC-13 (encrypted tags)

Key concern: JSON blob backward compatibility (Pitfall 21). All new fields must be optional.

**v0.4 — Key Generation and Export**

REQ coverage: AES-01–07, RSA-01–08, ED-01–08, UI-08 (Key Generator screen)

Pitfalls primarily addressed: 17 (PKCS#8 vs PKCS#1), 18 (OpenSSH private-key format), 19 (fingerprint format)

RSA 4096 keygen must run in a goroutine — it can take 1–5 seconds and will freeze the UI if called on the main thread.

**v0.5 — Cross-Platform Hardening**

REQ coverage: XPLAT-01–05, DOC-01–05, VAULT-13 (lock-on-system-sleep, Should-have)

Pitfalls primarily addressed: 15 (macOS code-signing friction), 14 (cross-platform path handling full audit)

**v1.0 — MVP Complete**

Final manual cross-platform test matrix (PRD §14.3, 22 scenarios). README threat model. Release build smoke-test on clean macOS + Windows. No new features.

### Phase Ordering Rationale

1. Security infrastructure before any feature: `internal/aad/`, `internal/apperr/`, `internal/session/`, `internal/logger/`, and the CI lint rules must exist before the first record is ever saved. AAD is not retrofittable.
2. Vault lifecycle before records: the DEK/KEK envelope, unlock flow, and lock-state machine are the foundation everything else sits on.
3. Credential records before expanded record types: validates the encryption pipeline end-to-end before adding record-type complexity.
4. Key generation together (v0.4): AES/RSA/Ed25519 generators share `internal/generator/` and `internal/keyexport/`. Doing them in the same phase avoids partial key-export infrastructure.
5. Cross-platform hardening before final test pass: path issues and OS-specific behaviors must be resolved before the 22-scenario manual test matrix is locked in.

### Research Flags

Phases needing deeper research during planning:

- **v0.1 — golang-migrate driver import path:** Verify the exact subpackage name and DSN format for the pure-Go SQLite driver in golang-migrate/v4. Run `go mod why github.com/mattn/go-sqlite3` to confirm CGO has not crept in. (MEDIUM confidence on exact path.)
- **v0.4 — OpenSSH private-key marshal:** Verify `golang.org/x/crypto/ssh.MarshalPrivateKey` availability at the chosen module version before planning ED-06. If unavailable, defer ED-06 to post-v1.0 and document in README.
- **v0.2 — Lock-on-sleep OS hooks:** System sleep detection (VAULT-13) requires platform-specific APIs (IOKit on macOS, WM_WTSSESSION_CHANGE on Windows). Research the exact Wails/OS hook approach at v0.2 phase planning if targeting v0.5.

Phases with standard, well-documented patterns (skip or shorten research):

- **v0.1 crypto primitives:** Argon2id, AES-256-GCM, nonce generation, DEK/KEK envelope — PRD-locked to Go stdlib + `golang.org/x/crypto`. No new research needed.
- **v0.3 record types:** Record structure fully specified in PRD §7. Adding API key and secure note types is form + encryption pattern, no new research.
- **v0.4 PKCS#8 and OpenSSH public-key marshaling:** Fully documented in STACK.md §5 with verified code patterns.

---

## Open Questions to Resolve at Phase-Research Time

These are aggregated from all four research files. They are not scope-creep recommendations — they are gaps in the explicit decision record that downstream phase planners need to resolve.

**Resolve at v0.1 planning:**
1. Confirm the exact import path for the pure-Go SQLite driver in golang-migrate/v4 (`database/sqlite` vs another subpackage). Verify DSN params that set `journal_mode=DELETE` and `foreign_keys=ON`.
2. Decide: should `@mantine/core` stay at `^8.3.18` or upgrade to v9? If v9, the phase-1 plan must account for the migration guide. If v8, note the re-evaluation point.
3. Decide: should password strength (PASS-07 / FR-033, `@zxcvbn-ts/core` frontend-only) ship in v0.1 or defer to v0.5? PRD marks it Should-have.

**Resolve at v0.2 planning:**
4. Decide: should lock-on-system-sleep (VAULT-13) target v0.5 or v1.0? Currently flagged as Should-have for MVP hardening, required by v1.0 in REQUIREMENTS.md.
5. Decide: should clipboard / no-recovery warnings be surfaced inside the app (Settings or About screen) and not only in README? PRD §19 puts them in README only.

**Resolve at v0.4 planning:**
6. Verify `golang.org/x/crypto/ssh.MarshalPrivateKey` availability for ED-06 (Ed25519 OpenSSH private-key export). If unavailable, defer to post-v1.0 and document in README.

**Resolve as explicit policy before roadmap is finalized:**
7. Add "remember last unlock / stay unlocked across launches" to PROJECT.md Out of Scope explicitly. Currently implicit only; a future contributor may propose it as a UX improvement.
8. Add "telemetry / analytics" as an explicit "we will never add this" exclusion in PROJECT.md. Currently a silent omission in PRD §3.3.
9. Decide whether CSV import goes on the post-MVP roadmap as a v1.x candidate. Without it, migrating existing credentials requires retyping everything.
10. Document in README that "copy the SQLite file" is the intended backup path, given the "no recovery" stance.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack (version pins) | HIGH | Wails v2.12.0, Go 1.25/1.26, Mantine v8.3.18, Zustand v5 — confirmed via official release pages and changelogs as of 2026-04-29. |
| Features (scope and defaults) | HIGH | PRD scope is fixed; defaults validated against KeePassXC, Bitwarden, 1Password, OWASP, NIST SP 800-57. |
| Architecture (package boundaries, state, lock-state machine) | HIGH | Derived from PRD §2/§5/§6/§10; Wails idioms verified against official docs and maintainer posts. |
| Pitfalls (crypto/storage) | HIGH | Go stdlib + AEAD literature is well-settled; pitfalls verified against real issue trackers (Wails #4112, #2534). |
| Pitfalls (Wails-specific cross-platform) | MEDIUM | Framework-version-sensitive; some behaviors change with Wails releases. Verify on each build. |
| golang-migrate driver import path | MEDIUM | Sparse documentation on the pure-Go subpackage. Verify in v0.1. |
| Ed25519 OpenSSH private-key marshal API | MEDIUM | `ssh.MarshalPrivateKey` exists in newer `x/crypto`; verify at chosen module version in v0.4. |

**Overall confidence:** HIGH for the decisions that matter most. The two MEDIUM items are both operational verifications for specific Go import paths, not architectural questions.

### Gaps to Address

- **golang-migrate pure-Go driver DSN format:** Verify `sqlite://file:...` vs `sqlite:///abs/path` vs other DSN forms at v0.1 implementation. If mattn appears in `go.sum`, the wrong driver was imported.
- **Ed25519 OpenSSH private-key export:** Verify `golang.org/x/crypto/ssh.MarshalPrivateKey` is available at the project's pinned `x/crypto` version before planning ED-06. Do not hand-roll the OpenSSH private-key format under any circumstances.
- **Mantine v8 vs v9 final call:** Decide at v0.1 phase research, not mid-implementation. Either is defensible; switch must happen at a milestone boundary.
- **Zxcvbn integration timing:** Verify `@zxcvbn-ts/core` v3 currency (last release date, npm audit status) at v0.1 phase planning. If stale or abandoned, defer to post-MVP.

---

## Sources

Aggregated from all four research files. Full citations in STACK.md §14, FEATURES.md Sources, ARCHITECTURE.md Sources, PITFALLS.md Integration Gotchas.

### Primary (HIGH confidence)
- Wails v2.12.0 release page and changelog — version confirmed, macOS fixes verified
- `golang.org/x/crypto/argon2` godoc — `IDKey` signature, parameter semantics
- `golang.org/x/crypto/ssh` godoc — `MarshalAuthorizedKey`, `NewPublicKey`, `FingerprintSHA256`, `ParseRawPrivateKey`
- `crypto/x509` source — `MarshalPKCS8PrivateKey` accepted types
- Mantine changelog — v8.3.18 (2026-03-17), v9.0 (2026-03-31), v9.1 (2026-04-21)
- Go release history — Go 1.24 EOL confirmed 2026-02-11
- OWASP Password Storage Cheat Sheet — Argon2id minimum parameters
- NIST SP 800-57 Part 3 Rev 1 — RSA 3072 as preferred key size beyond 2030
- PRD §2/§4/§5/§6/§10/§13/§14/§15/§18/§20 — authoritative source for all locked decisions

### Secondary (MEDIUM confidence)
- Wails Issue #4112 — CGO cross-compile regression in Wails 2.10+
- Wails Issue #2534 — `ClipboardSetText` macOS silent failure (fixed in v2.12)
- DataStation SQLite Go benchmark + HN thread — modernc vs mattn performance comparison
- Product comparisons (KeePassXC, Bitwarden, 1Password) for default behavioral validation

### Tertiary (operational — verify at implementation)
- golang-migrate `database/sqlite` pure-Go subpackage DSN format — sparse docs, verify in v0.1
- `golang.org/x/crypto/ssh.MarshalPrivateKey` availability — verify at chosen module version in v0.4

---
*Research completed: 2026-04-29*
*Ready for roadmap: yes*
