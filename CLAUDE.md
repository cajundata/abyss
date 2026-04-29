<!-- GSD:project-start source:PROJECT.md -->
## Project

**Abyss — Local Password & Key Vault**

Abyss is a local-first desktop password and cryptographic key vault for macOS and Windows. It lets a single user create and unlock an encrypted SQLite vault, store credentials and keys, generate strong passwords and AES/RSA/Ed25519 keys, and safely copy or export selected secrets. The product target is a learning-grade but functionally useful personal vault — not a commercial password-manager competitor.

**Core Value:** A user can create an encrypted local vault with a master password, store and retrieve secrets, and never see plaintext sensitive data persisted to disk. If only one thing must work, it is: **vault confidentiality and tamper-evidence (DEK/KEK envelope encryption with AES-256-GCM AAD), end-to-end, on both macOS and Windows.**

### Constraints

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
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## 0. Already Locked by PRD — Do Not Re-Debate
| Locked Choice | Source | Rationale (recap from PRD) |
|---|---|---|
| Go + Wails **v2 stable** (NOT v3 alpha) | PRD §2.1, §18 | Wails v2 is the explicit learning target; v3 is alpha and unstable for MVP. |
| Vite + React + TypeScript + Mantine (NOT Next.js, no SSR) | PRD §2.2, §18 Hard Rule #10/#11 | Inside Wails the frontend is a static SPA; SSR/server components add no value and break the embed model. |
| SQLite with **rollback journal** (`PRAGMA journal_mode = DELETE`) | PRD §2.4, §6.2, §18 Hard Rule #12 | Vault portability beats throughput; users zip/move/back-up the file. WAL adds sidecar files. |
| **NOT** SQLCipher | PRD §2.4 | Record-level encryption preferred over whole-DB encryption for portability + transparency. |
| `golang-migrate` (NOT hand-rolled), embedded into the binary | PRD §2.5 | Versioned, repeatable, embeddable. |
| AES-256-GCM, Argon2id, `crypto/rand`, `golang.org/x/crypto/ssh` | PRD §5, §18 Hard Rules #8/#13 | Standard, vetted primitives only. No custom algorithms. No `math/rand`. |
| DEK/KEK envelope with AAD on every AES-GCM op | PRD §2.6, §5.2, §5.4 | Industry-standard envelope; AAD prevents record swap. |
| `application_id = 0x4E56544E` validated before unlock | PRD §6.1 | File-type identification beyond extension. |
| Target OS: macOS + Windows (NOT Linux) | PRD §2.3, §18 Hard Rule #5 | Reduces packaging surface for a learning project. |
| Typed `AppErrorCode` DTO at the Wails boundary; no string-matching errors in frontend | PRD §10.1, §18 Hard Rule #14 | Clean error contracts. |
## 1. Recommended Stack
### Core Technologies
| Technology | Version (April 2026) | Purpose | Why this version |
|------------|----------------------|---------|------------------|
| **Go** | **1.25.9** (preferred) or **1.26.2** (latest) | Backend language for Wails app | Go 1.24 reached EOL on 2026-02-11; Wails v2 requires Go 1.21+ but Go 1.23.3+ is required for macOS 15+. 1.25 is the conservative LTS-grade pick; 1.26 is current. Confidence: HIGH. |
| **Wails** | **v2.12.0** (released 2026-03-26) | Desktop framework: Go backend + WebView frontend, JS↔Go method bindings | Latest v2 stable. Includes the macOS 26 (Tahoe) WebView crash fix during rapid UI updates and the macOS clipboard mojibake fix (LANG env for pbpaste/pbcopy). Confidence: HIGH. |
| **Node.js** | **20.x LTS** (Iron) or **22.x LTS** (Jod) | Frontend toolchain runtime (Vite, npm) | Node 22 LTS is the current active LTS as of April 2026; Node 20 is in maintenance LTS until April 2026. Pick Node 22 for new projects. Confidence: HIGH. |
| **TypeScript** | **5.9.x** (use 5.9.2+) | Frontend type system | TS 6.0 shipped 2026-03-23 as the final JS-based release. TS 7.0 beta (Go-native, ~10x faster) released 2026-04-21. **Stick to 5.9 for MVP** — 6.0 has deprecation warnings, 7.0 is beta. Confidence: HIGH. |
| **Vite** | **8.0.x** (use 8.0.10) | Frontend bundler/dev server | Vite 8 (released 2026-03-12) ships Rolldown (Rust) as the unified bundler — 10–30x faster builds. Wails `wails dev` proxies to whatever Vite version is in `frontend/package.json`. Confidence: HIGH. |
| **React** | **19.2.5** | UI library | React 19.2 stable since Oct 2025; Mantine v9 *requires* React 19.2+. Confidence: HIGH. |
| **Mantine (core + hooks)** | **v8.3.18** (last 8.x) — **DO NOT use v9 yet** | Component library + hook utilities | **See §1a below for critical Mantine version decision.** Confidence: HIGH on rationale; MEDIUM on whether v9 will stabilize fast enough during MVP. |
| **SQLite (via Go driver)** | `modernc.org/sqlite` **v1.36.0+** (CGO-free) | Embedded database | **See §1b for critical driver decision.** Confidence: HIGH. |
| **golang-migrate** | `github.com/golang-migrate/migrate/v4` (v4 API stable/frozen) | Schema migrations, embedded via `iofs` source | v4 API is frozen. Use the `iofs` driver with Go 1.16+ `embed.FS`. Confidence: HIGH. |
### 1a. Mantine version: v8.3 vs v9 — pick **v8.3.18** for MVP
- v9 *requires* React 19.2+, which is fine, but its v9.1 features (`deduplicateInlineStyles`, `<Activity>` integration, refactored hooks using stable `useEffectEvent`) are nice-to-haves, not must-haves. You will spend more time chasing v9 patch noise than you save by adopting early.
- The v7→v8 migration guide (https://mantine.dev/guides/7x-to-8x/) and v8→v9 guide are both well documented. Upgrading mid-MVP is a known operation, not a leap.
- v8.3 already has the modern `useForm` uncontrolled mode (significant render-perf improvement) and built-in Standard Schema support for Zod via `schemaResolver`, which covers everything PRD §9.3/§9.4/§9.6/§9.8 needs.
### 1b. SQLite driver: `modernc.org/sqlite` (pure Go) over `mattn/go-sqlite3` (CGO)
| Factor | `mattn/go-sqlite3` | `modernc.org/sqlite` |
|---|---|---|
| CGO required | YES (`CGO_ENABLED=1` + gcc) | NO (pure Go) |
| Cross-compile macOS → Windows | Painful: needs MinGW (`x86_64-w64-mingw32-gcc`) | Trivial: `GOOS=windows go build` |
| Wails build complexity | High on cross-compile | Low |
| Performance — INSERT | Baseline | ~2x slower |
| Performance — SELECT | Baseline | 10%–2x slower |
| Binary size | Smaller (links system libsqlite where possible) | Larger (~2 MB more, embedded) |
| Stability | Battle-tested, used everywhere | v1.36+ also widely used (Caddy, gotosocial, Apache Arrow); v1 series is stable. |
### Supporting Libraries (Go backend)
| Library | Version | Purpose | When/Why to use |
|---------|---------|---------|-----------------|
| `golang.org/x/crypto/argon2` | latest (track Go release) | Argon2id KDF | PRD §5.3 lock. Use `argon2.IDKey(pw, salt, time, memory, threads, keyLen)`. Note: package returns the raw key only — store algorithm/params/salt yourself in `key_wrappings.kdf_params_json`. Confidence: HIGH. |
| `golang.org/x/crypto/ssh` | latest | OpenSSH public-key marshaling, fingerprints | PRD §18 lock. Use `ssh.NewPublicKey(pub)` then `ssh.MarshalAuthorizedKey(pubKey)` for OpenSSH format; `ssh.FingerprintSHA256(pubKey)` for fingerprint. Confidence: HIGH. |
| `crypto/x509` (stdlib) | n/a | PKCS#8 PEM encoding | `x509.MarshalPKCS8PrivateKey` accepts `*rsa.PrivateKey`, `*ecdsa.PrivateKey`, `ed25519.PrivateKey` (NOT a pointer), `*ecdh.PrivateKey`. Wrap result in `pem.Block{Type: "PRIVATE KEY"}`. Confidence: HIGH. |
| `crypto/rand` (stdlib) | n/a | All security-sensitive randomness | PRD §18 Hard Rule #13. Use for DEK, salts, nonces, password chars, AES key material. Confidence: HIGH. |
| `crypto/aes` + `crypto/cipher` (stdlib) | n/a | AES-256-GCM | `aes.NewCipher(key)` → `cipher.NewGCM(block)` → `gcm.Seal(nil, nonce, plaintext, aad)` / `gcm.Open(nil, nonce, ciphertext, aad)`. Confidence: HIGH. |
| `crypto/ed25519` (stdlib) | n/a | Ed25519 keypair generation | `ed25519.GenerateKey(rand.Reader)`. Confidence: HIGH. |
| `crypto/rsa` (stdlib) | n/a | RSA keypair generation | `rsa.GenerateKey(rand.Reader, bits)` for 2048/3072/4096. Confidence: HIGH. |
| `github.com/golang-migrate/migrate/v4` | v4 (frozen API) | Migrations runner | Use with `iofs` source driver for embed.FS. See §3 below for wiring. Confidence: HIGH. |
| `github.com/golang-migrate/migrate/v4/database/sqlite` | matches /v4 | SQLite database driver for migrate **with `modernc.org/sqlite`** | The default `database/sqlite3` subpackage uses `mattn`. Use the `database/sqlite` (without "3") subpackage which is pure-Go-compatible. Confidence: MEDIUM — verify the exact import name and DSN format during v0.1 implementation. |
| `github.com/google/uuid` | v1.6.x | `vault_id`, `dek_id`, `record_id`, `wrapping_id` generation | UUIDv4 via `uuid.New().String()`. PRD does not mandate UUID format but the schema uses TEXT primary keys — UUIDs are the obvious choice. Alternative: `crypto/rand`-derived ULIDs. Confidence: HIGH. |
| `github.com/stretchr/testify` | v1.10+ | Test assertions for Go | Hybrid approach: stdlib `testing` + `testify/assert` and `testify/require` only where readability gains are large (multi-field equality, error matching). For golden-vector AAD/AES tests, prefer stdlib `bytes.Equal` + `t.Errorf` for clarity. Confidence: MEDIUM (this is a style call, not a correctness call). |
| `github.com/stretchr/testify/require` | v1.10+ | Fail-fast assertions in setup/preconditions | Useful inside `TestMain`/`setup` to abort on infrastructure errors. Confidence: HIGH. |
- The PRD excludes a custom strength estimator (§8.3) and asks for a vetted zxcvbn-style library only if any. **Recommendation: defer zxcvbn entirely from MVP** — it's a "Should" requirement (FR-033), not "Must". The Go ports (`nbutton23/zxcvbn-go`, last commit Feb 2023; `trustelem/zxcvbn`) are unmaintained or thin. If strength UI is desired, run **`@zxcvbn-ts/core` v3** in the *frontend* (no Go side dependency) so the master password never crosses the Wails boundary just for a strength check. Confidence: MEDIUM — the frontend-only approach is cleaner but document the tradeoff that the master-password is briefly in the renderer process before submission.
### Supporting Libraries (Frontend)
| Library | Version | Purpose | When/Why to use |
|---------|---------|---------|-----------------|
| `@mantine/core` | **^8.3.18** | UI component library | See §1a. |
| `@mantine/hooks` | **^8.3.18** | Hook utilities incl. `useClipboard`, `useDebouncedValue`, `useIdle`, `useInterval`, `useTimeout` | `useIdle` is critical for the auto-lock implementation (PRD §13). `useInterval` drives the clipboard countdown progress bar (PRD §9.7). Note: `useClipboard` is a *frontend* clipboard hook — DO NOT use it for secret material. Backend `CopyToClipboard` Wails method (PRD §10.6) owns secret writes. Confidence: HIGH. |
| `@mantine/form` | **^8.3.18** | Form state management | Built-in Standard Schema support. Pair with `zod`. Use `mode: 'uncontrolled'` for performance. Confidence: HIGH. |
| `@mantine/notifications` | **^8.3.18** | Toast notifications for "copied", "cleared", error feedback | Optional but well-integrated. Confidence: HIGH. |
| `@mantine/modals` | **^8.3.18** | Centralized modal API; useful for the private-key warning modal (PRD §9.8) | Optional — could also use `<Modal>` directly. Confidence: HIGH. |
| `zod` | **v3.x** (use 3.23+) | Runtime schema validation, paired with `@mantine/form` `schemaResolver` | Zod 4 exists but Mantine's `schemaResolver` requires `mantine-form-zod-resolver@1.2.1+` for v4 support; staying on v3 is the path of least friction. Confidence: HIGH. |
| `zustand` | **v5.x** (use 5.0.x) | Client-side state for vault session, lock state, current record, search query | **See §1c below.** Confidence: HIGH. |
| `react-router-dom` | **v6.x** (use 6.30+) or **v7.x** | Routing among the 10 PRD §9.1 screens | v6 is mature; v7 added new APIs but v6 is fine and has a smaller learning curve. Confidence: HIGH. |
| `vitest` | **v3.x** (use 3.2+) | Test runner | Vitest 3 is the current major. Reuses Vite config. Confidence: HIGH. |
| `@testing-library/react` | **v16.x** | Component testing | Compatible with React 19. Confidence: HIGH. |
| `@testing-library/user-event` | **v14.x** | Realistic user interactions in tests | Confidence: HIGH. |
| `jsdom` | **v25.x** | DOM env for Vitest | Confidence: HIGH. |
| `@zxcvbn-ts/core` + relevant language pack | **v3.x** | (Optional) Password strength estimator for master-password and generated-password feedback | **Defer to v0.5+ unless requested.** Confidence: MEDIUM. |
### 1c. Frontend state management: **Zustand**, with a small surface
| Option | Verdict |
|---|---|
| **React Context only** | Works for the 1–2 globals (vault status, theme), but you'll quickly hit re-render storms once you have a record list + search + clipboard countdown. Don't make the whole UI re-render every second from the countdown. |
| **Zustand** | **RECOMMENDED.** Single store (or 2 — `vaultStore`, `clipboardStore`), selectors prevent re-renders, no provider boilerplate, zero ceremony. ~1 KB. |
| **Jotai** | Atomic model is great in theory but overkill for this app's scope; primary state shape is "the current vault session" — flat and module-scoped, not atomically composed. |
| **Redux / Redux Toolkit** | Forbidden by common sense for an app this small. Don't. |
### 1d. Clipboard: **Wails runtime API**, not a third-party Go clipboard package
| Option | Verdict |
|---|---|
| **Wails runtime: `runtime.ClipboardSetText(ctx, text)` / `ClipboardGetText`** | **RECOMMENDED.** Already in your dep tree. Text-only (which is exactly what you need). Returns `error` so you can map to `CLIPBOARD_FAILED`. macOS/Windows handled. |
| `golang.design/x/clipboard` | Adds image + watcher; you don't need either. Imports `cgo` on Windows (uses Windows API directly via syscalls but adds complexity). |
| `atotto/clipboard` | Older, calls `pbcopy`/`xclip`/PowerShell `Set-Clipboard`. Works but invokes external processes — fragile, slower, and can stall a goroutine. |
- Backend `CopyToClipboard(req)` writes `req.value`, records `secretKind` for sanitized logging, starts no timer (frontend owns the timer).
- Backend `ClearClipboard()` calls `ClipboardSetText(ctx, "")`. Optional pre-clear write of a harmless string per PRD FR-064 ("Could").
- Frontend `useInterval` drives the progress bar; calls `ClearClipboard` at zero.
### Development Tools
| Tool | Purpose | Notes |
|------|---------|-------|
| `wails` CLI | Project scaffold, dev mode, build | `go install github.com/wailsapp/wails/v2/cmd/wails@v2.12.0`. `wails doctor` reports missing deps. Confidence: HIGH. |
| Xcode Command Line Tools | macOS WebKit headers | Required on macOS for Wails. `xcode-select --install`. Confidence: HIGH. |
| WebView2 Runtime (Windows) | Edge Chromium WebView host | Pre-installed on Windows 11; Wails v2 ships a self-contained build option. Confidence: HIGH. |
| Node.js 22 LTS + npm 10 | Frontend toolchain | Comes with Wails dev mode. Confidence: HIGH. |
| `golangci-lint` | Go linting | v1.62+ as of April 2026. Recommended in CI. Confidence: HIGH. |
| `gofumpt` (or `gofmt`) | Go formatting | Optional; `gofumpt` is stricter. Confidence: MEDIUM. |
| `prettier` | Frontend formatting | v3.x. Confidence: HIGH. |
| `eslint` | Frontend linting | v9.x with flat config. The Wails react-ts template ships an older config; expect to upgrade. Confidence: MEDIUM. |
| **Vitest UI** (`@vitest/ui`) | Optional test browser | Useful during dev. Confidence: MEDIUM. |
| **Playwright** for E2E | NOT recommended for MVP | Wails apps don't expose a CDP-attachable browser by default; E2E tooling for Wails is immature. Use the manual cross-platform test matrix from PRD §14.3 instead. Confidence: HIGH (recommendation). |
## 2. Installation
### Backend (Go)
# Wails CLI
# Verify environment
# Scaffold (run once, in a sibling/empty folder, then move generated frontend/ into repo)
### Backend module dependencies
# Pure-Go SQLite driver
# Migrations
# UUIDs
# Test assertions (optional but recommended)
# x/crypto already comes in indirectly via Wails; ensure it's direct:
### Frontend (in `frontend/`)
# Mantine
# Mantine peer deps (already present from template, but pin)
# Validation
# State & routing
# Dev: testing
# Dev: tooling
### `frontend/index.html` & root entry
- Import `@mantine/core/styles.css` (and `@mantine/notifications/styles.css` if used) before app styles.
- Wrap `<App />` in `<MantineProvider defaultColorScheme="auto">` (per PRD §9 theme preference setting).
## 3. golang-migrate Wiring (embedded migrations)
## 4. Argon2id Parameter Handling
## 5. OpenSSH / PEM / PKCS#8 Marshaling
- `ssh.ParseAuthorizedKey(pubOpenSSH)` → returns `ssh.PublicKey`, comment, options, rest. Use this in tests to verify your exported public key parses back to an equivalent key.
- `ssh.ParseRawPrivateKey(privPEM)` accepts `*rsa.PrivateKey`, `*ecdsa.PrivateKey`, `ed25519.PrivateKey` from PKCS#8, PKCS#1, OpenSSL, OpenSSH formats.
- Or use `x509.ParsePKCS8PrivateKey(block.Bytes)` for direct PKCS#8 round-trip.
## 6. SQLite Test Isolation Strategy
| Test type | Pattern |
|---|---|
| **Pure crypto / encoding** (no DB) | stdlib only, `testing` package. Use golden vectors for AAD/AES decryption — known plaintext, key, nonce, AAD → expected ciphertext. |
| **Storage / SQL / migration** | Per-test temp file: `dir := t.TempDir(); path := dir + "/vault.db"`. The OS cleans up automatically. Avoid `:memory:` for migration tests because `iofs` + connection-pooling can hit "database is locked" with shared memory DBs unless you set `?cache=shared&mode=memory`. |
| **Integration** (full vault lifecycle) | Temp-file vault per test. `t.Cleanup(func(){ db.Close() })`. |
| **Wails-bound services** | Test the Go service struct directly (e.g., `vaultService.UnlockVault(req)`); do not test the binding layer. The binding layer is autogenerated by Wails — trust it. |
## 7. Build & Packaging
| Aspect | macOS | Windows | Confidence |
|---|---|---|---|
| Dev build | `wails dev` | `wails dev` | HIGH |
| Production build | `wails build -platform darwin/universal` (Intel + ARM fat binary) or `darwin/arm64` only | `wails build -platform windows/amd64` | HIGH |
| Cross-compile macOS → Windows | Works because we picked pure-Go SQLite. NSIS installer requires a Windows machine OR `makensis` via Homebrew. | n/a | MEDIUM |
| Code signing | Optional for MVP (learning project). Defer notarization. App will warn user on first run. | Optional. Unsigned `.exe` shows SmartScreen warning. Defer signing certs. | HIGH on "defer is fine"; MEDIUM on whether the user wants to ship unsigned to themselves. |
| Installer | `.app` bundle (default), or `.dmg` via `hdiutil` (manual scripting; not built into Wails v2). | NSIS via `wails build -platform windows/amd64 -nsis`. | HIGH |
| Auto-update | NOT in MVP scope (PRD §3.3). | NOT in MVP scope. | HIGH |
- macOS: Apple Developer ID Application certificate ($99/yr) + notarization via `notarytool`. **Defer.** A self-signed/ad-hoc signed `.app` runs on the developer's own machine after right-click → Open. Document this in the README.
- Windows: Authenticode certificate (~$200–$700/yr for code signing). **Defer.** SmartScreen will scare users; for a single-user learning project this is acceptable.
## 8. Frontend Layout in Wails
## 9. Alternatives Considered
| Recommended | Alternative | When to use the alternative |
|---|---|---|
| `modernc.org/sqlite` | `mattn/go-sqlite3` | If you accept CGO toolchain on both build hosts (or use platform-native CI) and want the absolute best performance. |
| Wails v2.12.0 | Wails v3 alpha | Once v3 reaches stable. **Not before then.** PRD forbids it for MVP. |
| Mantine v8.3.18 | Mantine v9.1+ | Once v9.x has 3+ months of stabilization (~July 2026) and you've started a fresh phase. Migrate at a milestone boundary, not mid-phase. |
| Zustand v5 | Plain React Context | If the entire app fits on 1–2 screens. Abyss has 10 screens; Zustand pays off immediately. |
| Wails runtime clipboard | `golang.design/x/clipboard` | If you need image clipboard support (you don't). |
| Vitest + RTL (no E2E) | Add Playwright E2E later | Only if the manual cross-platform matrix in PRD §14.3 becomes too painful. Unlikely for a 1-developer learning project. |
| `@mantine/form` + Zod via `schemaResolver` | React Hook Form + Zod | If you find Mantine's form ergonomics insufficient. Not expected. |
| stdlib `testing` + selective `testify` | Pure stdlib, no testify | If repo style strongly favors stdlib. Either choice is fine. |
| Defer code signing | Sign + notarize from day 1 | If publishing to other users. Not MVP scope. |
## 10. What NOT to Use
| Avoid | Why | Use Instead |
|---|---|---|
| **Wails v3 alpha** | Pre-release, breaking changes weekly, PRD §2.1 forbids it. | Wails v2.12.0 |
| **Next.js / SSR** | PRD §2.2, §18 Hard Rule #10/#11 forbid it. SSR is meaningless inside a static-embedded webview. | Vite + React SPA |
| **SQLCipher / `mutecomm/go-sqlcipher`** | PRD §2.4 forbids it; record-level encryption is the chosen model. | Plain SQLite + AES-256-GCM record envelope |
| **`mattn/go-sqlite3`** (default for many tutorials) | Requires CGO; cross-compile macOS↔Windows is painful for a one-machine dev workflow. | `modernc.org/sqlite` |
| **WAL mode** (`PRAGMA journal_mode=WAL`) | PRD §2.4, §6.2, §18 Hard Rule #12 forbid it for MVP. Sidecar files break vault portability. | `journal_mode = DELETE` (rollback) |
| **`math/rand`** for ANY security-sensitive randomness | PRD §18 Hard Rule #13. Not cryptographically secure. | `crypto/rand` |
| **Custom Argon2 / AES / GCM implementations** | PRD §18 Hard Rule #8. Constant-time and side-channel correctness are easy to get wrong. | `golang.org/x/crypto/argon2`, stdlib `crypto/aes` + `cipher.NewGCM` |
| **`bcrypt` or `scrypt`** for KEK derivation | PRD §5.1 mandates Argon2id specifically. | `argon2.IDKey` |
| **Storing master password / KEK / DEK in any persistent location** | PRD §5.2, §12.2. | Memory only; clear on lock; never log. |
| **Custom password strength estimator** | PRD §8.3, §18 Hard Rule equivalent. Hand-rolled estimators are systematically wrong. | Defer entirely, OR use `@zxcvbn-ts/core` v3 in the frontend only. |
| **String-matching error messages on the frontend** | PRD §10.1, §18 Hard Rule #14. | `AppErrorCode` enum + map to display strings in the frontend. |
| **`@mantine/form` with `mode: 'controlled'`** for large forms | High re-render cost. | `mode: 'uncontrolled'` (default in v8). |
| **Shared mutable global state without selector-based access** | Re-render storms; the clipboard countdown will cause the entire UI to re-render every second. | Zustand stores with selectors (`useStore(s => s.specificField)`). |
| **`atotto/clipboard`** | Spawns external processes (`pbcopy`, `powershell`). Slower, fragile, no error path on failure. | `runtime.ClipboardSetText` from Wails v2 |
| **Auto-update libraries (e.g., `go-selfupdate`)** | PRD §3.3 explicitly defers auto-update. | Manual builds, manual versioning. |
| **`os.WriteFile` for vault writes** | SQLite owns all vault file writes. | `database/sql` API only. |
## 11. Stack Patterns by Variant
- Update React peer requirement to `react@^19.2`.
- Add `mantine-form-zod-resolver@^1.2.1` if using Zod 4.
- Replace any usage of removed v8 APIs per the 8x→9x migration guide.
- Risk: chasing patch releases during MVP. Reward: future-proofing.
- `modernc.org/sqlite` is mandatory. Don't even try CGO cross-compile.
- Use GitHub Actions for Windows build if possible (matrix on `windows-latest` + `macos-latest`); the macOS-host-only approach works but adds friction.
- macOS: Apple Developer ID + `notarytool` + staple ticket.
- Windows: Authenticode cert + `signtool.exe`.
- Add release-signing CI matrix.
- Update README to remove the "ad-hoc signed only" caveat.
## 12. Version Compatibility Matrix
| Component A | Compatible with | Notes |
|---|---|---|
| Wails v2.12.0 | Go 1.21+ (Go 1.23.3+ on macOS 15+) | Use Go 1.25 or 1.26 in 2026. Go 1.24 EOL. |
| Wails v2.12.0 | Node 18+ for frontend tooling | Use Node 22 LTS. |
| Mantine 8.3 | React 18.x or 19.x | React 19.2 works; tests pass. Mantine 9 *requires* 19.2+. |
| `@mantine/form` v8 + `zod` v3 | Native via `schemaResolver` | No separate resolver package needed. Zod v4 needs `mantine-form-zod-resolver@1.2.1+`. |
| `modernc.org/sqlite` v1.36+ | Go 1.21+ | Avoid retracted versions (e.g., v1.34.3). |
| `golang-migrate/v4` | Go 1.18+ | API frozen on v3 & v4. Use `iofs` source for `embed.FS`. |
| Vitest v3.2 | Vite v5/v6/v7/v8 | Wails template default Vite version varies by Wails CLI version; bump Vite + Vitest together. |
| React Testing Library v16 | React 18, React 19 | Required for React 19 support. |
## 13. Confidence Summary
| Area | Confidence | Why |
|---|---|---|
| Wails v2.12.0 as latest stable | HIGH | Confirmed via wailsapp/wails GitHub release page (March 2026). |
| Go 1.25/1.26 recommendation | HIGH | Go 1.24 EOL date confirmed, current LTS-grade pick. |
| Mantine v8.3.18 over v9 for MVP | HIGH | v9 is <1mo old as of MVP planning; 8.3 is "last 8.x" and stable. |
| `modernc.org/sqlite` over `mattn/go-sqlite3` | HIGH | CGO cross-compile pain is well documented; modernc is widely used in production projects (Caddy, Apache Arrow, gotosocial). |
| Zustand v5 for state | HIGH | Standard recommendation for medium React apps; matches Abyss's shape. |
| Wails runtime clipboard over Go libs | HIGH | Already in dep tree; PRD §2.7 lock; v2.12 has the macOS LANG fix. |
| Argon2id IDKey signature & params | HIGH | golang.org/x/crypto/argon2 godoc directly verified. |
| OpenSSH/PEM/PKCS#8 round-trip API | HIGH | golang.org/x/crypto/ssh godoc directly verified; supports RSA + Ed25519 with PKCS#8. |
| `database/sqlite` migrate driver path & DSN params | MEDIUM | Documentation is sparse; verify exact import path and DSN keys in v0.1 implementation phase. |
| Defer code signing for MVP | HIGH | Learning-grade scope; PRD allows deferral; ad-hoc/self-sign workflow is documented. |
| Vitest + RTL versions | HIGH | Major library versions verified via npm + recent guides. |
| Zod v3 over v4 for now | MEDIUM | Both work; v3 has the smoothest path with Mantine v8's built-in `schemaResolver`. |
| TypeScript 5.9 over 6/7 for MVP | HIGH | TS 6 has deprecation warnings; TS 7 is beta. |
## 14. Sources
- [Wails Releases (GitHub)](https://github.com/wailsapp/wails/releases) — v2.12.0 confirmed as latest v2 stable, March 2026.
- [Wails Changelog](https://wails.io/changelog/) — verified v2.12.0 fixes (macOS Tahoe WebView, clipboard mojibake).
- [Wails Installation docs](https://wails.io/docs/gettingstarted/installation/) — Go version requirements.
- [Wails Clipboard Runtime](https://wails.io/docs/reference/runtime/clipboard/) — `ClipboardSetText` / `ClipboardGetText` API.
- [Wails NSIS installer guide](https://wails.io/docs/guides/windows-installer/) — Windows installer build flow.
- [Wails Code Signing guide](https://wails.io/docs/guides/signing/) — macOS and Windows signing requirements.
- [golang.org/x/crypto/argon2 godoc](https://pkg.go.dev/golang.org/x/crypto/argon2) — `IDKey` signature, parameter semantics.
- [golang.org/x/crypto/ssh godoc](https://pkg.go.dev/golang.org/x/crypto/ssh) — `MarshalAuthorizedKey`, `NewPublicKey`, `FingerprintSHA256`, `ParseRawPrivateKey`.
- [crypto/x509 PKCS#8 source](https://go.dev/src/crypto/x509/pkcs8.go) — supported key types.
- [Mantine all releases](https://mantine.dev/changelog/all-releases/) — v8.3.18 (2026-03-17), v9.0 (2026-03-31), v9.1 (2026-04-21).
- [Mantine v8.3.0 changelog](https://mantine.dev/changelog/8-3-0/) — uncontrolled `useForm` mode.
- [Mantine 7→8 migration guide](https://mantine.dev/guides/7x-to-8x/) and [8→9 migration guide](https://mantine.dev/guides/8x-to-9x/).
- [Mantine form schema validation](https://mantine.dev/form/schema-validation/) — Standard Schema + Zod.
- [Mantine useClipboard hook](https://mantine.dev/hooks/use-clipboard/) — frontend-only clipboard hook (NOT for secrets).
- [modernc.org/sqlite godoc](https://pkg.go.dev/modernc.org/sqlite) — pure-Go driver, version retractions.
- [golang-migrate/v4 godoc](https://pkg.go.dev/github.com/golang-migrate/migrate/v4) — API frozen, `iofs` source.
- [Vite 8 release announcement](https://vite.dev/blog/announcing-vite8) — Rolldown stable bundler, March 2026.
- [React 19.2 announcement](https://react.dev/blog/2025/10/01/react-19-2) — current stable line.
- [TypeScript 7.0 beta announcement](https://developer.microsoft.com/blog/typescript-7-native-preview-in-visual-studio-2026) — context for sticking on 5.9.
- [Go release history](https://go.dev/doc/devel/release) — current stable + EOL info.
- [DataStation: SQLite in Go, with and without cgo](https://datastation.multiprocess.io/blog/2022-05-12-sqlite-in-go-with-and-without-cgo.html) — performance benchmarks across Go SQLite drivers.
- [Hacker News: I benchmarked six Go SQLite drivers](https://news.ycombinator.com/item?id=38626698) — community sanity-check on perf claims.
- [Wails Issue #4112 — cross-compile Win from Mac broke after 2.10.1](https://github.com/wailsapp/wails/issues/4112) — supports `modernc.org/sqlite` recommendation.
- [Wails Issue #2534 — `ClipboardSetText` on macOS silent failure](https://github.com/wailsapp/wails/issues/2534) — supports pinning to v2.12.0+.
- [Alex Edwards: How to Hash and Verify Passwords with Argon2 in Go](https://www.alexedwards.net/blog/how-to-hash-and-verify-passwords-with-argon2-in-go) — production-quality Argon2id reference.
- [State Management 2025/2026 comparisons (multiple sources)](https://dev.to/hijazi313/state-management-in-2025-when-to-use-context-redux-zustand-or-jotai-2d2k) — Zustand vs Context vs Jotai recommendation.
- [Playwright vs Cypress 2026](https://getautonoma.com/blog/playwright-vs-cypress) — context for E2E recommendation (defer entirely for Wails MVP).
- The exact import path for the pure-Go SQLite database driver in `golang-migrate/v4` (`database/sqlite` vs `database/sqlite3`) — verify in v0.1 implementation. If `mattn/go-sqlite3` reappears in `go.sum`, the wrong subpackage was imported.
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
