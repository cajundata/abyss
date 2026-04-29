# Phase 1: v0.1 Dogfoodable Vault - Context

**Gathered:** 2026-04-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 1 delivers the first runnable encrypted local vault on the primary dev OS, plus a Windows verification pass at phase end. Concretely: vault create / unlock / manual lock; credential record CRUD with masking, reveal, copy, search, and encrypted tags; password generation with clipboard countdown; and the security-critical scaffolding that cannot be retrofitted later — AAD helper, AppError DTO, sanitized logger, session/lock-state machine, fail-closed migrations, frontend api wrapper, and CI lint gates.

61 requirements span this phase (VAULT, REC, PASS, CLIP, API, UI, SEC, STORE families per `.planning/REQUIREMENTS.md`). The full list is locked in `.planning/ROADMAP.md` Phase 1 entry — this CONTEXT does not duplicate it; downstream agents read REQUIREMENTS + ROADMAP directly.

**What this phase does NOT include** (deferred per ROADMAP):
- Auto-lock + lock-on-sleep (Phase 2 / Phase 5)
- Change master password (Phase 2)
- AAD-tampering exit-criterion tests (Phase 2 — but the AAD itself ships here)
- API key / secure note / symmetric key / asymmetric key pair record types (Phase 3 / Phase 4)
- AES / RSA / Ed25519 generators and key export (Phase 4)
- Settings screen UI (Phase 6 — but a per-copy clipboard-timeout menu and last-vault-path memory ship here, see decisions below)
- Password strength meter / zxcvbn (Phase 6)
- Cross-platform manual matrix's full 22-scenario pass (Phase 5 / Phase 6 — Phase 1 ships a smaller "vault-lifecycle on both OSes" smoke)

</domain>

<decisions>
## Implementation Decisions

### Plan Decomposition

