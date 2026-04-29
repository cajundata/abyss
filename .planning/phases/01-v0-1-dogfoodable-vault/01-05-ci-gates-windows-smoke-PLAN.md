---
phase: 01-v0-1-dogfoodable-vault
plan: 05
type: execute
wave: 5
depends_on: ["01-04-credential-crud-passgen-clipboard-PLAN.md"]
files_modified:
  - .github/workflows/ci.yml
  - .planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md
  - scripts/audit-ci-gates.sh
  - README.md
autonomous: false
requirements_addressed:
  - SEC-09
  - VAULT-04

must_haves:
  truths:
    - "All five Phase 1 CI gates from D-02 are committed and active in `.github/workflows/ci.yml`: (1) errcheck + golangci-lint baseline (plan 01), (2) `mattn/go-sqlite3` absent grep gate on go.sum (plan 01), (3) forbidigo banning math/rand (plan 02), (4) wails generate module no-drift (plan 03), (5) SQLite no-plaintext grep gate (plan 04). Plus the api-wrapper boundary check (plan 03)."
    - "An audit script `scripts/audit-ci-gates.sh` enumerates the 6 gates and verifies each is committed; CI runs this script weekly and on every PR to catch silent gate disablement."
    - "Cross-OS lifecycle smoke per CONTEXT D-09 passes on both macOS AND Windows from fresh release builds (NOT `wails dev`): create vault → set master pw → unlock → create credential → view (masked) → reveal → edit → delete → generate password → copy with countdown → manual lock → re-unlock. Result documented in 01-VERIFICATION.md with build-platform output, commit SHA, and date for each OS."
    - "`wails build -platform windows/amd64` from a macOS host produces a runnable .exe (pure-Go SQLite + zero-CGO toolchain promise from D-06 verified)."
    - "Windows smoke uses the unsigned `.exe` (SmartScreen warning is acceptable for v0.1 per CLAUDE.md §7); macOS uses an ad-hoc-signed `.app` (`codesign --force --deep --sign - ./build/bin/abyss.app`). README documents both."
  artifacts:
    - path: ".planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md"
      provides: "Full Phase 1 verification log: per-plan D-05 sign-offs (plans 01-04), final D-09 dual-OS smoke, audit-ci-gates.sh output, commit SHAs"
    - path: "scripts/audit-ci-gates.sh"
      provides: "Enumerates all 6 Phase 1 CI gates and asserts each is committed"
    - path: "README.md"
      provides: "Phase-1 minimal README: project purpose, macOS + Windows dev/build/run instructions, ad-hoc-sign + SmartScreen notes; FULL threat-model README is Phase 6 (DOC-01..DOC-05)"
  key_links:
    - from: "scripts/audit-ci-gates.sh"
      to: ".github/workflows/ci.yml + .golangci.yml + scripts/check-wails-bindings.sh + scripts/check-no-plaintext.sh"
      via: "asserts each gate's marker string is present in its config file"
      pattern: "audit-ci-gates"
    - from: ".planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md"
      to: "All 4 prior plans' SUMMARY.md files"
      via: "Final phase verification consolidates D-05 plan-by-plan signoffs and adds D-09 phase-end dual-OS smoke"
      pattern: "01-VERIFICATION.md"
---

<objective>
Close out Phase 1: audit that every CI gate from CONTEXT D-02 actually shipped (no silent disablement), perform the full D-09 dual-OS lifecycle smoke from fresh release builds (not `wails dev`), produce the Phase 1 minimal README (full README is DOC-01..DOC-05 in Phase 6), and document everything in `01-VERIFICATION.md`. This plan is intentionally small — most of the heavy lifting was done in plans 01-04. Plan 5 is the gatekeeper that proves Phase 1 is shippable.

Purpose: Phase 1 establishes the cross-cutting scaffolding that cannot be retrofitted (CONTEXT D-02 + STATE.md decisions). If any of the 6 CI gates was silently dropped during plans 01-04, Phase 2 builds on a weaker floor; this plan catches it. The D-09 dual-OS smoke verifies that the macOS-iteration-primary / Windows-verifies-each-plan loop didn't drift apart — at the end of Phase 1 we need a fresh-build pass on both OSes from a single commit, not just `wails dev` rehearsals.

Output: A signed verification document (`01-VERIFICATION.md`) declaring Phase 1 complete, with audit output proving the CI floor is intact and dual-OS smoke results from release builds.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@docs/PRD.md
@CLAUDE.md
@.planning/STATE.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-01-storage-migrations-PLAN.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-02-crypto-scaffolding-PLAN.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-03-wails-api-frontend-shell-PLAN.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-04-credential-crud-passgen-clipboard-PLAN.md

