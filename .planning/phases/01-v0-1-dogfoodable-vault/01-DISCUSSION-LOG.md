# Phase 1: v0.1 Dogfoodable Vault - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-29
**Phase:** 1-v0-1-dogfoodable-vault
**Areas discussed:** Plan decomposition shape, Dev OS + Windows smoke, First-launch & vault file handling, Phase 1 settings surface

---

## Plan decomposition shape

### Q: Given foundation-first is required, how would you like Phase 1 broken into plans?

| Option | Description | Selected |
|--------|-------------|----------|
| 5 plans, layered (Recommended) | (1) Storage+migrations+app_id, (2) Crypto+AAD+session+apperr+logger, (3) Wails API + frontend api wrapper + UI shell, (4) Credential CRUD + masking/reveal/copy/search/tags + password generator + clipboard countdown, (5) CI gates + Windows smoke. Plan 4 can run after 3 lands; CI gates land alongside earlier plans. | ✓ |
| 3 chunky plans | (1) Backend foundation (storage + crypto + AAD + session + apperr + logger + Wails API), (2) Frontend (api wrapper + UI shell + credential CRUD + pw gen + clipboard), (3) CI gates + Windows smoke. Fewer review checkpoints; each plan is bigger. | |
| One mega-plan | Single PLAN.md walks all 61 reqs top-to-bottom in dependency order. Simplest planning artifact, but very long single review cycle and no natural commit boundaries. | |
| 7-10 small plans | Tightest decomposition: each plan delivers ~6-9 reqs. Most review checkpoints (good for catching scaffolding mistakes early), but more orchestration overhead. | |

**User's choice:** 5 plans, layered (Recommended)
**Notes:** —

### Q: When should CI gates (math/rand ban, no-plaintext grep, wails-bindings drift, errcheck) land?

| Option | Description | Selected |
|--------|-------------|----------|
| With the code they protect (Recommended) | Each gate lands in the same plan as the code it guards: `forbidigo math/rand` ban lands in plan 2 (Crypto), `wails generate module` check lands in plan 3 (API), SQLite no-plaintext grep lands in plan 4 (Records). Plan 5 only adds Windows smoke + any remaining gates. Catches scaffolding mistakes earliest — most resilient. | ✓ |
| All gates in plan 5 | Plans 1-4 don't worry about CI; plan 5 sweeps them all in. Simpler dependency graph, but a CI gate added late may surface violations that need going back to fix. | |
| All gates land in plan 1, before any other code | Set up `golangci-lint` + `forbidigo math/rand` + `errcheck` at scaffold time. The grep test + bindings drift check land later. Most paranoid — lints fire from the very first PR. Some gates can't run yet (no SQLite file to grep). | |

**User's choice:** With the code they protect (Recommended)
**Notes:** —

---

## Dev OS + Windows smoke

### Q: Which OS will you be developing on day-to-day for Phase 1?

| Option | Description | Selected |
|--------|-------------|----------|
| macOS only | Daily dev on macOS; Windows is a smoke-test target only. | |
| Windows only | Daily dev on Windows; macOS is a smoke-test target only. | |
| Both — alternate / dual machine | You have both available and switch between them. | ✓ |

**User's choice:** Both — alternate / dual machine
**Notes:** —

### Q: When you alternate, which OS is the iteration primary (daily `wails dev`) vs verification target (build/run once per plan)?

| Option | Description | Selected |
|--------|-------------|----------|
| macOS primary, Windows verify-each-plan (Recommended) | Daily `wails dev` and tight loop on macOS. At the end of each plan, switch to Windows, `git pull`, `wails dev`, exercise the new feature, sign off. Catches Windows path/clipboard/Argon2-perf bugs at plan boundaries. | ✓ |
| Windows primary, macOS verify-each-plan | Mirror of above, with Windows as the iteration env. | |
| Truly co-equal — work on whichever you're at | No primary/secondary; both OSes get continuous use. Doubles `wails dev` setup time. | |

**User's choice:** macOS primary, Windows verify-each-plan (Recommended)
**Notes:** —

### Q: End-of-phase Windows smoke: what's the formal pass for Phase 1?