- **D-01: Five layered plans, foundation-first.** Phase 1 splits into 5 plans executed in order:
  1. **Storage + migrations** — `modernc.org/sqlite` driver wired into `database/sql`; `golang-migrate/v4` with `iofs` source + embed.FS; first migration creates `vault_metadata`, `deks`, `key_wrappings`, `records`, `settings` per PRD §6.3; `application_id = 0x4E56544E` set; `PRAGMA journal_mode=DELETE` + `foreign_keys=ON` confirmed via DSN; fail-closed-on-dirty migration.
  2. **Crypto + scaffolding** — `internal/aad/Build()` with golden-vector tests (AAD strings exact per PRD §5.4 — record metadata, record secret, wrapped DEK; AAD MUST NOT include `schema_version`); AES-256-GCM encrypt/decrypt helpers; Argon2id KEK derivation with profile + per-wrapping params + salt; DEK random via `crypto/rand`; `internal/apperr/{AppError,Code,Message,Cause,ToDTO()}`; `internal/session/{RequireUnlocked, Lock, Zero}`; `internal/logger/` slog wrapper with redaction list (PRD §12.2). The AAD helper MUST be invoked in the FIRST `cipher.Seal` call this codebase ever makes — there is no "ship now, AAD later" path.
  3. **Wails API surface + frontend api wrapper + UI shell** — Wails methods per PRD §10 scoped to Phase 1: `CreateVault`, `OpenVault`, `UnlockVault`, `LockVault`, `GetVaultStatus`, `ListRecords`, `GetRecord`, `CreateRecord`, `UpdateRecord`, `DeleteRecord`, `SearchRecords`, `GeneratePassword`, `CopyToClipboard`, `ClearClipboard`. `AppErrorCode` enum + `ToDTO()` everywhere. `frontend/src/api/` typed wrapper layer with `normalizeError` (no string-matching errors per PRD §18 hard rule #14). UI shell with router + 6 Phase-1 screens scaffolded (Welcome / Create Vault / Unlock / Vault Home / Record Detail / Password Generator) using Mantine v8.3.18 + `@mantine/core/styles.css` + `<MantineProvider defaultColorScheme="auto">`.
  4. **Credential CRUD + Password Generator + Clipboard countdown** — credential record encrypt/decrypt round-trip via the AAD helper; metadata blob JSON marshal/unmarshal that tolerates forward/backward compatibility (Pitfall 21: future fields must be `omitempty`); encrypted tags inside metadata blob (REC-13 promoted to Phase 1); masking-by-default + temporary reveal + per-field copy; in-memory search post-unlock; `crypto/rand` password generator (length 24 default, classes + exclude-ambiguous); clipboard countdown component (UI-11) with "Clear now" button (CLIP-04 promoted) and per-copy timeout menu (see D-13).
  5. **CI gates polish + Windows smoke** — any CI gates not landed in plans 2-4 (catch-all); cross-OS smoke pass (see D-09).
- **D-02: CI gates land WITH the code they protect, not at the end.** Specifically:
  - `forbidigo` ban on `math/rand` lands in plan 2 alongside the first crypto code.
  - `wails generate module` no-drift check lands in plan 3 alongside the first DTO definitions.
  - SQLite no-plaintext grep test (PRD §14.3 Hard Rule equivalent — title / username / password / notes / tags / URL never in plaintext columns) lands in plan 4 alongside the first record encrypt.
  - `errcheck` and `golangci-lint` baseline lands in plan 1 (project scaffold).
  - Plan 5 is a catch-all for any gate that couldn't land earlier.
- **D-03: API surface in Phase 1 is credential-only.** Per ROADMAP notes: ship `CreateRecord`/`GetRecord`/`UpdateRecord`/`DeleteRecord`/`ListRecords`/`SearchRecords` now, but only `record_type = "credential"` is implemented. Phase 3 extends to `api_key`/`secure_note`/`symmetric_key`; Phase 4 extends to `asymmetric_key_pair`. The DTO surface itself is final in Phase 1; later phases add discriminated union arms in the metadata blob, not new methods.

### Cross-Platform Discipline

- **D-04: Both macOS and Windows are available; macOS is iteration-primary.** Daily `wails dev` runs on macOS. The user has both machines, so dual-platform issues surface continuously rather than at phase boundaries.
- **D-05: Windows verifies at the end of EACH plan, not just at the end of the phase.** When a plan completes on macOS, `git pull` on Windows, `wails dev`, exercise the new feature, sign off. Catches Windows path / clipboard / Argon2 perf regressions at plan boundaries — cheaper to fix than at phase end.
- **D-06: SQLite driver is `modernc.org/sqlite` (pure Go).** Locked here because dual-machine workflow above only works without CGO. Researcher MUST verify in plan 1: after wiring, run `go mod why github.com/mattn/go-sqlite3` and confirm CGO has not crept in via a transitive dep. If it has, fail the plan.
- **D-07: golang-migrate uses the pure-Go-compatible SQLite database driver.** Open research question for plan 1: confirm exact import path — `github.com/golang-migrate/migrate/v4/database/sqlite` (without "3", pure-Go-compatible) vs `database/sqlite3` (mattn/CGO). Verify DSN params that set `journal_mode=DELETE` and `foreign_keys=ON`; document the chosen DSN format in the plan.
- **D-08: Mantine v8.3.18 over v9.** Locked per `CLAUDE.md` §1a. Re-evaluate at v1.0 milestone boundary, not mid-Phase. Mid-phase upgrade to v9 is forbidden.
- **D-09: End-of-phase Windows smoke is a full vault-lifecycle pass on both OSes.** Manual scenario: create vault → set master pw → unlock → create + view + edit + delete a credential → generate password → copy with countdown → manual lock → re-unlock. Run on macOS AND Windows from a fresh build of each. Result documented in `01-VERIFICATION.md` (Phase 5 will run the full PRD §14.3 22-scenario matrix; Phase 1's smoke is a strict subset).

### First-Launch & Vault File Handling

- **D-10: Fresh install shows Welcome with Create button auto-focused.** Three-button surface per PRD §9.2 (Create new vault / Open existing vault / View security limitations); on detect-no-vault-ever-opened, focus moves to Create so Enter creates. Preserves the documented 10-screen surface and the "I copied a vault file from another machine" workflow.
- **D-11: Subsequent launches remember the last-opened vault path and route to Unlock prefilled.** "Last vault path" is a single string stored OUTSIDE the encrypted vault (the vault is locked at launch, so its `settings` table is unreachable). Suggested location: `~/Library/Application Support/Abyss/config.json` (macOS) and `%APPDATA%\Abyss\config.json` (Windows) — researcher to confirm canonical Wails/cross-platform pattern. PRD §9.2 defers a recent-vaults LIST; a single-last-path is not in the deferred set.
- **D-12: Default vault folder + extension are opinionated but overridable.** On Create, the OS file dialog opens at `~/Documents/Abyss/` (macOS) or `%USERPROFILE%\Documents\Abyss\` (Windows), default filename `vault.abyssvault`. User can pick any folder/name. Matches PRD's "users zip / move / back-up the file" portability thesis. The `.abyssvault` extension is informational only — `application_id = 0x4E56544E` is the canonical identity check at every Open per PRD §6.1, and that check stays authoritative.
- **D-13: Manual lock routes to Unlock screen with vault path prefilled.** Sequence on lock: in-memory DEK zeroed via `crypto.Zero` then nilled, decrypted record state cleared, clipboard clear attempted, frontend stores wiped via `vault:locked` event, then router pushes Unlock screen with the same vault path filled in. User just retypes master password.

### Phase 1 Settings Surface (No Settings Screen Until Phase 6)

- **D-14: Clipboard timeout in v0.1 = per-copy override menu.** Default 30s. The clipboard countdown component (UI-11) supports two copy paths: a primary "Copy" button (uses 30s) and a `Copy with timeout...` menu (lets user pick 10 / 30 / 60 / 120s for THIS copy). No persisted user preference in v0.1. Phase 6 Settings will add a persisted default-timeout that overrides 30s. The `CopyToClipboard(req)` Wails method per PRD §10.6 already accepts `timeoutMs` — the per-copy override just lets the frontend pass different values; the contract doesn't change between Phase 1 and Phase 6.
- **D-15: Theme = `defaultColorScheme="auto"`, no toggle in v0.1.** Phase 1 wraps `<App />` in `<MantineProvider defaultColorScheme="auto">`. macOS users get dark mode automatically when system is dark; Windows likewise. Explicit theme picker waits for Phase 6 Settings.
- **D-16: Master password feedback in v0.1 = length-only validator.** A counter (`{n} / 12 minimum`) and the PRD-locked ≥12-char rule. NO color-coded strength bar, NO zxcvbn. PASS-07 (zxcvbn integration) ships whole in Phase 6 to avoid shipping then replacing a meter mid-project. This is a strict scope match with REQUIREMENTS.md.

### Tags, Search, and Reveal/Copy Ergonomics (Defer to /gsd-ui-phase)

- **D-17: REC-13 (encrypted tags) ships in Phase 1 inside the metadata blob.** Already promoted from PRD §15 v0.3 to Phase 1 by ROADMAP notes; restated here for downstream clarity. Tag values live inside the already-encrypted `metadata_blob`, not plaintext columns. Tag UX (chip vs free-text input, autocomplete from prior tags, color coding) is a UI-fidelity question deferred to `/gsd-ui-phase` for Phase 1.
- **D-18: REC-09 (in-memory search) is post-unlock, scope and live-vs-explicit deferred to /gsd-ui-phase.** PRD locks "search records in memory after unlock" but not the field scope (title-only vs title+tags+username+URL+notes) or the input model (live debounced vs explicit Enter). Defer to UI phase.
- **D-19: Reveal-secret UX (hold-to-reveal vs click-to-toggle vs auto-hide-after-N-seconds), per-field copy button placement, edit-on-Detail vs separate Edit screen, and confirm-delete UX (modal vs type-to-confirm) are all deferred to /gsd-ui-phase.** PRD §9.5 / §9.6 describe the surface but don't pin the interaction model.

### Claude's Discretion

- **D-20: UUIDv4 via `github.com/google/uuid` for `vault_id`, `dek_id`, `record_id`, `wrapping_id`** — CLAUDE.md recommends this; PRD §6.3 leaves the TEXT primary-key generation up to implementation. Approving.
- **D-21: Test framework = stdlib `testing` + selective `testify/require` for setup preconditions and `testify/assert` for multi-field equality.** CLAUDE.md flagged this as a style call; approving the hybrid approach.
- **D-22: First crypto golden-vector tests use known-plaintext / known-key / known-nonce / known-AAD → known-ciphertext fixtures** committed in `internal/aad/testdata/` and `internal/crypto/testdata/`. AAD-tampering exit-criterion tests are Phase 2; Phase 1 ships only the positive-path golden vectors.
- **D-23: Per-record DEK ID is the active DEK at record creation time** (PRD §6.3 schema supports DEK rotation in v2; Phase 1 just records the FK). DEK rotation workflow is CRYPTO-V2-04, deferred.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project source-of-truth (highest priority)

- `docs/PRD.md` v1.2 (Architecture-Locked MVP Build Contract) — authoritative source. The PRD is the contract; ROADMAP, REQUIREMENTS, and CONTEXT are derivative artifacts. When this CONTEXT and the PRD conflict, the PRD wins. Specifically critical for Phase 1:
  - §2 (Locked Architecture Decisions)
  - §3.3 (out of scope)
  - §4 (threat model)
  - §5 (Cryptographic Surface — AES-256-GCM, Argon2id, AAD strings, KEK/DEK envelope)
  - §6 (SQLite shape — `application_id`, journal mode, 5 tables)
  - §7 (record schemas, Phase 1 implements §7.1 credential)
  - §8 (functional behaviors — vault lifecycle, records, password gen, clipboard, generators in later phases)
  - §9 (10 screens — §9.2 Welcome, §9.3 Create, §9.4 Unlock, §9.5 Vault Home, §9.6 Record Detail, §9.7 Clipboard countdown, §9.8 private-key warning Phase 4)
  - §10 (Wails method signatures + AppErrorCode enum)
  - §12 (sanitized logging policy + safe error messaging)
  - §14 (test requirements — §14.1 Go unit, §14.3 cross-OS matrix)
  - §15 (milestone breakdown — Phase 1 = v0.1)
  - §18 (Hard Rules — these override any ambiguity)
  - §19 (README requirements, Phase 6)
  - §20 (20-step implementation sequence)

### Project planning artifacts

- `.planning/PROJECT.md` — project overview, requirements, key decisions, constraints
- `.planning/REQUIREMENTS.md` — full v1 REQ-ID list + per-phase mapping (Phase 1 covers 61 of 115)
- `.planning/ROADMAP.md` — Phase 1 entry (lines 24-46) — phase goal, requirements, success criteria, scaffolding directive, open research questions
- `CLAUDE.md` — codebase + user instructions, including the "Already Locked by PRD — Do Not Re-Debate" table at top of Tech Stack section

### Project research artifacts

- `.planning/research/STACK.md` — version-pinned recommendations: Go 1.25.9 / Wails v2.12.0 / Node 22 LTS / TypeScript 5.9 / Vite 8 / React 19.2 / Mantine 8.3.18 / `modernc.org/sqlite` v1.36+ / `golang-migrate` v4. Includes installation commands, golang-migrate iofs wiring, Argon2id parameter handling, OpenSSH/PEM/PKCS#8 marshaling reference (Phase 4-relevant).
- `.planning/research/ARCHITECTURE.md` — service-layer architecture, idle-timer topology (Phase 2-relevant), event-emit pattern (`vault:locked` event used by D-13)
- `.planning/research/PITFALLS.md` — 23 catalogued pitfalls. Phase-1-relevant: Pitfall 17 (PEM block type confusion — Phase 4), Pitfall 19 (fingerprint format — Phase 4), Pitfall 21 (metadata-blob forward/backward compat — RELEVANT NOW for plan 4), Pitfall 22 (Argon2 param storage), and the "AAD must ship in first cipher.Seal" directive (RELEVANT NOW for plan 2).
- `.planning/research/FEATURES.md` — feature priority matrix; supports the deferred ideas list
- `.planning/research/SUMMARY.md` — research synthesis

### Open research questions to resolve in plan-phase research (verbatim from ROADMAP Phase 1 notes)

1. Confirm exact import path for the pure-Go SQLite database driver in `golang-migrate/v4` (`database/sqlite` vs `database/sqlite3`); after wiring, run `go mod why github.com/mattn/go-sqlite3` to confirm CGO has not crept in.
2. Verify DSN params that set `journal_mode=DELETE` and `foreign_keys=ON`.
3. Mantine v8.3.18 vs v9 — recommendation is v8.3.18 (D-08 confirms this, lock through Phase 6).
4. PASS-07 zxcvbn placement — currently mapped to Phase 6 (D-16 confirms; Phase 1 ships length-only validator).
5. Canonical cross-platform path for app-config file storing "last vault path" (D-11) — `~/Library/Application Support/Abyss/` vs `os.UserConfigDir()` vs Wails-specific helper. Researcher to confirm.

### Configuration and constraints

- `.planning/config.json` — workflow toggles
- `.planning/STATE.md` — current execution state

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

None — Phase 1 is the first build phase. The repo currently contains: `docs/PRD.md`, `.planning/` artifacts, `CLAUDE.md`, `README.md` (placeholder), `.gitignore`. No Go modules, no `frontend/`, no `wails.json` yet.

### Established Patterns

None in code. Patterns are established BY Phase 1 plans 1-2-3, specifically:
- `internal/aad/Build()` is the single source of truth for AAD strings
- `internal/apperr/AppError{Code,Message,Cause}.ToDTO()` is the single error-flow path to the Wails boundary
- `internal/session/RequireUnlocked()` is the single guard called by every secret-touching method
- `internal/logger/` is the single redaction-aware slog wrapper
- `frontend/src/api/` is the single typed wrapper layer; React components MUST NOT import generated Wails bindings directly
- `embed.FS` + `iofs` is the single migration delivery mechanism

These patterns ARE the contract — any deviation is a Pitfall risk.

### Integration Points

- Wails v2.12.0 scaffold (`wails init -n abyss -t react-ts`) creates `app.go` with method bindings, `frontend/` for React, `wails.json` for build config. Plan 1 must scaffold Wails BEFORE writing any Go code.
- macOS app bundle path / Windows .exe path conventions are baked into Wails's `wails build` output. Code-signing is deferred for MVP per CLAUDE.md §7 — ad-hoc-signed `.app` runs after right-click → Open on macOS; unsigned `.exe` triggers Windows SmartScreen.

</code_context>

<specifics>
## Specific Ideas

- The Welcome screen's third button is "View security limitations" (PRD §9.2), not a generic "About" button. This phrasing ships in Phase 1; the full About / Security Model screen (UI-10) is Phase 6. The Phase 1 button can either route to a minimal modal listing the three limitations from PRD §4.4 (no recovery, clipboard manager limitations, Go memory limitations) or be a stub that opens an alert dialog. UI fidelity choice deferred to `/gsd-ui-phase`, but the button MUST exist and MUST route somewhere usable in v0.1.
- The clipboard countdown component drives the progress bar from the frontend per CLAUDE.md §1d — backend `CopyToClipboard(req)` writes the secret, frontend `useInterval` from `@mantine/hooks` runs the countdown, frontend calls `ClearClipboard()` at zero. The "Clear now" button (CLIP-04) is also a frontend-driven call to `ClearClipboard()`. Backend never owns the countdown timer.
- Master password input field on Create / Unlock / (later) Change Master MUST mask by default and SHOULD support a reveal toggle (eye icon). PRD §9.3 / §9.4 describe the screens but don't pin reveal toggle. Mantine has `<PasswordInput>` with built-in visibility toggle — recommend using it. Decision: use Mantine's default behavior for `<PasswordInput>`. Note for `/gsd-ui-phase`: confirm visibility-toggle is enabled in v0.1 (it is by default).

</specifics>

<deferred>
## Deferred Ideas

These came up during analysis but are out of scope for Phase 1. Don't lose them.

- **Vault file integrity check beyond `application_id`** — A row-level checksum or signed manifest could detect bit-rot or tampering at vault open. PRD only mandates `application_id` validation. Possible candidate for v1.x; track in PROJECT.md after v1.0 ships.
- **Two-instance file-lock behavior** — What happens if the user opens the same `.abyssvault` from two Abyss instances simultaneously? SQLite rollback journal will serialize, but the second instance's UI will show a stale view. Worth a Phase 2 or Phase 5 conversation. Currently no requirement covers this; not a blocker for v0.1.
- **Behavior when the remembered vault path no longer exists at launch** — D-11 stores the last path. If the file was moved/deleted, the Unlock screen needs a graceful fallback (route to Welcome with an info banner). Phase 1 implementation should include this fallback; the gray area is the COPY ("This vault is no longer at this path. Pick a different one.") — defer copy/UI to `/gsd-ui-phase`.
- **Edit-on-Detail vs separate Edit screen for credentials** — UI-fidelity question for `/gsd-ui-phase`.
- **Confirm-delete UX (modal with `Delete` button vs type-to-confirm)** — UI-fidelity question for `/gsd-ui-phase`. PRD only mandates "explicit confirmation" (REC-08).
- **Per-secret tag autocomplete from previously-used tags** — UI-fidelity question for `/gsd-ui-phase`. The encrypted tag storage (D-17) supports it; the input UX does not yet.
- **Recent vaults list on Welcome** — explicitly deferred by PRD §9.2 (REC-V2-01). D-11 ships a single-last-path; the LIST stays deferred to v1.x.
- **Theme picker, persisted clipboard-timeout default, persisted-search-state, last-record-type-filter persistence** — all deferred to Phase 6 Settings (UI-09).
- **Lock-on-system-sleep / lid-close (VAULT-13)** — deferred to Phase 5 per ROADMAP.
- **Auto-lock idle timer (VAULT-08)** — deferred to Phase 2 per ROADMAP. Phase 1 ships only manual lock.
- **AAD-tampering exit-criterion tests (cross-record swap, record_type swap, dek_id swap)** — deferred to Phase 2 per ROADMAP. Phase 1 ships positive-path golden vectors only.

</deferred>

---

*Phase: 1-v0-1-dogfoodable-vault*
*Context gathered: 2026-04-29*
