# Pitfalls Research

**Domain:** Local-first desktop password & key vault (Go + Wails v2 + SQLite + DEK/KEK envelope crypto)
**Researched:** 2026-04-29
**Confidence:** HIGH for crypto/storage pitfalls (Go stdlib + AEAD literature is well-settled), MEDIUM for Wails-specific and cross-platform behaviors (framework-version-sensitive)

---

## Phase Vocabulary

These names appear in the "Phase to address" fields and map directly to PRD §15 milestones and §20 build steps.

| Phase Name (used here)      | PRD §20 step(s) | PRD §15 milestone |
|-----------------------------|-----------------|-------------------|
| Project Skeleton            | 1               | v0.1              |
| Storage / Migration         | 2, 3            | v0.1              |
| KDF                         | 4               | v0.1              |
| Crypto / AEAD               | 5, 8            | v0.1 (AAD enforced v0.2) |
| Vault Lifecycle             | 6               | v0.1              |
| Records / Encryption        | 7, 8            | v0.1              |
| Password Generator          | 9               | v0.1              |
| Clipboard                   | 10              | v0.1              |
| Auto-Lock                   | 11              | v0.2              |
| Master Password Change      | 12              | v0.2              |
| Expanded Records            | 13              | v0.3              |
| Key Generation              | 14, 15, 16      | v0.4              |
| Key Export                  | 17              | v0.4              |
| Cross-Platform Hardening    | 18              | v0.5              |
| Documentation / Threat Model| 19              | v0.5–v1.0         |
| Final Test Pass             | 20              | v1.0              |

---

## Critical Pitfalls

These are catastrophic — they silently break confidentiality, integrity, or correctness, and the failure is usually invisible until it is too late.

### Pitfall 1: AES-GCM Nonce Reuse Under the Same Key

**What goes wrong:**
Encrypting two different plaintexts with AES-GCM using the same `(key, nonce)` pair catastrophically breaks confidentiality (the XOR of plaintexts leaks) and authentication (the GHASH authentication key is recoverable). For Abyss this means: if the DEK encrypts two records with the same nonce, both records' plaintexts can be recovered, and an attacker can forge new records under the DEK without knowing it.

**Why it happens:**
- Developer reuses a single nonce variable across multiple encrypt calls in a loop.
- Counter-style nonce generator that wraps after `2^32` writes (or after a process restart resets the counter to zero).
- Test code seeds a deterministic RNG and that helper accidentally gets used in production.
- Re-encrypting the same record on edit and reusing the prior nonce instead of generating a new one.
- Random nonce generated at higher scope than the encrypt call (e.g. once per session, then reused).

**How to avoid:**
- Single helper `crypto.NewNonce() ([12]byte, error)` that calls `crypto/rand.Read` for exactly 12 bytes every call. No counter mode. No callers may construct a nonce themselves.
- Helper returns an error on `rand.Read` failure; never returns a partially-filled slice.
- On every record write (create, update, master-password-change re-wrap), generate a brand new nonce — never carry one forward.
- Document the random-nonce birthday bound: with 96-bit random nonces under one DEK, collision probability is `~2^-32` after `2^32` encryptions; fine for a personal vault but document the assumption.
- Forbid `cipher.NewGCMWithNonceSize` with anything other than 12 unless explicitly approved.

**Warning signs:**
- Any code path that captures a nonce in a struct field rather than generating it inline at encrypt time.
- A `NonceFor(recordID)` style helper — deterministic nonces are a smell.
- Unit tests that pass a fixed nonce and reuse it across multiple `Seal` calls.
- A nonce field whose type is `int` or `[]byte` rather than fixed-size `[12]byte`.

**Phase to address:** Crypto / AEAD (early — before any record is ever written).

**Acceptance check (for phase plan):**
- Unit test: call `NewNonce()` 1,000,000 times and assert no duplicates and no all-zero output.
- Unit test: encrypt the same plaintext twice; assert the ciphertexts differ.
- Code review checklist: no `nonce` variable lives longer than one `Seal`/`Open` call.

---

### Pitfall 2: AAD Construction Errors

**What goes wrong:**
The AAD (Additional Authenticated Data) is the only thing binding a ciphertext to its identity (record ID, type, dek_id, blob role). If AAD is constructed inconsistently between encrypt and decrypt, decryption fails for legitimate records (correctness bug). If AAD is too lax (missing fields, wrong order, wrong separators), an attacker can swap `encrypted_secret_blob` between two records of the same `record_id` and the swap will decrypt successfully — defeating the tamper-evidence guarantee in PRD §4.2 and §5.4.

**Why it happens:**
- Two code paths build AAD independently (one in encrypt, one in decrypt) and drift.
- Builder uses string concatenation with no separator: `"vault123record456metadata"` collides with `"vault12record3456metadata"`.
- Off-by-one in field order: `record_type || record_id` instead of `record_id || record_type`.
- `schema_version` accidentally included (PRD §5.4 explicitly forbids), forcing re-encryption on every migration.
- `record_envelope_version` represented as `int` in encrypt and `string` in decrypt.
- `dek_id` passed as the active DEK at decrypt time rather than the DEK the record was originally encrypted with.

**How to avoid:**
- One single helper per AAD role:
  ```go
  func MetadataAAD(vaultID string, envelopeVersion uint32, recordID string, recordType string, dekID string) []byte
  func SecretAAD(vaultID string, envelopeVersion uint32, recordID string, recordType string, dekID string) []byte
  func WrappedDEKAAD(vaultID string, dekID string, wrappingType string) []byte
  ```
- Use a fixed delimiter byte that cannot appear in any field (e.g. `0x00`) and document it. Encode `envelopeVersion` as 4 bytes big-endian, not its decimal string.
- Both encrypt and decrypt code paths must call the same helper. No inline AAD construction.
- Golden-vector tests: hard-coded inputs → hard-coded expected AAD bytes (hex). Catches silent reordering during refactors.
- Tamper tests: encrypt record A's metadata with record A's AAD; attempt to decrypt with record B's AAD; assert it fails.
- Read at decrypt time: `dek_id` from the row being decrypted, not from the active session.

**Warning signs:**
- AAD built from a `fmt.Sprintf` call (separator and ordering are implicit and fragile).
- Encrypt and decrypt code paths in different files with no shared helper.
- A test that mutates the ciphertext bytes to verify GCM rejects it but no test that mutates the AAD inputs.
- `schema_version` appears anywhere near the AAD builder.

**Phase to address:** Crypto / AEAD phase (initial design — AAD shape is hard to change later because it would force re-encryption of all records). PRD's AAD-tampering test is in v0.2 milestone exit criteria.

**Acceptance check:**
- Tamper test: swap `encrypted_secret_blob` between two records → decrypt fails.
- Tamper test: change `record_type` in the row but not in the ciphertext → decrypt fails.
- Tamper test: change `dek_id` in the row → decrypt fails.
- Golden-vector test: AAD bytes for known inputs are stable across builds.
- Code search: `grep -r "schema_version" internal/crypto` returns zero hits.

---

### Pitfall 3: Argon2id Parameter Regression Through Test Harness

**What goes wrong:**
Production ships with `memory=19 MiB, iterations=2` (or worse, `time=1, memory=4 KiB`) because someone lowered Argon2id parameters in tests for speed and the test path was wired into production. The vault is now brute-forceable in seconds rather than the intended hundreds-of-milliseconds-per-attempt cost.

**Why it happens:**
- One `kdf.Derive(password, salt)` function reads parameters from a global var, and the test setup mutates that global.
- Default parameters are decided in `init()` and tests override the var.
- Parameters live in a config file and the developer commits the test config.
- Helper `kdf.Fast()` exists for tests and gets accidentally imported by production code paths.
- Parameters are stored only at the wrapping level, but the production default constant gets edited for "CI speed."

**How to avoid:**
- Two distinct, explicitly named functions: `kdf.DefaultParams() Params` (production-grade, returns the PRD §5.3 target / fallback / minimum profiles) and `kdf.ParamsForTesting() Params` (only callable from `_test.go` files).
- Use Go's build tag or package-internal visibility to prevent `ParamsForTesting` from being callable from non-test code (e.g. put it in a file named `params_testing.go` in a separate `kdftest` package, or guard with `//go:build testing`).
- KDF parameters are stored per `key_wrappings` row (PRD §5.3) — verify them on every unwrap; if a wrapping row claims `memory=4 KiB` the app should refuse to use it (treat as corrupt or downgrade attack).
- A "production profile" guardrail: at vault create time, if measured Argon2 derive time is under, say, 100 ms, log a warning and bump parameters.
- A separate benchmark test that asserts `DefaultParams` derive time is within an expected range on the dev machine; this catches accidental downgrades.

**Warning signs:**
- A single `var kdfParams = ...` package-level variable.
- Test files calling `kdf.SetParams(...)` to globally mutate the production path.
- `kdf_params_json` rows in a real vault containing very low values.
- `os.Getenv("FAST_KDF")` branches in non-test code.

**Phase to address:** KDF phase. Verify again at v0.2 milestone (Security Foundation).

**Acceptance check:**
- Unit test: in production mode, derive time on the dev machine is between 200 ms and 2 s (dev-machine specific; calibrate at first run).
- Unit test: `key_wrappings.kdf_params_json` parsed at unwrap time has `memory >= 19*1024` and `iterations >= 2` and `key_length == 32` (the absolute floor from PRD §5.3) — refuse to unwrap otherwise.
- `grep` for any production-path import of testing parameters returns zero hits.

---

### Pitfall 4: Master Password Heap Residency via Wails String Boundary