| Option | Description | Selected |
|--------|-------------|----------|
| Full vault lifecycle on both (Recommended) | Manual: create a vault, set master pw, unlock, create+view+edit+delete a credential, generate password, copy with countdown, lock vault, re-unlock — on macOS AND Windows, fresh build of each. | ✓ |
| Build-only smoke on Windows | `wails build -platform windows/amd64` succeeds + the .exe launches and shows the Welcome screen. No CRUD verification. | |
| Per-plan smoke + final summary | Each plan logs its own per-OS smoke result; end of Phase 1 just aggregates them. | |

**User's choice:** Full vault lifecycle on both (Recommended)
**Notes:** —

---

## First-launch & vault file handling

### Q: Fresh install — user launches Abyss for the first time, no vault file anywhere on disk. What do they see?

| Option | Description | Selected |
|--------|-------------|----------|
| Welcome screen, Create auto-focused (Recommended) | Per PRD §9.2: three buttons (Create / Open / View security limitations). On fresh install, Create is auto-focused so Enter creates a vault. Preserves documented 10-screen surface and supports the 'I copied a vault file from another machine' workflow. | ✓ |
| Auto-route to Create (skip Welcome) | Detect 'no vault has ever been opened' and jump straight to Create Vault. One less screen on day one. But user can't easily browse to a copied-from-another-machine vault. | |
| Welcome with no auto-focus | Strict PRD §9.2 — user clicks deliberately. Slightly more keystrokes; emphasizes the 'View security limitations' option. | |

**User's choice:** Welcome screen, Create auto-focused (Recommended)
**Notes:** —

### Q: On subsequent launches when a vault was opened previously, what should happen?

| Option | Description | Selected |
|--------|-------------|----------|
| Remember last-vault path → Unlock screen prefilled (Recommended) | Store last vault path in a non-encrypted app-config file (vault is locked at launch). Jump straight to Unlock with that path prefilled — user just types master password. | ✓ |
| Always show Welcome — no path memory | Strict 'no remembered state across launches' posture. Higher friction. | |
| Welcome with last path prefilled in 'Open existing' button | Hybrid: still see Welcome, but 'Open existing' shows the last vault filename. Slightly more visible state. | |

**User's choice:** Remember last-vault path → Unlock screen prefilled (Recommended)
**Notes:** —

### Q: Default vault folder + file extension — how opinionated should Abyss be?