<interfaces>
<!-- This plan PRODUCES no Go/TS interfaces. It produces the verification artifact + audit script + Phase 1 README. -->
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Build the CI-gate audit script + extend .github/workflows/ci.yml to run it on every PR</name>
  <files>scripts/audit-ci-gates.sh, .github/workflows/ci.yml</files>
  <read_first>
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-02 (the 5 gates that should have landed in plans 01-04, plus the api-wrapper boundary gate from plan 03)
    - .github/workflows/ci.yml (current state — every prior plan extended it; this plan reads it to identify which gates are present)
    - .golangci.yml (errcheck + forbidigo presence)
    - scripts/check-wails-bindings.sh (no-drift gate from plan 03)
    - scripts/check-no-plaintext.sh (no-plaintext gate from plan 04)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-01-storage-migrations-PLAN.md (acceptance criteria for plan 01 gates)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-02-crypto-scaffolding-PLAN.md (acceptance criteria for plan 02 gates)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-03-wails-api-frontend-shell-PLAN.md (acceptance criteria for plan 03 gates)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-04-credential-crud-passgen-clipboard-PLAN.md (acceptance criteria for plan 04 gates)
  </read_first>
  <action>
    1. **`scripts/audit-ci-gates.sh`** — single script that asserts each of the 6 Phase 1 CI gates is present in its config. Each check is a literal grep on the canonical marker string. The script exits 0 if all 6 pass; otherwise it prints which gate failed and exits non-zero.

       ```bash
       #!/usr/bin/env bash
       # Phase 1 CI gate audit (CONTEXT D-02). Catches silent disablement.
       set -euo pipefail
       cd "$(dirname "$0")/.."

       FAIL=0
       check() {
         local name="$1" file="$2" pattern="$3"
         if ! grep -qE "$pattern" "$file"; then
           echo "FAIL: gate '$name' missing from $file (expected pattern: $pattern)"
           FAIL=1
         else
           echo "OK:   gate '$name' present in $file"
         fi
       }

       # Gate 1 — plan 01: errcheck + golangci-lint baseline (.golangci.yml)
       check "errcheck baseline (plan 01)" ".golangci.yml" "errcheck"
       check "golangci-lint runs in CI (plan 01)" ".github/workflows/ci.yml" "golangci-lint"

       # Gate 2 — plan 01: mattn/go-sqlite3 absent in go.sum (D-06)
       check "no-mattn grep gate (plan 01)" ".github/workflows/ci.yml" "github\.com/mattn/go-sqlite3"
       # Defense in depth: also verify the gate is currently passing on this commit.
       if grep -q 'github.com/mattn/go-sqlite3' go.sum 2>/dev/null; then
         echo "FAIL: gate 'no-mattn' is not currently green — go.sum contains mattn/go-sqlite3"
         FAIL=1
       else
         echo "OK:   no-mattn gate green on this commit"
       fi

       # Gate 3 — plan 02: forbidigo banning math/rand
       check "forbidigo math/rand ban (plan 02)" ".golangci.yml" "forbidigo"
       check "math/rand pattern in forbidigo (plan 02)" ".golangci.yml" "math/rand"

       # Gate 4 — plan 03: wails generate module no-drift
       check "wails generate module no-drift (plan 03)" ".github/workflows/ci.yml" "wails generate module"
       if [ ! -x "scripts/check-wails-bindings.sh" ]; then
         echo "FAIL: scripts/check-wails-bindings.sh missing or not executable"
         FAIL=1
       else
         echo "OK:   scripts/check-wails-bindings.sh present and executable"
       fi

       # Gate 5 — plan 03: api-wrapper boundary check
       check "api-wrapper boundary check (plan 03)" ".github/workflows/ci.yml" "wailsjs/go/main/App"

       # Gate 6 — plan 04: SQLite no-plaintext grep gate
       check "SQLite no-plaintext gate (plan 04)" ".github/workflows/ci.yml" "check-no-plaintext"
       if [ ! -x "scripts/check-no-plaintext.sh" ]; then
         echo "FAIL: scripts/check-no-plaintext.sh missing or not executable"
         FAIL=1
       else
         echo "OK:   scripts/check-no-plaintext.sh present and executable"
       fi

       if [ "$FAIL" -ne 0 ]; then
         echo ""
         echo "FAIL: one or more Phase 1 CI gates are missing or disabled (D-02 violation)."
         exit 1
       fi
       echo ""
       echo "OK: all 6 Phase 1 CI gates accounted for."
       ```

       Make executable: `chmod +x scripts/audit-ci-gates.sh`.

    2. **`.github/workflows/ci.yml`** — add a step (run on `ubuntu-latest` only, near the top of the job after checkout):
       ```yaml
       - name: Audit Phase 1 CI gates
         if: runner.os == 'Linux'
         run: ./scripts/audit-ci-gates.sh
       ```

    3. Run locally: `./scripts/audit-ci-gates.sh` — must exit 0 with all 6 OK lines.

    4. If any gate is missing, **STOP** and report the missing gate to the user (this plan blocks if the audit fails — that's the entire point of plan 5). Do not silently re-add a missing gate; the prior plan should have shipped it. The user decides: re-execute the failing plan, or accept the gap with documentation.

    5. Commit: `chore(01-05): audit-ci-gates.sh + CI step to enforce D-02 gate inventory`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss && ./scripts/audit-ci-gates.sh</automated>
  </verify>
  <acceptance_criteria>
    - `scripts/audit-ci-gates.sh` is executable (`test -x scripts/audit-ci-gates.sh`)
    - `scripts/audit-ci-gates.sh` exits 0 with output containing 6 distinct `OK:` lines (one per gate plus the no-mattn-currently-green check)
    - `.github/workflows/ci.yml` contains the literal string `audit-ci-gates.sh` (verify: `grep -c 'audit-ci-gates' .github/workflows/ci.yml` returns at least 1)
    - CI run on the plan-05 branch shows the audit step green on ubuntu-latest
    - If audit fails: this plan is BLOCKED. The blocking gate is named in the failure output; user must re-execute the failing plan before plan 05 can continue.
  </acceptance_criteria>
  <done>The CI floor is provably intact. Future PRs that delete a gate file or remove a CI step will fail the audit and block merge.</done>
</task>

<task type="checkpoint:human-verify" gate="blocking">
  <name>Task 2: D-09 dual-OS lifecycle smoke from fresh release builds</name>
  <files>.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md</files>
  <what-built>
    Phase 1 is feature-complete. Plans 01-04 implemented Wails+Vite scaffold, modernc.org/sqlite + migrations, AAD helper + AES-GCM + Argon2id + apperr + session + logger, 14 Wails methods + 6 frontend screens + vault:locked event, credential CRUD + password generator + clipboard countdown + SQLite no-plaintext gate. CI is green on ubuntu/macos/windows including all 6 D-02 gates. Dev-mode (`wails dev`) lifecycle smoke was signed off at every plan boundary on both macOS and Windows.

    Plan 5 has just landed `scripts/audit-ci-gates.sh`. Now the user (Weldon) must run the FINAL D-09 smoke: full vault lifecycle from a fresh **release** build (NOT `wails dev`) on both OSes. This catches packaging-time issues that dev-mode hides (asset embedding, signing, OS file dialog behavior, WebView2 vs WebKit rendering).
  </what-built>
  <how-to-verify>
    Perform these steps IN ORDER on macOS first, then Windows. Record results in `.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md` using the template provided in Task 3.

    ## macOS smoke

    1. `cd /c/Users/weldo/Projects/abyss`
    2. `git status` — must be clean (no uncommitted changes).
    3. Note the commit SHA: `git rev-parse HEAD`.
    4. **Build a release artifact:**
       ```bash
       wails build -platform darwin/universal
       ```
       Expect output `Built in build/bin/abyss.app`.
    5. **Ad-hoc sign** (per CLAUDE.md §7 + .planning/research/STACK.md §11):
       ```bash
       codesign --force --deep --sign - ./build/bin/abyss.app
       ```
    6. **Launch from Finder** (NOT from the terminal): right-click `abyss.app` → Open. Confirm Gatekeeper dialog ("opening an app from an unidentified developer") and click Open. (For ad-hoc-signed apps the user has to do right-click → Open the FIRST time; subsequent launches don't prompt.)
    7. **Lifecycle steps (CONTEXT D-09 verbatim):**
       a. Welcome screen renders with `Abyss` heading + 3 CTAs.
       b. Click `Create new vault`. Pick path `~/Documents/Abyss/test-v0.1.abyssvault`. Master password `correctpassword12!`. Confirm. Check the no-recovery acknowledgement. Click `Create vault`.
       c. Vault Home renders with yellow `Unlocked` pill.
       d. Click `New record`. Title `GitHub`, username `me@example.com`, URL `https://github.com`, password `hunter2-test-pw`, notes `test note`, tags `dev` and `github`. Click `Save credential`.
       e. Record list shows the GitHub credential. Click it.
       f. Record Detail renders. Password is masked (`••••••••••••`). Click reveal eye → password shows. Wait 30s → password auto-hides.
       g. Click the per-field copy icon next to Username → ClipboardCountdown card appears bottom-right with text `Secret copied. Clipboard will be cleared in 30 seconds.` and a progress bar.
       h. Wait 5s → progress bar switches to yellow color. Wait until 0 → toast `Clipboard cleared.` appears.
       i. Click `Edit` → fields become editable in-place. Change title to `GitHub Personal`. Click `Save credential`.
       j. Record list shows updated title.
       k. Open the record again. Click `Delete`. Modal `Delete this credential?` appears. Click red `Delete` button. Routed back to Vault Home, empty state visible.
       l. Click `Generate password`. Slider at 24, all 4 classes checked, exclude-ambiguous checked. Click `Generate again`. A 24-char password renders (no `O`, `0`, `l`, `1`, `I`).
       m. Click the `▾` menu next to `Copy` → pick `Copy (10s)`. Countdown shows 10s remaining and ticks down. At 0, clipboard cleared.
       n. Click `Lock vault` button. Routed to Unlock screen. Type `correctpassword12!` → unlock succeeds → Vault Home empty.
       o. Type wrong password → inline error renders the PRD-LOCKED string `Unable to unlock vault. Check your master password and try again.`
       p. Quit the app (Cmd-Q). Re-launch via Finder. Vault path is remembered → app routes directly to Unlock screen with path prefilled.

    8. If any step fails: record the failure in 01-VERIFICATION.md with screenshot/log paste; STOP; this plan blocks until the failure is fixed by the relevant prior plan.

    ## Windows smoke

    Repeat the same 8 steps on a Windows machine (`git pull` first to ensure the same commit SHA). Adjustments:

    1. `cd C:\Users\weldo\Projects\abyss`
    2. `wails build -platform windows/amd64` produces `build\bin\abyss.exe`. **Cross-compile from macOS works** because of the pure-Go SQLite (D-06 verification — if cross-compile fails with a CGO error, plan 1's D-06 lock is broken).
    3. Launch by double-clicking `abyss.exe` from Explorer. SmartScreen warning expected (per CLAUDE.md §7); click `More info` → `Run anyway`.
    4. Repeat steps 7a-7p above. The Windows file dialog will use a `.abyssvault` extension; default folder is `%USERPROFILE%\Documents\Abyss\`.
    5. The remembered path on Windows lives at `%APPDATA%\Abyss\config.json` (D-11 verified).

    On any failure: STOP and document.
  </how-to-verify>
  <resume-signal>Type `approved` once both macOS and Windows smokes are signed off in 01-VERIFICATION.md. If a smoke fails, type `failed: {macOS|Windows}: {step letter}: {description}` and identify which prior plan owns the bug.</resume-signal>
</task>

<task type="auto">
  <name>Task 3: Write 01-VERIFICATION.md (final phase verification log) + Phase 1 minimal README</name>
  <files>.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md, README.md</files>
  <read_first>
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md (full file — verify decision IDs covered in summary)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-01-SUMMARY.md, 01-02-SUMMARY.md, 01-03-SUMMARY.md, 01-04-SUMMARY.md (per-plan summaries with D-05 sign-offs from prior plans)
    - .planning/ROADMAP.md (Phase 1 success criteria — 6 numbered statements)
    - .planning/REQUIREMENTS.md (the 60 Phase 1 requirement IDs that this verification declares complete)
    - docs/PRD.md §15 v0.1 exit criteria (6 statements that match ROADMAP success criteria)
    - docs/PRD.md §19 (README requirements — DOC-01..DOC-05 ship in Phase 6; Phase 1 README is intentionally minimal)
    - CLAUDE.md §7 (build & packaging — ad-hoc sign + SmartScreen notes)
    - .planning/research/STACK.md §11 (Stack patterns by variant — packaging notes per OS)
  </read_first>
  <action>
    1. **`.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md`** — Phase 1 final verification log. Structure:
       ```markdown
       # Phase 1 — v0.1 Dogfoodable Vault — Verification Log

       **Phase status:** COMPLETE  (when all sections below are signed)
       **Final verification date:** YYYY-MM-DD
       **Final commit SHA:** {git rev-parse HEAD output at sign-off}

       ## Phase 1 Success Criteria (from ROADMAP.md / PRD §15)

       | # | Criterion | Met? | Evidence |
       |---|-----------|------|----------|
       | 1 | User can create a new vault file and unlock it with a master password (≥12 chars), receiving a generic failure message on any wrong-password attempt. | YES | macOS smoke step 7b/7n/7o; test `TestService_Unlock_WrongPassword_ReturnsUnlockFailed` |
       | 2 | User can create, view, edit, search, and delete credential records; secrets are masked by default and revealable on demand. | YES | macOS smoke step 7d-7k; tests `TestService_Create_Get_RoundTrip`, `TestService_Update_*`, `TestService_Delete_*` |
       | 3 | SQLite vault file contains no plaintext sensitive material — title, username, password, notes, tags, URL never appear in plaintext columns (verified by automated grep test in CI on every PR). | YES | `scripts/check-no-plaintext.sh` green in CI on every PR |
       | 4 | User can generate a configurable password (length 24 default, classes/exclude-ambiguous toggles), copy it with a visible countdown progress bar, and save it as a credential. | YES | macOS smoke step 7l-7m; tests in `internal/generator/password_test.go` |
       | 5 | Locking the vault clears the in-memory DEK, decrypted record state, and best-effort clipboard contents; locked vault rejects all secret-touching API calls with VAULT_LOCKED. | YES | macOS smoke step 7n; test `TestLock_ZerosDEK`, `TestService_RequireUnlocked_AllMethods`; vault:locked event handler in App.tsx |
       | 6 | App builds and runs on the primary dev OS, AND a Windows build smoke-test succeeds at end of phase. | YES | Per-plan D-05 sign-offs (plans 01-04) + Final D-09 dual-OS smoke (this document, sections below) |

       ## CI Gate Audit (CONTEXT D-02)

       Output of `./scripts/audit-ci-gates.sh` at final commit:
       ```
       {paste actual output of audit script — 6 OK: lines plus the "all 6 gates" footer}
       ```

       ## Per-Plan D-05 Sign-Offs (cross-OS at each plan boundary)

       (Refer to per-plan SUMMARY files. Reproduce the macOS + Windows commit SHA + date for each plan here.)

       | Plan | macOS sign-off | Windows sign-off |
       |------|----------------|------------------|
       | 01 storage-migrations | {date}, {SHA} | {date}, {SHA} |
       | 02 crypto-scaffolding | {date}, {SHA} | {date}, {SHA} |
       | 03 wails-api-frontend-shell | {date}, {SHA} | {date}, {SHA} |
       | 04 credential-crud-passgen-clipboard | {date}, {SHA} | {date}, {SHA} |

       ## Final D-09 Dual-OS Smoke (release builds)

       ### macOS

       - Build platform: `darwin/universal`
       - Build command: `wails build -platform darwin/universal`
       - Sign command: `codesign --force --deep --sign - ./build/bin/abyss.app`
       - Launch: right-click `abyss.app` → Open in Finder
       - Lifecycle steps (per Task 2 how-to-verify): all PASS  (or list specific failures)
       - Tested commit: {SHA}
       - Date: {date}
       - Tester: Weldon

       Notes: {any quirks observed; e.g., "macOS Tahoe pbcopy LANG fix from Wails v2.12.0 confirmed working"}

       ### Windows

       - Build platform: `windows/amd64`
       - Build command: `wails build -platform windows/amd64` (cross-compiled from macOS — D-06 verified, no CGO toolchain needed)
       - Launch: double-click `abyss.exe` in Explorer; SmartScreen → More info → Run anyway
       - Lifecycle steps (per Task 2 how-to-verify): all PASS  (or list specific failures)
       - Tested commit: {SHA}
       - Date: {date}
       - Tester: Weldon

       Notes: {any quirks observed; e.g., file dialog default folder, config.json path}

       ## Phase 1 Requirement Coverage

       60 of 60 Phase 1 requirement IDs from REQUIREMENTS.md are addressed across plans 01-04 (with VAULT-04 + SEC-09 also reinforced by plan 05). See per-plan `requirements_addressed` frontmatter for the full mapping.

       ## Open Items Forward to Phase 2

       (Items NOT in Phase 1 scope but worth flagging now)
       - Auto-lock idle timer (VAULT-08) — Phase 2.
       - Change master password (VAULT-09) — Phase 2.
       - AAD-tampering exit-criterion tests beyond the positive-path golden vectors (CONTEXT D-22) — Phase 2.
       - Best-effort byte-slice zeroing audit (SEC-10 — partial mitigation in Phase 1 via `crypto.Zero` and `session.Lock`; full audit in Phase 2).
       - Argon2id KDF tuning benchmark on slowest dev machine (SEC-02) — Phase 2.
       - TEST-01 consolidated Go-unit-test deliverable — Phase 2 (initial subset shipped here).

       ## Sign-Off

       I, Weldon, confirm Phase 1 is complete. The vault file at `~/Documents/Abyss/test-v0.1.abyssvault` (macOS) and `%USERPROFILE%\Documents\Abyss\test-v0.1.abyssvault` (Windows) was created, used, locked, re-unlocked, and the credential lifecycle was exercised end-to-end on both OSes from release builds.

       Signed: Weldon — {date}
       ```

    2. **`README.md`** — Phase 1 minimal README. Phase 6 (DOC-01..DOC-05) ships the full README per PRD §19. For now, replace whatever placeholder README is in the repo with this content:

       ```markdown
       # Abyss — Local Password & Key Vault (v0.1 Dogfoodable Vault)

       Local-first desktop password and cryptographic key vault for macOS and Windows.
       This is **v0.1** — feature-incomplete (only credential records, no key generators yet).
       The full v1.0 README ships in Phase 6.

       ## Status

       - **Current phase:** Phase 1 — v0.1 Dogfoodable Vault — COMPLETE
       - **Next phase:** Phase 2 — v0.2 Security Foundation (auto-lock, change master password, AAD-tampering tests, full Go test pass)
       - See `.planning/ROADMAP.md` for the full phase plan.

       ## What works in v0.1

       - Create a local SQLite vault with a master password (≥12 chars).
       - Unlock / manual-lock the vault.
       - Create / view / edit / search / delete credential records (title, username, URL, password, notes, tags).
       - Generate a strong random password (8-128 chars, configurable classes, exclude-ambiguous toggle).
       - Copy secrets to the OS clipboard with a visible countdown and best-effort auto-clear.

       ## What does NOT work yet

       - Auto-lock after idle (Phase 2)
       - Change master password (Phase 2)
       - API key / secure note / symmetric key / asymmetric key pair record types (Phase 3 / Phase 4)
       - AES / RSA / Ed25519 key generation + export (Phase 4)
       - Settings screen + theme picker (Phase 6)
       - Password strength meter (Phase 6)

       ## Required warnings (full versions ship in Phase 6 README)

       - **There is no password recovery.** If you lose your master password, the vault cannot be recovered.
       - **Clipboard clearing is best-effort.** OS features and third-party clipboard managers may retain history outside this app's control.
       - **Memory hygiene is best-effort.** Go's runtime makes guaranteed zero-out of secret data difficult.
       - **This is a learning-grade project.** It uses modern cryptographic patterns but has not been formally audited.

       ## Tech stack (locked — see `CLAUDE.md` for rationale)

       - Go 1.25+ + Wails v2.12.0 (NOT v3 alpha)
       - Vite 8 + React 19.2 + TypeScript 5.9 + Mantine v8.3.18
       - SQLite via `modernc.org/sqlite` (pure Go, NO CGO)
       - `golang-migrate/v4` with embedded migrations
       - AES-256-GCM + Argon2id + DEK/KEK envelope encryption with mandatory AAD

       ## Setup (macOS)

       Prerequisites:

       - Go 1.25+ (`brew install go`)
       - Node 22 LTS (`brew install node@22`)
       - Wails CLI: `go install github.com/wailsapp/wails/v2/cmd/wails@v2.12.0`
       - Xcode Command Line Tools: `xcode-select --install`

       Build & run:

       ```bash
       git clone <this-repo>
       cd abyss
       cd frontend && npm install && cd ..
       wails dev               # development mode with hot reload
       # OR
       wails build -platform darwin/universal
       codesign --force --deep --sign - ./build/bin/abyss.app
       open ./build/bin/abyss.app
       ```

       The first launch of an ad-hoc-signed `.app` will trigger a Gatekeeper dialog: right-click → Open in Finder.

       ## Setup (Windows)

       Prerequisites:

       - Go 1.25+ (download from go.dev)
       - Node 22 LTS
       - Wails CLI: `go install github.com/wailsapp/wails/v2/cmd/wails@v2.12.0`
       - WebView2 Runtime (pre-installed on Windows 11)

       Build & run:

       ```cmd
       git clone <this-repo>
       cd abyss
       cd frontend && npm install && cd ..
       wails dev
       :: OR
       wails build -platform windows/amd64
       :: launch via Explorer; SmartScreen will warn — click "More info" → "Run anyway"
       ```

       Cross-compile from macOS to Windows works because Abyss uses pure-Go SQLite. From a macOS host:

       ```bash
       wails build -platform windows/amd64
       ```

       ## Where the vault file lives by default

       - macOS: `~/Documents/Abyss/vault.abyssvault`
       - Windows: `%USERPROFILE%\Documents\Abyss\vault.abyssvault`

       The "last opened vault" path is remembered in:

       - macOS: `~/Library/Application Support/Abyss/config.json`
       - Windows: `%APPDATA%\Abyss\config.json`

       Both are plain JSON; nothing sensitive is persisted there.

       ## Tests

       Go tests:

       ```bash
       go test ./... -race -count=1
       ```

       Frontend tests (Vitest):

       ```bash
       cd frontend && npm test
       ```

       CI (GitHub Actions) runs on every push: ubuntu-latest + macos-latest + windows-latest. The CI gate inventory is documented in `.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md` and audited by `scripts/audit-ci-gates.sh`.

       ## License

       MIT. See `LICENSE`.
       ```

    3. Run `./scripts/audit-ci-gates.sh` one more time and paste the actual output into `01-VERIFICATION.md` §"CI Gate Audit".

    4. Run the manual D-09 smoke per Task 2 on macOS and Windows (already done as the human-verify checkpoint above). Fill in the actual SHAs and dates in `01-VERIFICATION.md`.

    5. Commit: `docs(01-05): Phase 1 verification log + minimal README + CI-gate audit captured`.

    6. Update `.planning/STATE.md` (the planner does NOT update STATE; this is for the executor that runs plan 5):
       - Mark Phase 1 status `complete` (or whatever STATE conventions use).
       - Bump `progress.completed_phases` to 1.
       - Move "Phase 1 research flag" entries from Blockers/Concerns to "Resolved" (they were resolved inside the plans).
       - Update `last_activity` and `last_updated` timestamps.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss && ./scripts/audit-ci-gates.sh && test -f .planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md && test -f README.md && grep -q 'Phase 1 — v0.1 Dogfoodable Vault — COMPLETE' .planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md</automated>
  </verify>
  <acceptance_criteria>
    - `.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md` exists
    - `01-VERIFICATION.md` contains the literal string `Phase 1 — v0.1 Dogfoodable Vault — Verification Log`
    - `01-VERIFICATION.md` contains a row in the success-criteria table for each of the 6 ROADMAP success criteria, each marked `YES`
    - `01-VERIFICATION.md` contains the actual output of `audit-ci-gates.sh` (6 OK lines)
    - `01-VERIFICATION.md` contains macOS smoke commit SHA and date in the §"Final D-09 Dual-OS Smoke / macOS" section
    - `01-VERIFICATION.md` contains Windows smoke commit SHA and date in the §"Final D-09 Dual-OS Smoke / Windows" section
    - `01-VERIFICATION.md` contains the user's sign-off line `Signed: Weldon — {date}` (with an actual date)
    - `README.md` contains the literal string `v0.1 Dogfoodable Vault` and `Phase 1 — v0.1 Dogfoodable Vault — COMPLETE`
    - `README.md` contains the locked tech-stack list (Go + Wails v2.12.0 NOT v3, Mantine v8.3.18, modernc.org/sqlite, etc.)
    - `README.md` contains the macOS ad-hoc-sign instructions and Windows SmartScreen note (CLAUDE.md §7)
    - `./scripts/audit-ci-gates.sh` exits 0
  </acceptance_criteria>
  <done>Phase 1 verification log + minimal README committed; STATE.md reflects Phase 1 complete; the vault is provably dogfoodable on macOS AND Windows from release builds.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| CI configuration ↔ committed source | A future PR could silently delete a gate file or remove a CI step; `audit-ci-gates.sh` runs in CI itself to catch this. |
| Release build ↔ user's OS launcher | macOS Gatekeeper + Windows SmartScreen warn the user about unsigned/ad-hoc builds; user accepts the warning. |
| Cross-compile macOS → Windows | The build pipeline trusts that pure-Go SQLite + the Go toolchain produce a working binary; mitigated by D-06 lock (verified in plan 01) plus the actual D-09 Windows smoke. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-05-01 | Tampering | A future PR silently disables a CI gate (e.g., comments out the no-plaintext step or deletes `.golangci.yml`) | mitigate | `scripts/audit-ci-gates.sh` runs as a dedicated CI step on every PR; failure blocks merge. The script's grep patterns are the canonical marker strings of each gate; renaming a gate file requires updating the audit script in the same PR (intentional pressure to keep the audit in sync). |
| T-05-02 | Spoofing | A test bypass via build tag + skipped tests | mitigate | `golangci-lint run` includes `unused` linter; CI runs `go test ./... -race -count=1` (not `-short` in CI); audit script greps for the `golangci-lint` invocation in CI yaml so it can't be silently dropped. |
| T-05-03 | Information Disclosure | macOS smoke leaves a real vault file at `~/Documents/Abyss/test-v0.1.abyssvault` containing the test password `hunter2-test-pw` | accept | Test password is sentinel-grade (already widely known and not used elsewhere); test vault is in user's home directory under their control; user can delete after smoke. README v1.0 documents the smoke procedure. |
| T-05-04 | Information Disclosure | Windows smoke runs an unsigned `.exe` past SmartScreen — user is being trained to dismiss security warnings | accept | PRD §3.3 + CLAUDE.md §7 explicitly defer code signing for MVP; learning-grade project. README v1.0 documents that production users should expect to either build from source or wait for v1.x signed builds. |
| T-05-05 | Tampering | The fixture vault from `cmd/no-plaintext-fixture` accidentally writes to a path under version control | mitigate | The fixture binary takes `--out` and is ONLY invoked by `scripts/check-no-plaintext.sh` which uses `mktemp -d` + `trap rm`. The temp directory is outside the repo. Manual smoke uses a path under `~/Documents/Abyss/` which is `.gitignore`'d already (Wails scaffold ignores `*.abyssvault`). |
| T-05-06 | Denial of Service | Cross-compile from macOS to Windows pulls a CGO transitive dependency that breaks the build | mitigate | `go.sum` audit gate (plan 01 + audit script) catches `mattn/go-sqlite3`. The actual `wails build -platform windows/amd64` step in this plan's smoke is the integration-level proof that the toolchain stays CGO-free. |
</threat_model>

<verification>
**End-of-plan checks:**

1. `./scripts/audit-ci-gates.sh` exits 0 on macOS, Linux (CI), and Windows; output contains 6 OK lines plus the "all 6 gates accounted for" footer
2. `wails build -platform darwin/universal` exits 0 on macOS; produces a runnable `abyss.app`
3. `wails build -platform windows/amd64` exits 0 on macOS (cross-compile); produces a runnable `abyss.exe`
4. The macOS lifecycle smoke completes successfully (steps 7a-7p of Task 2)
5. The Windows lifecycle smoke completes successfully (steps 7a-7p of Task 2)
6. `01-VERIFICATION.md` contains commit SHAs + dates for both OS smokes; user sign-off present
7. `README.md` contains the locked tech-stack note + setup instructions for both OSes
8. CI workflow latest run reports `success` on ubuntu-latest + macos-latest + windows-latest INCLUDING the new audit-ci-gates step
9. `.planning/STATE.md` is updated to reflect Phase 1 complete (progress.completed_phases = 1)
</verification>

<success_criteria>
- All 6 D-02 CI gates inventoried and proven active by `scripts/audit-ci-gates.sh` (no silent disablement during plans 01-04)
- The audit script runs in CI on every PR; failure blocks merge
- Full Phase 1 lifecycle (Welcome → Create → Vault Home → Create credential → Reveal → Per-field copy with countdown → Edit → Delete → Generate password → Copy with per-copy menu → Lock → Re-unlock → Wrong-password generic failure → Persisted last-vault-path on relaunch) passes from a fresh release build on both macOS AND Windows
- Cross-compile from macOS to Windows works (D-06 pure-Go SQLite promise verified at the integration level)
- `01-VERIFICATION.md` is committed with commit SHAs, dates, and user sign-off for both OS smokes
- `README.md` is the Phase 1 minimal version (full README per DOC-01..DOC-05 ships in Phase 6); contains tech-stack lock, both-OS setup, ad-hoc-sign + SmartScreen notes, the four required warnings (no recovery, clipboard, memory, learning-grade)
- `.planning/STATE.md` updated to reflect Phase 1 status `complete`
</success_criteria>

<output>
After completion, create `.planning/phases/01-v0-1-dogfoodable-vault/01-05-SUMMARY.md` documenting:
- All 6 D-02 CI gates accounted for: errcheck/golangci-lint baseline (plan 01), no-mattn go.sum gate (plan 01), forbidigo math/rand ban (plan 02), wails generate module no-drift (plan 03), api-wrapper boundary check (plan 03), SQLite no-plaintext grep gate (plan 04). Plus the audit-ci-gates.sh meta-gate landed in this plan.
- Cross-compile from macOS to Windows produced a runnable abyss.exe — D-06 (pure-Go SQLite, no CGO) integration-verified.
- Final D-09 dual-OS lifecycle smoke result, with commit SHAs + dates per OS.
- All 60 Phase 1 requirement IDs marked complete in REQUIREMENTS.md (plan 5 is the trigger that flips them).
- Phase 2 carry-forward items (per `01-VERIFICATION.md` §"Open Items Forward to Phase 2"): VAULT-08 auto-lock, VAULT-09 change-master, full AAD-tampering tests (CONTEXT D-22), SEC-02 KDF benchmark, SEC-10 memory hygiene audit, TEST-01 consolidated test pass.
- Phase 1 closeout date + final commit SHA on the prod branch.
</output>