**What goes wrong:**
Go strings are immutable backed by an unaddressable byte array; you cannot reliably zero them. Wails passes `masterPassword` from the JS side as a Go `string` (per the DTOs in PRD §10.2). The string therefore lives in Go heap memory until garbage-collected, at which point the bytes may still sit in freed-but-not-overwritten memory. If malware or a memory dump catches that window, the master password is recoverable even after "lock."

This is partially acknowledged in PRD §4.4, but the pitfall is treating "I read the PRD" as having mitigated it. The Wails-generated bindings and JSON unmarshaling create *additional* string copies beyond the one in the DTO struct.

**Why it happens:**
- DTO field is typed `string` (it has to be — JSON can't carry `[]byte` reliably without base64).
- `json.Unmarshal` allocates a fresh string from the bytes buffer.
- Wails' internal binding code may make further copies.
- Developer holds the `string` past the KDF call "just in case" we need to retry.

**How to avoid:**
- At the very top of `UnlockVault` / `CreateVault` / `ChangeMasterPassword` (the only three entry points that touch master password), copy the string to a `[]byte` and immediately overwrite the local string variable with `""` (does not zero the original backing array, but stops it from being a live pointer).
- Pass `[]byte` everywhere downstream — KDF, validation, etc.
- After the `[]byte` has been fed to Argon2id, `crypto.Zero(buf)` (write zeros) and discard the slice.
- Never store the master password in a struct field. Never log the master password. Never stash it for later retry.
- README must state clearly (per PRD §4.4) that the master password may briefly persist in heap memory due to Go + Wails immutable-string boundaries; this is a known and documented residual risk.
- Code review checklist for the three entry points: zero allowed copies past the KDF call.

**Warning signs:**
- A field like `app.lastMasterPassword string` or any cache.
- The string `masterPassword` referenced more than two times in the unlock path.
- A retry/back-off mechanism that reuses the password.
- Any debug print of the request struct (logs the password by accident).

**Phase to address:** Vault Lifecycle phase. Re-audited at v0.2 (Security Foundation) and at Documentation phase (README).

**Acceptance check:**
- Code review: every `request.MasterPassword` reference is followed within ≤3 lines by `[]byte` conversion + KDF call + zero.
- Unit test: a sanitized-logging test that runs unlock with a sentinel password ("zzz-canary-zzz") and asserts the string never appears in any log line.
- README contains the Go memory limitations statement verbatim from PRD §4.4.

---

### Pitfall 5: DEK Retention After Lock

**What goes wrong:**
"Lock" returns the UI to the locked state but the DEK is still in memory because the in-memory session struct was not actually cleared, only marked as locked. A subsequent code path (a stale background task, a re-entrant Wails call, an event handler) successfully decrypts records even though the user thinks the vault is locked. Worse: the DEK bytes sit in a Go slice that gets reused without overwriting, so the bytes leak via slice aliasing.

**Why it happens:**
- "Lock" sets `session.IsLocked = true` but leaves `session.DEK []byte` in place.
- "Lock" reassigns `session = nil` but a goroutine retains a closure-captured reference.
- Lock zeroes the slice header but not the backing array — `dek = nil` does not overwrite memory.
- Lock relies on garbage collection; GC is non-deterministic and the bytes survive arbitrarily long.

**How to avoid:**
- Lock procedure (in this order):
  1. `crypto.Zero(session.DEK)` — overwrite the byte slice with zeros while it still has a valid pointer.
  2. Replace `session.DEK = nil`.
  3. Replace the entire session pointer with a new locked session: `vault.session.Store(&Session{Locked: true})`.
  4. Cancel any in-flight contexts (`session.cancel()`), which kills goroutines that captured the old DEK.
- All record-decryption call sites must read the DEK from the current session (atomic load) at the start of the call. If `Locked == true`, refuse and return `VAULT_LOCKED`.
- No code path may pass the DEK as a function argument out to a goroutine that outlives the request.
- Use `sync/atomic.Pointer[Session]` so the lock swap is observed by all goroutines without races.

**Warning signs:**
- Lock function body shorter than 5 lines.
- Lock function does not call any `Zero()` helper.
- A `var dek []byte` package-level variable.
- Decrypt functions that take `dek []byte` as a parameter rather than reading from current session.

**Phase to address:** Vault Lifecycle phase (initial implementation), tightened at Auto-Lock phase (v0.2).

**Acceptance check:**
- Test: unlock vault, decrypt a record (succeeds), call lock, attempt to call the *same* internal decrypt function with the *same* record — assert it returns `VAULT_LOCKED`. Re-do with raw byte access; assert DEK slice is zeroed.
- Test: decrypt under one DEK, lock, unlock with the same password, decrypt again — assert ciphertexts decrypt under the freshly-unwrapped DEK (proving lock didn't leave a stale reference around).
- Stress test: 10 concurrent decrypts in flight when lock is called — none should succeed after the lock instant.

---

### Pitfall 6: Frontend Caches Decrypted Records Past Lock

**What goes wrong:**
Backend correctly clears DEK and decrypted state on lock, but the React frontend has been holding decrypted record contents in component state, in a Zustand/Redux store, in `useState`, in the React Query cache, in a `localStorage` "draft" save, or in a closed modal that hasn't unmounted. After lock, the user navigates back and sees their decrypted password sitting on screen, or copies it via dev tools.

**Why it happens:**
- React state outlives the screen that displayed it (modal closed, but parent component still mounts the data).
- React Query caches `getRecord(id)` responses by default for 5 minutes.
- A "recently viewed" feature stores last-viewed record contents in memory.
- A form-state library auto-saves drafts to `localStorage` (which persists to disk!).
- A toast notification component displays "Copied: \*\*\*\*\*\*" but logs the unmasked value to a console hook.

**How to avoid:**
- A single `LockEvent` emitted from backend on lock (Wails event). Frontend has one global listener that:
  - Clears every Zustand/Redux store slice that contains record data.
  - Calls `queryClient.clear()` (React Query) or equivalent.
  - Navigates to the unlock screen, unmounting all record-detail / record-edit screens.
  - Clears any in-memory `useState` by virtue of unmount.
- Forbid `localStorage` / `sessionStorage` / `IndexedDB` writes of any decrypted record content. Lint rule or grep guard at CI.
- Form auto-save: only allowed for non-secret metadata, and the draft store must be cleared on lock.
- The `RecordDetail` screen must use a "controlled by lock state" wrapper that returns null when `isUnlocked === false`.

**Warning signs:**
- Any frontend state library configured with `persist` middleware on a slice that holds decrypted data.
- React Query default `staleTime` / `gcTime` left unmodified for record queries.
- A "recently viewed" or "favorites" list that stores anything beyond record IDs.
- Browser dev tools `Application → Local Storage` shows record-shaped data after lock.

**Phase to address:** Vault Lifecycle phase (initial lock UX wiring); deepened at Auto-Lock phase (v0.2) and at Records / Encryption phase.

**Acceptance check:**
- Frontend test: render `<RecordDetail>` with mock record, fire `LockEvent`, assert the rendered DOM contains no decrypted strings (use `screen.queryByText`).
- Frontend test: open record A, navigate to record B, fire lock, navigate back to record A — assert UI shows unlock screen, not stale data.
- Manual: open dev tools → Application tab; verify after lock that `localStorage`, `sessionStorage`, `IndexedDB` contain no record content.

---

### Pitfall 7: SQLite Plaintext Leakage Through Forgotten Field or Debug Path

**What goes wrong:**
A field that should be encrypted (notes, tags, URL, username, title, API provider name) ends up in a plaintext column or an index. Could happen because (a) a developer added a column for "search performance" and stored the title plaintext, (b) tags are stored in a plaintext join table, (c) a `LEFT JOIN` exposes plaintext debug columns added during testing, (d) an `application_id` is right but a debug `PRAGMA` left WAL files behind with plaintext, (e) `.sqlite-journal` rollback files contain plaintext written before encryption was added.

**Why it happens:**
- "We need search to be fast, let's denormalize the title" — incremental departure from encrypt-by-default.
- Migration adds a column without considering encryption boundary.
- Debug logging dumps a full `SELECT *` to a log file.
- Encrypted-tags requirement is `Should-have` (PRD FR-026) so first cut stores them plaintext "we'll fix it later" — and forgets.
- Search by tag does `LIKE '%dev%'` against a plaintext tag column.
- A `PRAGMA database_list` or schema export includes plaintext content in error messages.

**How to avoid:**
- Encrypt-by-default mental model: the only allowed plaintext columns per PRD §6.3 are:
  - `vault_metadata.id` (literal `'vault'`), `vault_metadata.vault_id`, `vault_metadata.schema_version`, `vault_metadata.record_envelope_version`, timestamps.
  - `deks.*` (no plaintext keys, just identity).
  - `key_wrappings.*` (params, salt, nonce, wrapped DEK — all non-secret or already-encrypted).
  - `records.id`, `records.vault_id`, `records.dek_id`, `records.record_type`, timestamps.
  - `records.public_key_openssh`, `records.public_key_pem`, `records.public_key_fingerprint` (public material — explicitly allowed).
  - `settings.key`, `settings.value` (must be non-sensitive UI prefs only — PRD §6.3).
- An end-to-end "grep test": create a fixture vault, populate records with sentinel strings (`"CANARY-TITLE-XYZ"`, `"CANARY-USERNAME-ABC"`, etc.), close the DB, then `grep` the `.sqlite` file *and* `.sqlite-journal` *and* any temp file for those sentinels. Test fails if any are found.
- Schema review at every migration: each new column gets a stamped justification — "encrypted blob," "non-sensitive," or "public key material." No fourth category.
- Code review checklist: no `SELECT *` in production code paths that log; explicit column lists only.
- No SQLite full-text search (`FTS5`) over encrypted columns — search must be in-memory after decryption (PRD §FR-022).

**Warning signs:**
- A column of type `TEXT` whose name implies user content (`title`, `notes`, `username`, `url`).
- A `tags` table with a `name TEXT` column.
- Logs containing record IDs alongside what looks like decrypted content.
- Any reference to FTS5, virtual tables, or `MATCH` operator in record search code.
- `schema_migrations.dirty = 1` rows that refer to a migration that mentions plaintext.

**Phase to address:** Records / Encryption phase. Re-verified at every milestone via grep test (v0.1, v0.2, v0.3, v0.4, v1.0). PRD §17 #14 makes this a release gate.

**Acceptance check:**
- Automated test in `internal/storage` test suite: `TestNoPlaintextLeakage` — populates 1 of each record type with canary strings, closes DB, scans file bytes for canaries, fails on any match.
- Manual cross-platform test matrix per PRD §14.3 final row.

---

### Pitfall 8: `golang-migrate` Dirty State Leaves Vault Half-Migrated

**What goes wrong:**
A migration fails halfway through (interrupted process, DB constraint, OS crash). `golang-migrate` sets `schema_migrations.dirty = 1`. Next launch, the app silently runs `Migrate.Up()` which refuses to proceed on dirty state, returns an error — and the developer's code ignores or mishandles the error and tries to use the vault anyway, leading to a half-migrated database with undefined behavior.

**Why it happens:**
- Developer runs `migrate.Up()` and only checks for `migrate.ErrNoChange` but not for dirty state.
- Production code wraps the dirty error in a generic "could not open vault" without surfacing recovery instructions.
- Test suite uses fresh DBs per test, so dirty state never appears in CI.
- PRD §2.5 says "Fail-closed on dirty migration state" but doesn't specify the UX, so the app silently boots with a corrupt schema.

**How to avoid:**
- At vault open, after running migrations, explicitly query `schema_migrations` and inspect `dirty`. If `dirty = 1`, return `MIGRATION_FAILED` with a clear message: *"This vault's schema is in a partially-migrated state from a previous interrupted operation. Restore from your latest backup of the .sqlite file, or contact the developer for recovery instructions."*
- Refuse to unlock or modify a dirty vault. No "force" path in MVP.
- Wrap migrations in a SQLite transaction where the migration system supports it (golang-migrate does this for SQLite by default for single-statement migrations; verify for multi-statement ones).
- Document the recovery procedure in README §"Known limitations": user must restore the vault file from backup.
- Test: simulate dirty state by manually setting `dirty = 1` in a fixture DB; assert app refuses to open.

**Warning signs:**
- Calls to `migrate.Up()` whose return value is only checked against `nil`.
- No dedicated `MIGRATION_FAILED` error code path in the unlock flow (it's listed in PRD §10.1, so absence is a flag).
- A "force migrate" or "auto-recover" code path.

**Phase to address:** Storage / Migration phase (initial implementation). Verified at v0.2 milestone (typed errors).

**Acceptance check:**
- Test: fixture DB with `schema_migrations` set to `dirty=1` → `OpenVault` returns `MIGRATION_FAILED`.
- Test: simulate mid-migration crash by killing the process; on next open, app surfaces the dirty error rather than silently retrying.
- README documents recovery procedure.

---

### Pitfall 9: AAD Coupling to `schema_version` Forces Re-Encryption on Every Migration

**What goes wrong:**
Someone reads the database schema and decides to include `schema_version` (or some other migration-coupled identifier) in the AAD. Now every migration that bumps the schema version invalidates every record's AAD; either records can no longer be decrypted, or the migration must re-encrypt every record (slow, error-prone, requires DEK at migration time which is wrong layering).

PRD §5.4 explicitly forbids this, but the prohibition only sticks if AAD construction is centralized and reviewed.

**Why it happens:**
- "AAD should bind ciphertext to its database state" — superficially sensible, deeply wrong.
- Confusion between `schema_version` (DB layout) and `record_envelope_version` (encryption envelope shape). The PRD intentionally separates them; including `record_envelope_version` is correct, including `schema_version` is incorrect.
- Copy-pasted AAD builder from another project that had different versioning semantics.

**How to avoid:**
- AAD helper signature does not accept `schemaVersion` as a parameter — it cannot be passed in by mistake.
- A code-level comment on the AAD helper: *"DO NOT add schema_version. See PRD §5.4."*
- Review checklist: any change to AAD shape must articulate what triggers record re-encryption.
- Increment `record_envelope_version` only when the encryption envelope itself changes (algorithm, AAD shape, blob structure). Schema migrations that add non-encrypted columns or settings rows do NOT bump it.

**Warning signs:**
- AAD builder takes `*sql.DB` or migration metadata as input.
- Code that re-encrypts records inside a migration.
- `schema_version` appears in the same file as `cipher.NewGCM`.

**Phase to address:** Crypto / AEAD phase (initial AAD design). Re-checked at every migration during all milestones.

**Acceptance check:**
- Test: bump `schema_version` (via migration) without changing record envelope; assert all records still decrypt.
- Code search: `grep -r "schema_version" internal/crypto/` returns zero matches.
- Code search: `record_envelope_version` is referenced in the AAD builder; `schema_version` is not.

---

### Pitfall 10: `crypto/rand` Error Swallowing → Zero-Filled Key

**What goes wrong:**
`crypto/rand.Read(buf)` can theoretically fail (extremely rare on macOS/Windows, but `io.ReadFull` semantics mean a short read returns an error). If the call site ignores the error and proceeds, `buf` is zero-filled. A "DEK" that is 32 zero bytes is a known-key, and every record encrypted under it is trivially decryptable. Same for nonces, salts, generated passwords.

**Why it happens:**
- Idiomatic Go code uses `_, err := rand.Read(buf)` and the `err` gets dropped during refactoring.
- A wrapper function `mustRand(n int) []byte` that calls `panic` instead of returning the error — but tests stub it without panicking.
- Code path that does `rand.Read(buf[:n])` where `n` was computed wrong (zero) and the call returns `nil` for "read 0 bytes successfully," leaving the original buffer intact.

**How to avoid:**
- Every `rand.Read` call site:
  ```go
  if _, err := rand.Read(buf); err != nil {
      return ErrInternalErrorWrap(err)
  }
  ```
- One internal helper for typical sizes:
  ```go
  func RandomBytes(n int) ([]byte, error)
  ```
  that guards `n > 0` and returns wrapped error.
- Map all `rand.Read` errors to the typed `INTERNAL_ERROR` at the boundary (PRD §10.1).
- Lint rule (golangci-lint `errcheck`) at CI to catch dropped errors from `crypto/rand`.

**Warning signs:**
- `_ = rand.Read(buf)` or `rand.Read(buf)` with no error check.
- Helper `mustRand` that panics in production.
- Tests that pass a deterministic `io.Reader` to crypto functions and the production code accepts that injection.

**Phase to address:** Crypto / AEAD phase (and Project Skeleton phase: enable errcheck in CI).

**Acceptance check:**
- `golangci-lint` errcheck enabled and CI gate.
- Unit test: stub `rand.Reader` to return an error; assert all crypto operations propagate `INTERNAL_ERROR`.
- Code search: `grep -rn "rand.Read" .` reviewed; every call site visibly checks `err`.

---

### Pitfall 11: `math/rand` Sneaks Into a Security Path Through "Helper Reuse"

**What goes wrong:**
A developer uses `math/rand` for a seemingly innocuous task — generating a UUID, a sample-data fixture, a "shuffle" of UI elements. Later, another developer reuses that helper for something security-relevant — a salt, a tag identifier that gets included in AAD, a "random" character pick in the password generator. Now one of the security-sensitive primitives is predictable.

PRD §18.1 hard rule #13 forbids `math/rand` for passwords, keys, salts, or nonces — but the rule is easy to bypass via indirection.

**Why it happens:**
- `math/rand` is the default in many Go tutorials and example code.
- `math/rand` import is one character shorter than `crypto/rand` and Go's import organizer may auto-import the wrong one.
- Fixtures and seed data use `math/rand` for determinism in tests.
- A helper named `randomString(n)` is implemented with `math/rand` and its security properties are not labeled at the call site.

**How to avoid:**
- Project-wide ban on `math/rand`. Add a CI check:
  ```bash
  grep -rn "math/rand" --include="*.go" . && exit 1 || exit 0
  ```
  (or use `go-critic`, `forbidigo` linter rule).
- All randomness goes through `internal/crypto/rng.go` with one entry point: `RandomBytes(n int)`. UUIDs come from `github.com/google/uuid` (which uses `crypto/rand` v4 by default, verify).
- For deterministic tests, inject a deterministic `io.Reader` into the function under test — never reach for `math/rand`.

**Warning signs:**
- Any file importing `"math/rand"` or `"math/rand/v2"`.
- A package called `rand` that wraps both stdlib variants.
- "Shuffle" or "pick random element" helpers without a documented source of randomness.

**Phase to address:** Project Skeleton phase (lint rule). Re-checked at every PR.

**Acceptance check:**
- CI lint rule rejects `math/rand` import in any file.
- Code review: no helper named `random*` exists outside of `internal/crypto/rng.go`.

---

### Pitfall 12: Master Password Unlock Oracle Through Differentiated Errors

**What goes wrong:**
Different error responses leak information about *why* unlock failed: "wrong password" vs. "vault file corrupt" vs. "unsupported version" vs. "KDF parameters out of range" vs. "cipher init failed." An attacker with offline access to the vault can probe these differences to short-circuit attacks (e.g., learn that a particular file *is* a vault, learn the KDF profile, learn approximate attempt cost).

PRD §9.4 mandates a single generic message; PRD §10.1 has `UNLOCK_FAILED` as a typed code. The implementation has to actively collapse all the underlying errors into that one code.

**Why it happens:**
- Error-mapping is implemented at too low a level; cipher errors, SQLite errors, KDF errors all bubble up as distinct typed errors and the boundary doesn't collapse them.
- Developer adds a "for debugging" branch to the unlock error message and forgets to remove it.
- Exception messages include underlying error text in the `details` field of `AppErrorDTO`.
- Frontend does string-matching on `message` (also forbidden by PRD §18.1 #14) and renders distinct UI per error.
- Timing oracle: KDF runs only on certain paths, so "valid file but wrong password" takes 1s, "invalid file" returns instantly.

**How to avoid:**
- Single error mapper at the Wails boundary in `app.go`:
  ```go
  func mapUnlockError(internal error) AppErrorDTO {
      if errors.Is(internal, vault.ErrUnsupportedFormat) {
          return AppErrorDTO{Code: "UNSUPPORTED_VAULT", Message: genericUnlock}
      }
      // Most cases:
      return AppErrorDTO{Code: "UNLOCK_FAILED", Message: genericUnlock}
  }
  ```
  where `UNSUPPORTED_VAULT` is allowed because the file *isn't a vault* (PRD §6.1's `application_id` mismatch is observable just by looking at the file). For everything else (wrong password, corrupt MAC, KDF failure, cipher failure, dirty migration after first format check), map to `UNLOCK_FAILED` with the single generic message.
- The `details` field of `AppErrorDTO` for `UNLOCK_FAILED` must be empty or contain only non-leaky data.
- Always run the KDF on unlock attempt (even if you can short-circuit on application_id mismatch — but prefer not to, to keep timing roughly constant). The KDF run is the dominant cost and naturally normalizes timing.
- Use `errors.Is` / `errors.As` to recognize internal error types; never `errors.Error()` string-match.

**Warning signs:**
- More than one return path from `UnlockVault` that produces a non-`UNLOCK_FAILED` error.
- An `UNLOCK_FAILED` error whose `message` varies depending on cause.
- Frontend code that branches on `error.message` rather than `error.code`.
- A debug log that prints the underlying cause to the user-facing error.

**Phase to address:** Vault Lifecycle phase (initial). Hardened at v0.2 (Security Foundation, typed error system).

**Acceptance check:**
- Test: unlock with wrong password → `UNLOCK_FAILED`.
- Test: unlock with corrupt wrapped DEK (mutate ciphertext byte) → `UNLOCK_FAILED`.
- Test: unlock with tampered AAD on the wrapped DEK → `UNLOCK_FAILED`.
- Test: open a non-vault file (wrong `application_id`) → `UNSUPPORTED_VAULT` (allowed).
- Test: open a half-migrated vault → `MIGRATION_FAILED` (allowed; not an unlock attempt).
- Test: timing variance between wrong password and corrupt-MAC unlock attempts is within KDF noise (no easy oracle).

---

### Pitfall 13: Wails Binding Type Drift Between Go and TypeScript

**What goes wrong:**
Go DTO struct and TypeScript DTO type drift apart. A field gets renamed in Go, frontend continues to read the old name, the field arrives as `undefined`, and the frontend silently treats it as missing — for `isUnlocked: boolean`, that means showing the unlock screen forever; for `vaultId: string`, that means joining records to the wrong vault; for `code: AppErrorCode`, that means falling back to an "unknown error" branch that exposes raw underlying messages.

**Why it happens:**
- TS types are hand-written from PRD §10 and not regenerated.
- Wails generates types via `wails dev` but the generated file is gitignored and per-developer.
- JSON struct tags on Go side disagree with TS field names (Go's default is uppercase, JSON marshaling lowercases first letter — but only if the tag is set correctly).
- A DTO field gets a new optional field; old frontend doesn't know about it; new field carries critical information.

**How to avoid:**
- Use Wails' generated TypeScript bindings (`wails generate module`) and check them in to git so type drift fails CI.
- Or: maintain a single source-of-truth schema (e.g. a `dtos.go` file) and a script that regenerates `dtos.ts` at build time; CI fails if `dtos.ts` is out of date.
- All Go DTO structs use explicit `json:"fieldName"` tags. Don't rely on default casing.
- TS types use `strict` mode; missing fields are compile errors, not runtime undefineds.
- Frontend test: mock the Wails binding with a typed factory; assert at compile time that the mock matches the Go struct shape (via the regenerated file).

**Warning signs:**
- TS DTO file is hand-edited and not generated.
- Go DTO without `json:` tags.
- Frontend code that does `if (response.someField)` checks for fields that should always be present.
- `wails generate` is in a developer's local script but not in CI.

**Phase to address:** Project Skeleton phase. Re-verified at every milestone that adds DTOs.

**Acceptance check:**
- CI step: `wails generate module` produces no diff against committed bindings.
- All Go DTO structs have `json:` tags on every field.
- Frontend tsconfig has `strict: true`.

---

### Pitfall 14: Cross-Platform Path Handling

**What goes wrong:**
Vault file paths break on Windows because of:
- Forward-slash vs. backslash separators.
- Drive-letter prefixes (`C:\`).
- Long-path limit (260 chars, unless prefixed with `\\?\`).
- Spaces and unicode in path names (especially Windows MAX_PATH and macOS NFD vs. NFC normalization).
- File dialog return types differ between OSes (some give absolute paths, some give file:// URLs).

User sees confusing errors like "vault not found" when the path is actually correct but encoded differently.

**Why it happens:**
- Use of `path` package (Unix-only semantics) instead of `path/filepath` (OS-aware).
- String concatenation `dir + "/" + filename` instead of `filepath.Join`.
- Hardcoded `/` in tests.
- Storing the path in settings using one format and reading it back using another.
- macOS Finder gives Unicode NFD (decomposed), user types the same name in NFC (composed); strings compare unequal.

**How to avoid:**
- Always use `path/filepath` (`Join`, `Clean`, `Abs`, `Ext`) in Go for OS paths. `path` is for slash-separated logical paths only (URLs, archive entries).
- Normalize paths at app boundary: `filepath.Clean(filepath.FromSlash(input))`.
- Unicode-normalize file names when comparing: `golang.org/x/text/unicode/norm` → NFC.
- For long paths on Windows, opt-in to long path support (manifest setting + `\\?\` prefix where appropriate).
- File dialog: use Wails' built-in dialog (`runtime.OpenFileDialog`) which normalizes return types per OS.
- Test fixture paths use `t.TempDir()` and `filepath.Join`, never literal `/` or `\`.

**Warning signs:**
- `import "path"` when handling filesystem paths.
- String-concatenated paths.
- Hardcoded `/` or `\` in path strings.
- Tests that pass on one OS and fail on another with "file not found."

**Phase to address:** Storage / Migration phase (initial — vault path handling), again at Cross-Platform Hardening (v0.5), again at Key Export phase (export destination paths).

**Acceptance check:**
- Manual test on macOS and Windows per PRD §14.3.
- Lint rule: forbid `import "path"` outside of explicit URL/archive contexts.
- Test: create vault at path with Unicode name; reopen via file dialog; assert path matches.

---

### Pitfall 15: macOS Code-Signing Friction Blocks Dev Testing

**What goes wrong:**
`wails dev` works fine locally, but `wails build` produces an unsigned `.app` that macOS Gatekeeper blocks ("application cannot be opened because the developer cannot be verified") unless quarantined. Developer ships an "MVP build" that nobody can launch without arcane terminal commands. Worse, an end-user double-clicks the `.app`, gets a generic Apple error, and concludes the app is broken.

**Why it happens:**
- Apple Developer ID costs money and has a setup process; learning-grade project skips it.
- Signed-but-not-notarized builds also fail Gatekeeper on macOS 10.15+.
- Wails defaults to building unsigned binaries unless config specifies a signing identity.
- README doesn't document the workaround.

**How to avoid:**
- Document in README the `xattr -d com.apple.quarantine /Applications/Abyss.app` workaround OR the right-click → Open path that bypasses Gatekeeper for unsigned apps.
- Optional: invest in Apple Developer ID for at least signed-but-not-notarized builds (still triggers a Gatekeeper warning, but a less severe one).
- Document Windows SmartScreen equivalent: unsigned `.exe` triggers "Windows protected your PC" — clicking "More info → Run anyway" is the workaround.
- Don't gate v0.1–v0.5 progress on signing; defer to v1.0 polish phase.
- For CI/release artifacts, mark them clearly as "unsigned development builds — see README for unblocking."

**Warning signs:**
- A `wails build` succeeds but the resulting binary won't launch on a clean macOS.
- README has no "macOS Gatekeeper" section.
- User reports "the app crashes immediately" without a crash log (Gatekeeper blocked it before launch).

**Phase to address:** Cross-Platform Hardening (v0.5) and Documentation / Threat Model phase (README).

**Acceptance check:**
- README §macOS setup includes the quarantine-removal command.
- README §Windows setup includes the SmartScreen unblock instructions.
- A CI build artifact is downloaded and successfully launched by a non-author on a clean Mac and a clean Windows VM.

---

### Pitfall 16: Idle Timer Skewed by System Sleep / Lid-Close

**What goes wrong:**
User unlocks vault, closes laptop lid for 8 hours, reopens. The 10-minute idle timer either:
- Fires immediately (Go timers paused during sleep, fire on resume),
- Fires "instantly" relative to sleep duration but vault has been theoretically unlocked for 8 hours,
- Doesn't fire because the timer was rescheduled wrong.

In the worst case, the vault is still unlocked after the laptop has been physically transported and opened by someone else.

PRD §13.4 marks system-sleep handling as Should-have, not required for v0.1. So MVP can skip it, but the documented behavior must match what the implementation does.

**Why it happens:**
- `time.AfterFunc` and `time.Timer` are wall-clock-ish but their behavior across sleep is OS-specific.
- Activity timestamp is updated on user input but not re-checked on system resume.
- Sleep events from the OS are not subscribed to.

**How to avoid:**
- Track `lastActivityAt time.Time` (wall clock) on every user activity event.
- On every "tick" of the idle check (suggest a 1-second poller, not a single Timer), compare `time.Since(lastActivityAt)` against the configured timeout. If exceeded, lock.
- A 1-second poller automatically handles sleep/wake because on resume it just observes that wall-clock has jumped.
- If subscribing to OS power events is feasible (via `wails`/`gopsutil`/platform-specific APIs), trigger immediate lock on suspend. Defer to post-v0.1 per PRD §13.4.
- Document the v0.1 behavior: "auto-lock fires when the app is in foreground and idle for [timeout]; behavior across sleep is best-effort."

**Warning signs:**
- A single `time.AfterFunc(timeout, lock)` with no re-check on activity.
- An auto-lock test that runs in `< 1s` of fake-time but never tests wall-clock skew.
- After-system-sleep manual test not in the v0.5 cross-platform matrix.

**Phase to address:** Auto-Lock phase (v0.2) for the polled-wall-clock check. System sleep events deferred to v0.5+ per PRD §13.4.

**Acceptance check:**
- Test: simulate `lastActivityAt = now() - 30 minutes` while configured timeout is 10 minutes; on next tick, vault should be locked.
- Manual: unlock vault, sleep laptop, wake after >timeout — vault should be locked.

---

### Pitfall 17: PKCS#8 vs. PKCS#1 RSA Private Key Format Confusion

**What goes wrong:**
PRD §FR-046 mandates PKCS#8 PEM (`-----BEGIN PRIVATE KEY-----`). Many Go tutorials use `x509.MarshalPKCS1PrivateKey` (`-----BEGIN RSA PRIVATE KEY-----`). They look similar in PEM output but they are different ASN.1 structures and consumers may not accept the wrong one. Worse, openssl tools accept both transparently, so dev testing passes; user's downstream tool (e.g. an SSH server, a JWT signer) rejects PKCS#1 and the user has no idea why.

**Why it happens:**
- `x509.MarshalPKCS1PrivateKey` exists, is named simply, and is the first hit in tutorials.
- PKCS#8 marshaling requires `x509.MarshalPKCS8PrivateKey` which takes `interface{}` (any private key) and is the more general API.
- PEM headers differ (`RSA PRIVATE KEY` vs. `PRIVATE KEY`) but tooling often accepts either.

**How to avoid:**
- Always use `x509.MarshalPKCS8PrivateKey(privateKey)` — it works for RSA, Ed25519, ECDSA uniformly. Wrap in PEM block with type `"PRIVATE KEY"` (not `"RSA PRIVATE KEY"`).
- Test: unmarshal the exported key with `x509.ParsePKCS8PrivateKey` and verify it round-trips to the same key.
- Test: parse the PEM block; assert the type is exactly `"PRIVATE KEY"`.
- Document the format clearly in the export UI: "PKCS#8 PEM (compatible with most modern tools)."

**Warning signs:**
- Any reference to `MarshalPKCS1PrivateKey` in the codebase.
- PEM header `-----BEGIN RSA PRIVATE KEY-----` in generated output.
- Tests that round-trip via openssl rather than via Go's parser.

**Phase to address:** Key Export phase (v0.4).

**Acceptance check:**
- Test: export RSA key → parse with `x509.ParsePKCS8PrivateKey` → round-trip OK.
- Test: export Ed25519 key → parse with `x509.ParsePKCS8PrivateKey` → round-trip OK.
- Code search: zero hits for `MarshalPKCS1`.

---

### Pitfall 18: OpenSSH Private Key Format for Ed25519 (No Stdlib Helper)

**What goes wrong:**
PRD §FR-054 makes "export Ed25519 private key in OpenSSH format" a Should-have. The OpenSSH private key format (used by `ssh-keygen` since OpenSSH 6.5+, file starts with `-----BEGIN OPENSSH PRIVATE KEY-----`) is a custom format with:
- Magic bytes `openssh-key-v1`,
- Cipher / KDF / public key / private key sections,
- A specific check-int verification mechanism,
- Optional passphrase encryption (bcrypt + AES).

Go's stdlib does NOT have a direct marshaler for this format. `golang.org/x/crypto/ssh` can *parse* it but not *marshal* it (as of late 2025 — newer versions added `MarshalPrivateKey` and `MarshalPrivateKeyWithPassphrase`; verify version). Naive implementations get it subtly wrong (wrong padding, wrong check-int, wrong cipher block alignment) and the resulting file will not round-trip through `ssh-keygen`.

**Why it happens:**
- It's a Should-have, so under deadline pressure someone hand-rolls it.
- The format is documented in `PROTOCOL.key` in OpenSSH source — most Go developers won't read it carefully.
- "It looks like a PEM file, must be similar" leads to wrong attempts.

**How to avoid:**
- **Recommended:** verify `golang.org/x/crypto/ssh.MarshalPrivateKey` (or `MarshalPrivateKeyWithPassphrase`) is available at the Go module version used; use it. If not available at chosen version, upgrade or defer.
- If stdlib helper is unavailable: defer FR-054 to post-v1.0 ("Should" is not "Must" — PRD §FR-053 already covers the Must with PKCS#8 PEM).
- Do NOT hand-roll the OpenSSH private key format.
- Document in README: "Ed25519 private key export: PKCS#8 PEM is supported. OpenSSH private key format is planned but not yet implemented."

**Warning signs:**
- Hand-rolled byte-buffer code that writes the magic `openssh-key-v1`.
- A function `marshalOpenSSHPrivate` with more than ~10 lines of logic.
- Tests for OpenSSH private export that don't round-trip via `ssh.ParsePrivateKey`.

**Phase to address:** Key Export phase (v0.4). Defer to post-v1.0 if stdlib doesn't support cleanly.

**Acceptance check:**
- If implemented: round-trip test via `ssh.ParsePrivateKey` on the exported bytes.
- If implemented: external test via `ssh-keygen -y -f exported.key` produces the matching public key.
- If deferred: README clearly states which formats are supported and which aren't.

---

### Pitfall 19: Public Key Fingerprint Format Inconsistency

**What goes wrong:**
The public key fingerprint stored in `records.public_key_fingerprint` ends up in different formats across the codebase. Some places use the legacy MD5-colon format (`md5:5e:b8:af:...`), some use the modern OpenSSH SHA-256 format (`SHA256:abc123base64...`), some use raw bytes hex. Users compare a copied fingerprint to one displayed elsewhere (e.g., GitHub deploy keys page) and they don't match — but they're the same key, just different format.

**Why it happens:**
- Old SSH tutorials still show MD5 fingerprints.
- Different tools display different formats by default.
- No project-wide convention is set.

**How to avoid:**
- Project standard: SHA-256 fingerprint in the OpenSSH format `SHA256:<base64-no-padding>`.
- One helper: `crypto/keys.OpenSSHFingerprintSHA256(publicKey ssh.PublicKey) string`.
- All UI displays use this helper. All exports include this fingerprint in the same format.
- Document the format in the UI tooltip: "SHA-256 fingerprint, OpenSSH format (matches `ssh-keygen -l -f`)."
- Test: compare against `ssh-keygen -l -E sha256 -f keyfile.pub` for known test vectors.

**Warning signs:**
- Multiple fingerprint helpers in different files.
- Any reference to MD5 in fingerprint code.
- Hex-encoded output for fingerprints (should be base64-no-padding).

**Phase to address:** Key Generation / Key Export phase (v0.4).

**Acceptance check:**
- Single helper exists; all call sites use it.
- Test: generated fingerprint matches `ssh-keygen -l -E sha256` output for fixture keys.

---

### Pitfall 20: Clipboard "Clear" Silently Fails

**What goes wrong:**
The app calls "clear clipboard" after the timeout, the call returns success, but the clipboard still contains the secret because:
- macOS pasteboard's "general" board pinned by another app.
- Windows clipboard contention (another app held it open, the clear failed silently).
- A clipboard manager (Alfred, Maccy, Ditto, Windows clipboard history) preserved the value before clear.
- The clear wrote an empty string but the OS-level history retained the prior value.

User believes secret is cleared; it isn't. PRD §FR-063 acknowledges this and requires documenting the limitation.

**Why it happens:**
- "Clear clipboard" APIs return success when they wrote, not when nobody else can read the prior value.
- Clipboard history is not part of the clipboard API; the app cannot inspect or clear it.
- macOS has multiple pasteboards (`general`, `find`, `font`, `ruler`, `drag`); clearing one doesn't affect others.

**How to avoid:**
- Treat clipboard clear as best-effort. The UI message should be: "Clipboard clear attempted." NOT "Clipboard cleared."
- Implement clear by writing an innocuous value (single space or empty string) rather than relying on a "clear" API. PRD FR-064 mentions "harmless overwrite before clearing" as Could-have.
- README documents the limitation per PRD §FR-063: "Clipboard clearing is best-effort. Operating system features and third-party clipboard managers may retain clipboard history outside this app's control." (The exact wording from PRD §19 README requirements.)
- Log "clipboard clear attempted" at debug level; never log success/failure as a security guarantee.
- Do NOT panic or surface errors to the user when clipboard write fails — at most show a toast "Could not access clipboard."

**Warning signs:**
- UI text reading "Clipboard cleared" or "Secret removed from clipboard."
- Code that asserts the clipboard write returned success and treats failure as security-critical.
- README missing the clipboard limitations section.

**Phase to address:** Clipboard phase (v0.1) and Documentation phase (README, v0.5–v1.0).

**Acceptance check:**
- All clipboard UI strings reviewed for "attempted" vs. "succeeded" wording.
- README contains the verbatim warning from PRD §19.
- Manual test: copy secret, install/use a clipboard manager (Alfred/Maccy on macOS, Windows clipboard history), verify the manager retains the value despite app's clear — and that this is documented behavior.

---

### Pitfall 21: JSON Metadata Blob Schema Drift Without Versioning

**What goes wrong:**
The encrypted metadata/secret blobs are JSON objects (PRD §7). In v0.3 someone adds a new field (e.g., `expiration_date` to API key records). Old vaults' blobs don't have that field; they still decrypt fine but the deserialized struct has zero-valued `expiration_date`, which the UI may render as "expires 0001-01-01" or crash. In v0.5 someone changes a field type (`tags []string` → `tags []Tag` for nested data), and old blobs cannot be deserialized at all without a migration path.

**Why it happens:**
- JSON schemas drift naturally when adding features.
- Without a version field inside the blob, there's no way to know how to deserialize.
- `record_envelope_version` (PRD §6.3) is at the row level, not the blob level — bumping it forces re-encryption, which is exactly what we want to avoid for ordinary schema additions.

**How to avoid:**
- Keep JSON additions backward-compatible:
  - Only ADD fields, never RENAME or REMOVE.
  - All new fields must be optional (Go: pointer or `omitempty`; TS: `?`).
  - Type changes are forbidden — new type means new field name.
- For breaking blob changes: bump `record_envelope_version`, write a migration that re-encrypts. This is rare and intentional.
- Decode with permissive options: ignore unknown fields, accept missing fields.
- Defaults computed at decode time, not stored: `expiration_date == nil` means "no expiration."

**Warning signs:**
- Blob struct fields with non-optional types after the first release.
- A field renamed in the Go struct without a JSON tag preserving the old name.
- A "v2 blob" type without a corresponding `record_envelope_version` bump.

**Phase to address:** Records / Encryption phase (initial design — set the convention early); Expanded Records phase (v0.3) when first new types are added.

**Acceptance check:**
- Test: decode v0.1-shape blob into v0.3-shape struct → succeeds with zero values for new fields.
- Convention documented: "JSON blobs are append-only; field renames or removals require `record_envelope_version` bump and a migration."

---

### Pitfall 22: Recent-Vaults / Last-Vault-Path UX Subverts Lock

**What goes wrong:**
A "remember last vault" feature stores the vault file path in app preferences (or in `settings` table, or in `localStorage`) so the user doesn't have to navigate to it every time. Convenient — but now anyone with access to the unlocked OS sees the vault path in the welcome screen, can copy the file, and try offline brute-force attacks on it. Worse if the path is leaked alongside any session token or "remember me" state.

PRD §9.2 already defers recent-vaults unless implementation is trivial and stores no sensitive data — but the deferral discipline must be enforced.

**Why it happens:**
- "Recent files" is a standard desktop UX expectation.
- Path itself isn't a secret in the strict sense — but it's a strong locator.
- Convenience pressure builds across milestones.

**How to avoid:**
- Defer "recent vaults" entirely for v0.1–v1.0. PRD §9.2 already says so.
- Welcome screen shows only "Create new" and "Open existing" actions. Open Existing always invokes the file dialog.
- If a recent-vaults feature is added post-v1.0, store ONLY the path and the last-opened timestamp, never any session/unlock state.
- Never persist any "remember password" / "stay unlocked across restarts" feature.

**Warning signs:**
- A `recent_vaults` table or settings entry.
- A `lastVaultPath` field in any persistent store.
- A "remember me" checkbox on the unlock screen.

**Phase to address:** Vault Lifecycle phase (initial) — keep the welcome screen minimal. Re-checked at every milestone via PRD §9.2 compliance review.

**Acceptance check:**
- Settings table contains no key matching `*vault*path*` or `*recent*`.
- Welcome screen has no "Recent" section.

---

### Pitfall 23: zxcvbn Password Strength Estimator Library Selection

**What goes wrong:**
PRD §FR-033 makes password strength display a Should-have, with the constraint of using a vetted library. JavaScript ports of zxcvbn vary widely:
- Original `dropbox/zxcvbn` (2016, unmaintained).
- `zxcvbn-ts` (TypeScript port, maintained as of recent).
- `@zxcvbn-ts/core` (newer modular variant).
- Random forks with security issues and abandoned dictionaries.

Picking a stale or buggy port means inaccurate strength estimates (false security) or supply-chain risk.

**Why it happens:**
- npm has many similarly-named packages.
- "It works" doesn't mean "it's accurate" or "it's maintained."

**How to avoid:**
- Use `@zxcvbn-ts/core` + `@zxcvbn-ts/language-en` (the actively maintained TypeScript port, as of late 2025/early 2026 — verify currency at implementation time).
- Pin exact versions in `package.json`; review npm audit at every dependency update.
- The strength meter is advisory only; do NOT block passphrases on a low score. PRD §11.1 says "passphrases must be allowed" and "arbitrary composition rules are not required."
- If the library is unavailable / abandoned by the time of implementation, defer the feature — it's a Should-have, not a Must-have.

**Warning signs:**
- Importing from `dropbox/zxcvbn` (the original, unmaintained).
- Implementing strength logic in-house ("I'll just count entropy").
- Strength score gates the form submission (not allowed per PRD).

**Phase to address:** Password Generator phase (v0.1) — but defer until library is verified.

**Acceptance check:**
- Library version pinned and reviewed.
- Strength score is advisory; no gating on master password creation.

---

## Technical Debt Patterns

Shortcuts that look reasonable but accumulate cost over the milestone arc.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Skip AAD on first AES-GCM implementation, "add it later" | Faster v0.1 prototype | Re-encrypting all records when AAD is added; harder to verify tamper-resistance retroactively. | Never — PRD §17 requires AAD by MVP. Implement AAD from the very first encrypt call. |
| Hand-write TS DTOs from PRD §10 | No tooling setup | Type drift on every Go DTO change; subtle frontend bugs. | Never long-term. Acceptable for the first day of Project Skeleton phase only. |
| Single `kdfParams` package var that tests mutate | Easy test setup | Production-config regression risk (Pitfall 3). | Never. Use separate testing-only package or build tag. |
| Store tags in plaintext column "for search" | Quick FTS implementation | Plaintext leakage; PRD FR-026 is encrypted-tags Should-have but PRD §17 #14 forbids any plaintext. | Never. Search is in-memory after unlock per PRD §FR-022. |
| Skip nonce-uniqueness test, "rand is rand" | Saves ~20 lines of test code | If a future refactor introduces a counter or buggy nonce helper, you don't catch it. | Never. Nonce reuse is catastrophic. |
| Use `path` instead of `path/filepath` "since we're on macOS for dev" | Code looks cleaner | Windows breaks at v0.5 cross-platform milestone. | Never — Windows is in scope from v0.1 per PRD §2.3. |
| Single-error generic in `UnlockVault` ("invalid input") | Simpler error path | Loses ability to surface `MIGRATION_FAILED` or `UNSUPPORTED_VAULT` distinctly when those are non-leaky cases. | Never — the typed `AppErrorCode` enum is the contract. |
| Defer cross-platform testing to v0.5 entirely | Faster v0.1–v0.4 | A path-handling or Wails-binding issue discovered at v0.5 means rewriting earlier code. | Acceptable to develop on one OS day-to-day; not acceptable to skip OS smoke-tests at each milestone boundary (PRD §2.3). |
| Persist "remember last vault path" early because users will want it | One fewer click | Vault locator leak; sets precedent for other "convenience" features that erode security posture. | Never in MVP per PRD §9.2. |
| Implement OpenSSH private key marshaling by hand (Pitfall 18) | "It's just bytes" | Format errors that round-trip-fail through ssh-keygen; user export is rejected by downstream tools. | Never. Use stdlib or defer the feature. |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| `golang-migrate` + Wails embed | Migrations as filesystem files; break in packaged binary | Use `go:embed` with `iofs` driver: `migrate.NewWithSourceInstance("iofs", d, "sqlite://...")`. |
| `mattn/go-sqlite3` cgo build | cgo not enabled on Windows CI; build silently uses pure-Go alternative or fails | Set `CGO_ENABLED=1` in build env; verify with `go list -m all`; alternatively use `modernc.org/sqlite` (pure-Go). Pick one and document. |
| Wails event subscription on lock | Frontend subscribes to `LockEvent` but doesn't unsubscribe on component unmount → memory leak / stale handlers | Use a single global subscription in app root; per-component listeners use `useEffect` cleanup. |
| `crypto/rand.Reader` in goroutines | Reader is safe for concurrent use, but tests stub it with non-thread-safe replacements | Document concurrency contract on stubs; use `sync.Mutex` in test stubs if needed. |
| `golang.org/x/crypto/argon2` `IDKey` parameters | Argument order: `password, salt, time, memory, threads, keyLen` is easy to mis-order | Wrap in a typed `Params` struct, never call `IDKey` directly from business logic. |
| `golang.org/x/crypto/ssh` public key marshaling | Some types require `ssh.NewPublicKey(*rsa.PublicKey)` first; wrong wrapper produces wrong bytes | Test against `ssh-keygen` output; explicitly call `ssh.MarshalAuthorizedKey`. |
| File dialog on macOS via Wails | macOS returns paths with NFD-normalized Unicode; comparisons fail against NFC user input | Normalize all paths with `golang.org/x/text/unicode/norm.NFC.String(p)` at boundary. |
| SQLite `application_id` PRAGMA | Set with `PRAGMA application_id = N;` at create; check with `SELECT application_id FROM pragma_application_id;` (or `PRAGMA application_id`) at open — but the result is `int` not `text` | Always read as int and compare to `0x4E56544E` (PRD §6.1). Reject non-matches before any other DB operation. |

---

## Performance Traps

This is a single-user local desktop app; "scale" means "personal vault grows over years."

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Argon2id memory pressure on low-end machines | Unlock takes 10+ seconds or OOMs on a laptop with 4 GiB RAM | PRD §5.3 fallback profile (64 MiB) and minimum (19 MiB); benchmark at first run; let user choose. | A 256 MiB Argon2id call on a 4 GiB MacBook Air with browser tabs open. |
| In-memory search over thousands of records | UI hitches when typing in search box | Debounce search input (200 ms); pre-tokenize titles on unlock into a search index in memory; lazy-decrypt only metadata, not secrets, for list view. | ~2,000+ records (rare for personal vault). |
| Eager decryption of all records at unlock | Long unlock times after vault grows; high memory residency → bigger memory-exposure window | Decrypt only metadata for record list (PRD §10.3 RecordSummaryDTO); decrypt secret blob only when user opens record detail. | ~500+ records. |
| Re-encrypting all records on master password change | Multi-second hang on change-password; battery drain; no good cancellation story | DON'T re-encrypt records — only re-wrap DEK (PRD §5.2 explicit). | Always wrong, regardless of scale. |
| RSA 4096 keygen blocks UI thread | UI freezes during keygen | Run keygen in a goroutine; emit progress events; show indeterminate spinner. RSA 4096 can take 1–5 seconds even on modern hardware. | Always — RSA 4096 is slow. RSA 2048 is fine to do synchronously. |
| Migration on every app open on a multi-MB vault | Slow open even when migrations are no-ops | `golang-migrate.Up` is fast when no migrations to run, but check it; avoid running on every screen change (only at vault open). | Vault > 50 MB or many no-op migration files. |

---

## Security Mistakes

Domain-specific issues beyond the critical pitfalls above.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Including the master password in stack traces | Master password leaks to log files / error reporters | Convert master password to `[]byte` and zero it before any function that could panic; never include in error wrappers. |
| Storing vault path in macOS Keychain or Windows Credential Manager "for convenience" | OS-level credential store leak; couples vault to OS account | Don't. Vault portability requires the file alone to be sufficient. |
| Using `==` to compare KEK-derived bytes | Timing oracle on byte-by-byte short-circuit comparison | Always `crypto/subtle.ConstantTimeCompare`. (For Abyss specifically, this matters for any auth-tag-like comparison; AES-GCM `Open` already does it internally, but be careful with custom verifiers.) |
| Trusting `record_type` from the row when decrypting | If row is tampered, attacker can swap `record_type` and AAD will match (because we read it from the row) | Bind `record_type` to the encrypted blob via AAD, but also verify after decrypt that the deserialized blob's content matches the expected type for that row. |
| Allowing the vault file to be opened while another instance has it open | Concurrent writes → DB corruption or lost updates | SQLite file locking handles this, but the UX should detect and show a clear error. Use `BUSY_TIMEOUT` and surface "vault is in use by another process." |
| Accepting any PEM block on import (not currently in MVP, but worth flagging if added) | PEM parsing libraries have historical CVEs (re: PEM bomb, infinite-loop) | Don't add PEM import in MVP. If added later, set max-size limits and use stdlib parsers only. |
| `//go:debug` flags or `unsafe` package usage in crypto code | Bypass of memory safety; potential for buffer reuse bugs | Forbid `unsafe` in `internal/crypto`. Lint or review-gate. |
| Logging the *length* of secrets ("password length: 24") | Length-based oracle; even safer than full content but still leaks data | Don't log lengths of secrets either. Log only fixed-vocabulary tags ("encrypt_record_metadata: ok"). |
| Using a SQLite ATTACH to merge two vaults | Unintended cross-vault data exposure if AAD doesn't include `vault_id` (it does, per PRD §5.4) | Don't add ATTACH-based features in MVP. AAD's `vault_id` binding is the safety net. |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Master password input shows characters by default | Shoulder-surfing; password manager managers' suggestions for "show password" can leak | Default masked; "show" toggle is per-input and resets on blur. |
| "Wrong password" error tells user how many attempts remain | Implies a lockout exists, encourages probing; PRD doesn't have lockouts | Show generic message every time. No counter. (PRD §9.4.) |
| Unlock screen shows the vault file path | Leaks vault location to anyone glancing at the screen | Show only the vault filename or just "Unlock vault." |
| Lock state is ambiguous (small icon, no banner) | User assumes vault is locked when it isn't | Big visual difference: locked = greyed out / disabled list / "Locked" banner. PRD §9.5 mandates this. |
| Private-key export warning becomes muscle memory ("OK OK OK") | User clicks through without reading | Require explicit secondary action: typed confirmation phrase, or longer hold-to-confirm, or clear two-step modal. PRD §9.8 mandates the modal text. |
| Clipboard countdown shown only as a number, no progress bar | Hard to perceive remaining time at a glance | Progress bar AND number per PRD §9.7. |
| "Saved!" toast lingers showing the saved record's title | Title might be sensitive (e.g. "AWS prod root keys") | Toast says "Record saved" with no title. |
| Error messages that mix dev-friendly and user-friendly text | Confusing; might leak | Two layers: log dev-friendly internally (without secrets); show typed-code-mapped user-friendly text only. |
| Timestamps shown in UTC | User confusion about "when did I last unlock" | Local time with explicit timezone label. |
| "Remember password for this session" checkbox | Defeats lock entirely | Don't. Master password is entered every unlock. Period. |

---

## "Looks Done But Isn't" Checklist

Before marking each milestone "done," verify:

### v0.1 Dogfoodable Vault
- [ ] **Vault create:** Verify the actual SQLite file has `application_id = 0x4E56544E` set (`PRAGMA application_id` at file inspection).
- [ ] **Vault unlock:** Wrong password fails with generic message; right password succeeds; both take similar wall-clock time (within KDF noise).
- [ ] **Vault lock:** DEK byte slice is overwritten (test via debug-only inspection); session pointer swapped; subsequent decrypt attempts return `VAULT_LOCKED`.
- [ ] **Records / encryption:** Run the "grep test" — populate sentinel-string records, close DB, scan the .sqlite file for sentinels. Zero hits.
- [ ] **AAD enforcement:** AAD-tampering test passes (swap secret blob between two records → decrypt fails). Even though PRD lists this in v0.2 exit criteria, AAD must be present from v0.1.
- [ ] **Nonce uniqueness:** 1M-call test produces no collisions and no zeros.
- [ ] **Argon2id parameters:** Production code uses `kdf.DefaultParams()`, not test parameters. `key_wrappings.kdf_params_json` matches PRD §5.3 target or fallback.
- [ ] **Password generator:** Generated 24-char password contains characters from all enabled classes (length only is not enough; verify class coverage).
- [ ] **Clipboard countdown:** Reset countdown when a new secret is copied; lock event triggers immediate clear attempt.
- [ ] **TypeScript bindings:** `wails generate module` produces no diff.
- [ ] **CI lint:** No `math/rand` import; no `unsafe` in crypto; `errcheck` enabled.

### v0.2 Security Foundation
- [ ] **Auto-lock:** Idle timer survives sleep/wake (poll-based, not single-shot).
- [ ] **Change master password:** Records' ciphertext bytes are byte-for-byte identical before and after change (only `key_wrappings` row changes).
- [ ] **Typed errors:** Frontend uses `code`, never `message`; verify by mocking error responses with same code but different messages and asserting same UI behavior.
- [ ] **Unlock oracle test:** Wrong password vs. corrupt MAC vs. tampered AAD all return identical `UNLOCK_FAILED` with identical message.
- [ ] **Migration dirty state:** Fixture vault with `dirty=1` → `MIGRATION_FAILED` surfaced cleanly.

### v0.3 Expanded Records
- [ ] **All record types:** Run the grep test against vaults containing one of each record type. No plaintext sensitive data.
- [ ] **Encrypted tags:** Tag values appear nowhere in plaintext columns.
- [ ] **Search:** Search query never decrypts more than it needs (lazy decrypt of secret blobs).
- [ ] **JSON blob compatibility:** v0.1-shape blobs (e.g., credential records created in v0.1) still decode after v0.3 deployment.

### v0.4 Key Generation and Export
- [ ] **PKCS#8 format:** Exported private keys parse via `x509.ParsePKCS8PrivateKey` for both RSA and Ed25519.
- [ ] **OpenSSH public key:** `ssh-keygen -l -f exported.pub` produces a valid fingerprint that matches our stored fingerprint.
- [ ] **Fingerprint format:** All fingerprints display as `SHA256:<base64>`.
- [ ] **Private key warning modal:** Cannot be bypassed by direct keyboard navigation; tested with screen reader.
- [ ] **Key generation in goroutine:** RSA 4096 keygen does not freeze the UI.

### v0.5 Cross-Platform Hardening
- [ ] **Build artifacts:** Both macOS `.app` and Windows `.exe` exist and launch on a clean machine.
- [ ] **README:** macOS Gatekeeper unblocking instructions are tested by a non-author.
- [ ] **README:** Windows SmartScreen instructions are tested by a non-author.
- [ ] **File dialog:** Open vault dialog works for paths with spaces, unicode, and long names on both OSes.
- [ ] **Manual matrix:** Every row in PRD §14.3 has a fresh check on both OSes.
- [ ] **Final grep test:** Vault produced through real workflow (not a fixture) passes the no-plaintext check.

### v1.0 MVP Complete
- [ ] **README:** Contains all 14 sections per PRD §19.
- [ ] **README:** Contains the verbatim warnings for security positioning, clipboard, master password recovery, Go memory limitations.
- [ ] **Threat model:** Documented per PRD §4.2 with both protected-against and not-protected-against lists.
- [ ] **All Definition of Done items (PRD §17) verified:** 30 items, each independently checked.

---

## Recovery Strategies

When pitfalls happen despite prevention.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Nonce reuse discovered after release | HIGH | Force-rotate DEK: generate new DEK, re-encrypt all records with new DEK + new nonces, archive old DEK as `retired`. Notify user that any vault used during the bug window may have been compromised. |
| AAD shape change required (record_envelope_version bump) | MEDIUM | Write a migration that decrypts records under old AAD, re-encrypts under new AAD with new nonce, increments `record_envelope_version`. Test on fixture vaults; back up before running. |
| Argon2id parameters too low in shipped build | HIGH | Force-rewrap on next unlock: derive new KEK with correct params, store new `key_wrappings` row, retire old. User experience: one slow unlock, then transparent. |
| SQLite plaintext leak discovered | HIGH | The SQLite file with the leak still exists in the user's filesystem and any backups. Recovery: instruct user to securely delete old vault file, re-create vault from scratch with all records re-entered. There is no in-place fix because the disk has already-leaked plaintext. |
| Dirty migration state | MEDIUM | Restore from backup. If no backup, manual SQL to inspect what was applied, finish manually, set `dirty=0`. Document procedure in README. |
| Master password forgotten | TOTAL LOSS | No recovery. Document this prominently per PRD §19 (no-recovery warning). |
| TypeScript binding drift discovered post-release | LOW | Regenerate bindings, ship patch. Affects future builds; users with old build see UI bug but no data loss. |
| Auto-lock not actually clearing DEK in some path | HIGH if exploited | Audit all code paths that hold DEK references; add the missing zero/clear; ship patch. Recommend users update immediately and lock+unlock at least once after update to flush stale state. |
| OpenSSH private key export produces invalid format | LOW | User noticed when downstream tool rejected key. Re-export using PKCS#8; ship fix. No data loss because the in-vault key is still correct. |

---

## Pitfall-to-Phase Mapping

This is the canonical mapping for downstream phase-planning agents. Each phase plan must include the corresponding verifications.

| # | Pitfall | Prevention Phase | Verification |
|---|---------|------------------|--------------|
| 1 | AES-GCM nonce reuse | Crypto / AEAD (v0.1) | Unit test: 1M nonce collisions test; same-plaintext-twice produces different ciphertexts. |
| 2 | AAD construction errors | Crypto / AEAD (v0.1) | Golden-vector tests for AAD bytes; tamper tests (record swap, type swap, dek_id swap). v0.2 milestone exit criterion. |
| 3 | Argon2id parameter regression | KDF (v0.1) + Project Skeleton (lint) | Separate `DefaultParams` / `ParamsForTesting`; floor-check on unwrap; benchmark gate. |
| 4 | Master password heap residency | Vault Lifecycle (v0.1) + Documentation (v1.0) | Sentinel-string log scan test; README §Go memory limitations verbatim. |
| 5 | DEK retention after lock | Vault Lifecycle (v0.1), Auto-Lock (v0.2) | Pre/post-lock decrypt test; concurrent-decrypt-during-lock stress test. |
| 6 | Frontend caches decrypted records past lock | Vault Lifecycle (v0.1), Auto-Lock (v0.2) | Frontend test: render → lock event → assert no decrypted strings in DOM. localStorage scan. |
| 7 | SQLite plaintext leakage | Records / Encryption (v0.1) | Grep test in test suite; manual matrix at v0.5; PRD §17 #14 release gate. |
| 8 | golang-migrate dirty state | Storage / Migration (v0.1) | Fixture-vault `dirty=1` test → `MIGRATION_FAILED`. |
| 9 | AAD coupled to schema_version | Crypto / AEAD (v0.1) | Code search: `grep schema_version internal/crypto/`; migration-without-re-encryption test. |
| 10 | crypto/rand error swallowing | Crypto / AEAD (v0.1) + Project Skeleton (errcheck CI) | errcheck lint at CI; stub `rand.Reader` to fail; assert `INTERNAL_ERROR`. |
| 11 | math/rand sneaks into security path | Project Skeleton (lint rule) | Lint rule: forbid `math/rand` import; helper centralization in `internal/crypto/rng.go`. |
| 12 | Unlock oracle leakage | Vault Lifecycle (v0.1), tightened v0.2 | Same-message test for wrong-password / corrupt-MAC / tampered-AAD. Timing-noise test. |
| 13 | Wails binding type drift | Project Skeleton (v0.1) | CI step: `wails generate module` produces no diff. tsconfig strict. |
| 14 | Cross-platform path handling | Storage / Migration (v0.1), Cross-Platform (v0.5), Key Export (v0.4) | Manual matrix per PRD §14.3; lint forbids `import "path"` in fs contexts. |
| 15 | macOS code-signing friction | Cross-Platform (v0.5), Documentation (v0.5–v1.0) | Non-author launches built artifact on clean macOS and Windows. |
| 16 | Idle timer skewed by sleep | Auto-Lock (v0.2) — polled; sleep events post-v0.5 | Wall-clock-skew test; manual sleep/wake test in v0.5 matrix. |
| 17 | PKCS#8 vs PKCS#1 confusion | Key Export (v0.4) | Round-trip test via `x509.ParsePKCS8PrivateKey`; PEM header assertion. |
| 18 | OpenSSH private key format | Key Export (v0.4) or post-v1.0 | Round-trip via `ssh.ParsePrivateKey` if implemented; otherwise README states deferred. |
| 19 | Public key fingerprint format | Key Generation (v0.4) | Single helper; matches `ssh-keygen -l -E sha256`. |
| 20 | Clipboard clear silently fails | Clipboard (v0.1), Documentation (v0.5–v1.0) | UI strings reviewed for "attempted" wording; README clipboard limitations verbatim. |
| 21 | JSON blob schema drift | Records / Encryption (v0.1), Expanded Records (v0.3) | Decode v0.1 blobs with v0.3 struct → succeeds; convention documented. |
| 22 | Recent-vaults UX subverts lock | Vault Lifecycle (v0.1), every milestone | Welcome screen has no Recent section; settings table has no path-recall key. |
| 23 | zxcvbn library selection | Password Generator (v0.1) | Use `@zxcvbn-ts/core` (or current maintained variant); pin version; advisory only. |

---

## Sources

- **PRD v1.2 (Architecture-Locked MVP Build Contract)** — `docs/PRD.md`, especially §4 Threat Model, §5 Cryptographic Design, §6 SQLite Storage, §10 Wails Binding Contract, §12 Errors and Logging, §13 Auto-Lock, §14 Testing, §17 Definition of Done, §18 Hard Rules, §19 README Requirements, §20 Final Implementation Sequence. (Authoritative for this project.)
- **NIST SP 800-38D** — Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC. Authoritative source for nonce-uniqueness requirements and the 96-bit random-nonce birthday bound.
- **RFC 9106** — Argon2 Memory-Hard Function for Password Hashing and Proof-of-Work Applications. Source for parameter recommendations matching PRD §5.3.
- **OpenSSH `PROTOCOL.key`** — OpenSSH private key file format documentation (in OpenSSH source tree). Reference for Pitfall 18.
- **Go stdlib documentation:**
  - `crypto/rand` — error semantics for `Read`.
  - `crypto/cipher.AEAD` — nonce reuse warning.
  - `crypto/x509.MarshalPKCS8PrivateKey` — preferred private key marshaling.
  - `golang.org/x/crypto/argon2` — `IDKey` parameters.
  - `golang.org/x/crypto/ssh` — public key marshaling and (recent versions) `MarshalPrivateKey`.
- **golang-migrate documentation** — dirty state semantics and `iofs` driver for `go:embed`.
- **Wails v2 documentation** — bindings, events, runtime APIs (https://wails.io/docs/).
- **zxcvbn-ts** — `@zxcvbn-ts/core` package documentation (current maintained TypeScript port).
- **General domain wisdom** — common patterns from the password-manager domain (KeePass, Bitwarden, 1Password): envelope encryption, AAD on records, no-recovery model, generic unlock errors. Confidence: HIGH for crypto patterns; MEDIUM for UX patterns (vary by product).

---

*Pitfalls research for: Abyss — local-first desktop password & key vault*
*Researched: 2026-04-29*
*Author: GSD project researcher (pitfalls dimension)*