| Option | Description | Selected |
|--------|-------------|----------|
| Suggest default folder + `.abyssvault` extension; user can override (Recommended) | File dialog opens at `~/Documents/Abyss/` (macOS) / `%USERPROFILE%\Documents\Abyss\` (Windows), default filename `vault.abyssvault`. Discoverable, portable, identifies the file alongside `application_id` validation. | ✓ |
| User picks anything — no default folder, no default extension | Maximally neutral; lowest opinion. Worse discoverability for non-technical users. | |
| Hidden app-data folder, file location abstracted | App stores under `~/Library/Application Support/Abyss/vault.abyssvault`. Conflicts with PRD's portability/zip-and-move story. | |

**User's choice:** Suggest default folder + `.abyssvault` extension; user can override (Recommended)
**Notes:** —

### Q: When the user manually locks the vault from the Vault Home screen, where should they land?

| Option | Description | Selected |
|--------|-------------|----------|
| Unlock screen with vault path prefilled (Recommended) | After lock: in-memory DEK / decrypted records / clipboard cleared, then UI routes to Unlock screen with the same vault path filled in. | ✓ |
| Welcome screen | After lock, return to 3-button Welcome. User must re-pick 'Open existing' (path still remembered, so one extra click). | |
| Same screen, faded/disabled with a 'Locked' overlay and an Unlock button | Vault Home stays mounted but visually obscured + every action throws `VAULT_LOCKED`. Risk of leaking state if the disabled overlay is fragile. | |

**User's choice:** Unlock screen with vault path prefilled (Recommended)
**Notes:** —

---

## Phase 1 settings surface

### Q: Clipboard timeout in Phase 1 — CLIP-03 mandates the timeout is configurable (10/30/60/120s, default 30s). How should the user choose it before Settings UI ships?

| Option | Description | Selected |
|--------|-------------|----------|
| Inline selector on the countdown bar (Recommended) | Clipboard countdown component (UI-11) gets a small dropdown: 10s/30s/60s/120s. Persists in non-sensitive `settings` table. | |
| Hardcoded 30s in Phase 1; configurability arrives with Settings in Phase 6 | 30s is the only choice in v0.1; CLIP-03 'configurable' is technically deferred. | |
| Tiny Phase-1-only Settings drawer/modal | Throwaway UI superseded by full Settings in Phase 6. | |
| Per-copy override (`Copy with timeout...` menu) | Default 30s; user can context-menu a secret to pick 10/30/60/120 just for that copy. No persistence. | ✓ |

**User's choice:** Per-copy override (`Copy with timeout...` menu)
**Notes:** Implication captured in CONTEXT D-14 — no persisted user preference in v0.1; Phase 6 Settings will introduce a persisted default. The existing `CopyToClipboard(req)` Wails contract already accepts `timeoutMs`, so the per-copy override is a frontend-only behavior change.

### Q: Theme in Phase 1 — Mantine has built-in dark/light/auto. What ships in v0.1?

| Option | Description | Selected |
|--------|-------------|----------|
| Auto only (follows OS) (Recommended) | `<MantineProvider defaultColorScheme="auto">` and no toggle. Theme picker arrives in Phase 6 Settings. | ✓ |
| Header toggle (sun/moon icon) | Persists in non-sensitive `settings` table. ~1 component. | |
| Light only | Force light mode. May feel jarring on dark-mode systems. | |

**User's choice:** Auto only (follows OS) (Recommended)
**Notes:** —

### Q: Master password strength feedback in Phase 1 — PRD PASS-07 (zxcvbn) ships in Phase 6. What does the user see during Create Vault / Change Master in v0.1?

| Option | Description | Selected |
|--------|-------------|----------|
| Length validator only — no strength meter (Recommended) | Just the PRD-locked '≥12 chars' rule + a counter. No color-coded bar, no zxcvbn. zxcvbn arrives whole in Phase 6. | ✓ |
| Basic length-tier visual feedback | Simple 3-tier color bar (red <12, yellow 12-15, green ≥16). NOT zxcvbn. Misleading (length ≠ strength). | |
| Ship a tiny zxcvbn integration in Phase 1 | Pulls Phase 6 PASS-07 forward. | |

**User's choice:** Length validator only — no strength meter (Recommended)
**Notes:** Note that REQUIREMENTS shows VAULT-09 (Change Master Password) as Phase 2, not Phase 1 — so the length-only validator only ships on the Create Vault screen in Phase 1.

---

## More-or-next checks

| Q | A |
|---|---|
| More questions about first-launch / vault file handling, or move to next? | Next area |
| We've discussed all four selected areas. Explore more, or write context? | I'm ready for context |

---

## Claude's Discretion

Captured as decisions D-20, D-21, D-22, D-23 in CONTEXT.md:
- UUIDv4 via `github.com/google/uuid` for IDs
- stdlib `testing` + selective `testify/require` and `testify/assert`
- AAD/crypto golden vectors as `testdata/` fixtures (positive-path only in Phase 1; AAD-tampering tests are Phase 2)
- Per-record DEK ID = active DEK at record creation time (DEK rotation is v2)

## Deferred Ideas

Captured in CONTEXT.md `<deferred>` section:
- Vault file integrity check beyond `application_id`
- Two-instance file-lock behavior
- Behavior when remembered vault path no longer exists at launch (the fallback ships in Phase 1; the COPY/UI is deferred to /gsd-ui-phase)
- Edit-on-Detail vs separate Edit screen
- Confirm-delete UX (modal vs type-to-confirm)
- Per-secret tag autocomplete
- Recent vaults list (deferred per PRD §9.2 / REC-V2-01)
- Theme picker, persisted clipboard-timeout default, persisted-search-state, last-record-type-filter (Phase 6 Settings)
- Lock-on-system-sleep / lid-close (Phase 5 / VAULT-13)
- Auto-lock idle timer (Phase 2 / VAULT-08)
- AAD-tampering exit-criterion tests (Phase 2)
