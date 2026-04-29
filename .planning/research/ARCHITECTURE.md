# Architecture Research

**Domain:** Local-first desktop password & key vault (Go + Wails v2 + Vite/React/TS/Mantine + SQLite)
**Researched:** 2026-04-29
**Confidence:** HIGH (PRD §2/§5/§6/§10/§18 are locked; Wails idioms verified against official docs)

> **Scope.** The PRD froze the *what* (stack, journal mode, encryption model, AAD strings, error DTO, screens, milestones). This document answers the *how* — directory tree, package boundaries, dependency direction, lock-state machine, AAD assembly point, idle-tracking topology, migration ordering, test pyramid, build-order implications.

---

## 1. System Overview

### 1.1 Layered Component Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│  PRESENTATION (frontend/src) — Vite + React + TS + Mantine SPA           │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌───────────────────────┐  │
│  │ screens/ │  │components│  │   hooks/   │  │ state/                │  │
│  │ (10 PRD) │  │/ (forms, │  │ (useIdle,  │  │ (sessionStore, lock-  │  │
│  │          │  │ countdwn)│  │  useVault) │  │  Status, settings)    │  │
│  └────┬─────┘  └────┬─────┘  └─────┬──────┘  └───────────┬───────────┘  │
│       └─────────────┴──────────────┴────────────────────┘               │
│                              ↓ calls                                    │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ frontend/src/api/  — typed wrappers around generated Wails bindings│ │
│  │  (DTOs from /wailsjs/go/main/App + AppErrorDTO normalization)      │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────┤
│  WAILS BOUNDARY (auto-generated /frontend/wailsjs/)                      │
│  Every call:  TS stub → IPC → Go method → (DTO, error) → TS Promise     │
│  Error contract: errors mapped to AppErrorDTO BEFORE crossing            │
├──────────────────────────────────────────────────────────────────────────┤
│  BINDING LAYER (./app.go) — thin façade, no business logic               │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ App struct: holds ctx + service handles                            │ │
│  │ Each public method: validate input → call service → map err → DTO  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────┤
│  SERVICE LAYER (./internal/*) — owns business logic, no Wails imports   │
│  ┌──────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐ │
│  │  vault   │ │ records │ │generator │ │clipboard │ │ session        │ │
│  │ (open/   │ │ (CRUD,  │ │(passwd,  │ │(copy/    │ │ (lock-state,   │ │
│  │  unlock/ │ │  search,│ │ AES, RSA,│ │ clear,   │ │  idle timer,   │ │
│  │  lock)   │ │ encrypt)│ │ Ed25519) │ │ kind tag)│ │  unlocked_at)  │ │
│  └────┬─────┘ └────┬────┘ └────┬─────┘ └────┬─────┘ └────────┬───────┘ │
│       └───────────┬┴─────────┬┴────────────┬┴─────────────────┘         │
│                   ↓          ↓             ↓                             │
│  ┌──────────┐ ┌─────────┐ ┌──────────┐ ┌──────────────────────────────┐ │
│  │  crypto  │ │  aad    │ │ keyexport│ │ apperr (typed AppError type) │ │
│  │ (KDF,    │ │ (single │ │ (OpenSSH,│ │  + mapping to AppErrorCode   │ │
│  │  AEAD,   │ │  helper │ │  PEM,    │ │                              │ │
│  │  nonce,  │ │  per §  │ │  PKCS#8) │ │                              │ │
│  │  wrap)   │ │  5.4)   │ │          │ │                              │ │
│  └──────────┘ └─────────┘ └──────────┘ └──────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────┤
│  PERSISTENCE LAYER (./internal/storage + ./migrations)                   │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ storage:  *sql.DB wiring, PRAGMA enforcement, repository interfaces│ │
│  │ migrations: golang-migrate driver + embed.FS of /migrations/*.sql  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────┤
│  OS / FILESYSTEM                                                          │
│  SQLite vault file (one user-chosen path) | OS clipboard | export paths  │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Responsibilities

| Component | Responsibility | Owns / Forbidden From |
|-----------|----------------|-----------------------|
| `frontend/src/screens/` | 10 PRD screens, route-level composition | No direct calls to `/wailsjs/`; goes through `api/` |
| `frontend/src/components/` | Reusable UI (CountdownBar, MaskedField, RecordForm, PrivateKeyWarningModal) | No business logic, no crypto |
| `frontend/src/hooks/` | `useIdleTracker`, `useVaultSession`, `useClipboardCountdown` | No SQLite, no fetch |
| `frontend/src/state/` | Zustand/Jotai store for `{lockStatus, vaultId, vaultPath, settings}` | Never holds decrypted secrets long-term |
| `frontend/src/api/` | Typed wrappers + error normalization + retry/lock-redirect | Single import boundary for `/wailsjs/go/main/App` |
| `frontend/src/errors/` | `AppErrorCode → user message` map + toast dispatcher | No string-matching on `error.message` |
| `app.go` | Wails `Bind` target. Public methods 1:1 with PRD §10 contract. | No SQL, no crypto primitives, no FS writes |
| `main.go` | `wails.Run(...)` config: window size, embed of `frontend/dist`, `Bind: []any{app}` | No app logic |
| `internal/vault` | Vault lifecycle: `Create`, `Open`, `Unlock`, `Lock`, `ChangeMasterPassword` | Owns the in-memory `Session` (DEK + metadata) |
| `internal/records` | Record CRUD; calls crypto + AAD; validates per §11.5 | Holds **no** decrypted record cache (see §3) |
| `internal/crypto` | Argon2id KDF, AES-GCM encrypt/decrypt, nonce gen, key-wrap | Pure functions; no DB, no logging of secrets |
| `internal/aad` | **One** function: `Build(kind, vaultID, recordID, recordType, dekID, envelopeVersion)` | Forbidden from importing `schema_version` |
| `internal/generator` | Password / AES / RSA / Ed25519 generation via `crypto/rand` | No `math/rand` (lint-enforced) |
| `internal/keyexport` | OpenSSH (`x/crypto/ssh`), PEM, PKCS#8 marshaling | Read-only; no DB |
| `internal/clipboard` | `Write(value, kind)`, `Clear()` via Wails runtime or `golang.design/x/clipboard` | Does not own the timeout — frontend countdown is the canonical source |
| `internal/session` | Lock-state machine, idle timer goroutine, `Activity()` ping handler | Authoritative `IsUnlocked()` check; all secret methods consult it |
| `internal/storage` | `*sql.DB` open, `PRAGMA application_id` check, `journal_mode=DELETE`, `foreign_keys=ON`, repository structs | No domain logic; no encryption |
| `internal/storage/migrations` | `golang-migrate` runner + `embed.FS` of `/migrations` | Fail-closed on dirty state |
| `internal/apperr` | `AppError{Code, Message, Cause}` Go type + `ToDTO()` | Single point of error sanitization |
| `internal/logger` | `slog` wrapper with sensitive-field redaction list | Forbidden from logging `password`, `dek`, `kek`, `value`, `secret`, `private_key` keys |

---

## 2. Recommended Project Structure

### 2.1 Top-Level Tree

```
abyss/
├── main.go                          # Wails entrypoint: options, embed, Bind
├── app.go                           # App struct + bound public methods (§10 contract)
├── go.mod / go.sum
├── wails.json                       # Wails build config
├── build/                           # Wails-managed build artifacts (icons, plists, NSIS)
│
├── internal/                        # All Go business logic; not bound to frontend directly
│   ├── apperr/
│   │   ├── codes.go                 # AppErrorCode constants (mirror of TS enum §10.1)
│   │   ├── error.go                 # AppError struct + ToDTO()
│   │   └── error_test.go
│   ├── logger/
│   │   ├── logger.go                # slog wrapper, redactor
│   │   └── logger_test.go
│   ├── crypto/
│   │   ├── kdf.go                   # Argon2id with profile + fallback (§5.3)
│   │   ├── aead.go                  # AES-256-GCM encrypt/decrypt
│   │   ├── nonce.go                 # crypto/rand nonce generator
│   │   ├── keywrap.go               # wrap/unwrap DEK
│   │   ├── kdf_test.go
│   │   ├── aead_test.go             # incl. AAD-tampering golden vectors
│   │   ├── nonce_test.go
│   │   └── keywrap_test.go
│   ├── aad/
│   │   ├── aad.go                   # ONE Build() helper for all 3 AAD shapes (§5.4)
│   │   └── aad_test.go              # golden vectors for metadata/secret/wrapped_dek
│   ├── storage/
│   │   ├── db.go                    # Open(), pragmas, application_id check
│   │   ├── repo_records.go          # records table CRUD (raw bytes only)
│   │   ├── repo_vault.go            # vault_metadata, deks, key_wrappings
│   │   ├── repo_settings.go
│   │   ├── migrations/
│   │   │   ├── migrate.go           # golang-migrate runner + embed.FS
│   │   │   └── migrate_test.go
│   │   └── db_test.go
│   ├── vault/
│   │   ├── service.go               # Create/Open/Unlock/Lock/ChangeMasterPassword
│   │   ├── service_test.go
│   │   └── doc.go                   # Package-level state diagram (godoc)
│   ├── session/
│   │   ├── session.go               # In-memory Session struct (DEK, vaultID, unlockedAt)
│   │   ├── lockstate.go             # State machine + transitions
│   │   ├── idle.go                  # Idle timer goroutine
│   │   └── session_test.go
│   ├── records/
│   │   ├── service.go               # CRUD + encrypt/decrypt orchestration
│   │   ├── validation.go            # §11.5 record validation
│   │   ├── search.go                # In-memory search after unlock
│   │   ├── types.go                 # Per-record-type metadata/secret schemas
│   │   └── service_test.go
│   ├── generator/
│   │   ├── password.go              # §11.2
│   │   ├── aes.go                   # §11.3
│   │   ├── rsa.go                   # §11.4
│   │   ├── ed25519.go
│   │   └── *_test.go
│   ├── keyexport/
│   │   ├── openssh.go               # x/crypto/ssh marshaling
│   │   ├── pem.go                   # PEM + PKCS#8
│   │   └── *_test.go
│   └── clipboard/
│       ├── clipboard.go             # Write(value, kind), Clear()
│       └── clipboard_test.go        # interface-based; OS impl behind build tags if needed
│
├── migrations/                      # golang-migrate SQL files; embedded by storage/migrations
│   ├── 0001_init.up.sql             # vault_metadata, deks, key_wrappings, records, settings
│   ├── 0001_init.down.sql
│   └── ...                          # Future migrations
│
└── frontend/                        # Vite SPA — generated by `wails init -t react-ts`, then extended
    ├── package.json
    ├── vite.config.ts
    ├── tsconfig.json
    ├── index.html
    ├── wailsjs/                     # AUTO-GENERATED — do not edit; gitignore'd or tracked
    │   ├── go/
    │   │   └── main/
    │   │       └── App.{js,d.ts}
    │   └── runtime/
    └── src/
        ├── main.tsx                 # MantineProvider + Router root
        ├── App.tsx                  # Top-level layout + lock-state gate
        ├── api/
        │   ├── index.ts             # Re-exports typed wrappers
        │   ├── vault.ts             # createVault, unlockVault, lockVault, ...
        │   ├── records.ts
        │   ├── generator.ts
        │   ├── clipboard.ts
        │   ├── exporter.ts
        │   └── errors.ts            # normalizeError(unknown) → AppErrorDTO
        ├── state/
        │   ├── sessionStore.ts      # Zustand: {lockStatus, vaultId, vaultPath, ...}
        │   ├── settingsStore.ts
        │   └── clipboardStore.ts    # countdown state (ttl, startedAt, secretKind)
        ├── hooks/
        │   ├── useVaultSession.ts
        │   ├── useIdleTracker.ts    # mouse/keyboard/nav listeners → backend ping
        │   ├── useClipboardCountdown.ts
        │   └── useLockGuard.ts      # Redirects to Unlock screen if locked
        ├── components/
        │   ├── CountdownBar/
        │   ├── MaskedField/
        │   ├── RecordForm/          # type-discriminated subforms
        │   ├── PrivateKeyWarningModal/
        │   └── LockBadge/
        ├── screens/
        │   ├── Welcome/
        │   ├── CreateVault/
        │   ├── UnlockVault/
        │   ├── VaultHome/
        │   ├── RecordDetail/
        │   ├── RecordEdit/
        │   ├── PasswordGenerator/
        │   ├── KeyGenerator/
        │   ├── Settings/
        │   └── About/
        ├── errors/
        │   ├── codes.ts             # AppErrorCode (mirror of Go)
        │   ├── messages.ts          # Code → user-facing string
        │   └── handler.ts           # Unified error→toast→redirect
        └── types/
            └── dto.ts               # All DTO types from §10 (single source for FE)
```

### 2.2 Critique of the Proposed Tree (from prompt)

The prompt-suggested tree was nearly correct. Adjustments made:

| Change | Rationale |
|--------|-----------|
| Split `internal/errors/` into `internal/apperr/` | "errors" shadows the stdlib `errors` package, causing import friction. |
| Added `internal/session/` separate from `internal/vault/` | The lock-state machine and idle timer have a different lifecycle than vault open/close; coupling them makes both harder to test. `vault` calls `session` to mutate state. |
| Added `internal/aad/` as its own package | PRD §5.4 mandates a single AAD format. A package boundary makes accidental in-line AAD construction a compiler error (must import the helper). |
| Added `internal/keyexport/` separate from `generator/` | Generation and export are distinct lifecycles: generators produce *new* material; exporters serialize *stored* material. Different test surfaces. |
| Added `internal/logger/` | Sanitized logging is cross-cutting (PRD §12.2). Centralizing the redaction list prevents drift. |
| Moved migrations driver into `internal/storage/migrations/` and SQL files to `/migrations/` at repo root | Two reasons: (a) `embed.FS` can reach the sibling directory cleanly; (b) keeping SQL alongside Go means schema diffs are visible without filesystem tricks. |
| Added `frontend/src/types/dto.ts` | The auto-generated `wailsjs/go/main/App.d.ts` is generated from struct tags. A hand-curated `dto.ts` is the canonical type surface for FE code, so changes to Go structs surface as TS errors at the api/ wrapper layer rather than 50 places downstream. |
| Added `frontend/src/state/clipboardStore.ts` | Frontend owns countdown UI per PRD §2.7; state must persist across screen switches. |

### 2.3 Structure Rationale

- **`/internal/`** — Go's `internal/` convention prevents external import. Since this is a single binary, every package goes there except `app.go`/`main.go`.
- **One package per concern** — `crypto` ≠ `aad` ≠ `keyexport`. Tight packages make tests fast and dependency cycles impossible.
- **Repository pattern in `storage/`** — services depend on repo interfaces, not `*sql.DB`. Tests use either a real tmp SQLite file (preferred for crypto-touching tests) or an interface mock.
- **Frontend `api/` as a single seam** — every Wails call goes through one of ~6 files. If the binding contract changes, the blast radius is small.

---

## 3. Where State Lives

| State | Lifetime | Storage | Cleared On |
|-------|----------|---------|-----------|
| Master password | Microseconds (during KEK derivation) | Local var, then KDF input | Returned to caller; KDF zeros internal buffer; Go strings are immutable so byte-slice variant is preferred at the binding boundary |
| KEK (32 bytes) | Microseconds (during unwrap) | `[]byte` local | Zeroed immediately after `aead.Open()` returns the DEK |
| **DEK (32 bytes)** | Until lock | `Session.dek []byte` (in-memory only) | Zeroed on `Lock()`, `AutoLock()`, app shutdown, change-password completion |
| Session metadata | Until lock | `Session{vaultID, vaultPath, unlockedAt, autoLockSeconds}` | Cleared on lock |
| Decrypted records | **Per-call only — no cache** | Returned to frontend, freed by GC after stack frame ends | N/A — never cached server-side |
| Frontend record list | Until lock or refresh | React state via `useState`/store | Cleared on lock event from backend |
| Idle timer | Until lock | `*time.Timer` in `session` package | Stopped + reset to nil on lock |
| Clipboard timer | Until expiry, manual clear, or lock | Frontend store + backend `*time.Timer` | Both cleared on copy-new-secret, manual clear, lock |
| Settings (clipboard timeout, autolock minutes, theme) | Persistent | `settings` table | App lifetime |
| Tags / encrypted metadata | Until lock | Decrypted on read; not cached | N/A |
| `vault_metadata`, `deks`, `key_wrappings` | Persistent | SQLite | App lifetime (encrypted at rest where applicable) |

### 3.1 The "No Decrypted Record Cache" Decision

**Trade-off considered:** A bounded LRU of decrypted records would speed up search and re-open. But:
- It widens the memory-exposure window (PRD §4.3).
- It complicates lock semantics ("did we clear all caches?" becomes a question).
- For ≤ a few thousand records (the realistic personal vault size), per-request decryption is fast (~<1 ms per record on a modern machine).

**Decision:** Decrypt-on-demand. `ListRecords()` decrypts only metadata (already small); `GetRecord(id)` decrypts metadata + secret. Search uses the decrypted-metadata path with a one-shot pass; future optimization can introduce a search-only metadata cache that lives in `internal/records/search.go` and is explicitly cleared on `Lock()`.

### 3.2 Go Memory Caveat

Per PRD §4.4: byte slices are zeroed where practical (`for i := range buf { buf[i] = 0 }` after use), but Go strings (which is what Wails passes for `masterPassword string`) cannot be reliably zeroed. The mitigation is documented in the README, not engineered around. **Where practical**, methods that take a master password should immediately copy the string into a `[]byte` and pass that down — at least the in-process post-binding handling avoids long-lived string references.

---

## 4. Frontend ↔ Backend Boundary

### 4.1 The Three Rules (PRD §10)

1. **No SQLite, crypto, or FS writes from frontend.** Lint rule: `frontend/src/**` must not import `better-sqlite3`, `crypto-js`, `node:fs`, etc. (Vite's browser target enforces most of this; an ESLint custom rule covers the rest.)
2. **Errors are mapped to `AppErrorDTO` before crossing.** No raw Go errors leak. Every method on `app.go` ends with `return dto, apperr.ToDTO(err)`.
3. **No string-matching errors in frontend.** All branching uses `code: AppErrorCode`.

### 4.2 The `frontend/src/api/` Wrapper Layer

```typescript
// frontend/src/api/vault.ts
import { CreateVault as RawCreateVault } from "../../wailsjs/go/main/App";
import type { CreateVaultRequest, VaultSessionDTO } from "../types/dto";
import { normalizeError } from "./errors";

export async function createVault(
  req: CreateVaultRequest
): Promise<VaultSessionDTO> {
  try {
    return (await RawCreateVault(req)) as VaultSessionDTO;
  } catch (e) {
    throw normalizeError(e);  // returns a typed AppErrorDTO
  }
}
```

```typescript
// frontend/src/api/errors.ts
import type { AppErrorDTO, AppErrorCode } from "../errors/codes";

export function normalizeError(e: unknown): AppErrorDTO {
  if (typeof e === "object" && e !== null && "code" in e) {
    return e as AppErrorDTO;
  }
  // Wails sometimes wraps errors as {message: string} — fall back to INTERNAL_ERROR
  return { code: "INTERNAL_ERROR", message: String(e) };
}
```

**Why this matters:** if the Go side ever fails to map an error (bug), the frontend still receives a typed object. The handler can also centralize behaviors like "on `VAULT_LOCKED`, redirect to Unlock screen."

### 4.3 The Hand-Curated `types/dto.ts`

The auto-generated `wailsjs/go/main/App.d.ts` is functional but driven by Go struct tags. A hand-curated `types/dto.ts` (mirroring PRD §10) gives:
- **Stricter union types** for `RecordType`, `AppErrorCode`, `secretKind` — Go's string-typed fields become narrow TS unions.
- **Discriminated unions** for `RecordDTO.metadata` based on `type` (a future improvement; MVP can use `Record<string, unknown>`).
- **A clear failure point** when the binding contract drifts: the `api/` wrapper file fails to compile.

---

## 5. Lock-State Machine

### 5.1 States and Transitions

```
                   ┌─────────────┐
                   │   closed    │ ◄────────────────────────┐
                   │ (no vault   │                          │
                   │  selected)  │                          │
                   └──────┬──────┘                          │
                          │ OpenVault(path)                 │
                          │  - check application_id         │
                          │  - run migrations (or fail)     │
                          ↓                                 │
                   ┌─────────────┐                          │
                   │   opening   │ ──── migration error ───→│
                   │ (DB open,   │     application_id error │
                   │  migrating) │                          │
                   └──────┬──────┘                          │
                          │ ok → schema current             │
                          ↓                                 │
                   ┌─────────────┐                          │
        ┌─────────►│   locked    │                          │
        │          │ (DB open,   │ ◄──────────────────┐     │
        │          │  no DEK)    │                    │     │
        │          └──────┬──────┘                    │     │
        │                 │ UnlockVault(password)     │     │
        │                 │  - derive KEK             │     │
        │                 │  - unwrap DEK             │     │
        │                 ↓                           │     │
        │          ┌─────────────┐                    │     │
        │          │  unlocking  │ ── unwrap fail ───→┘     │
        │          │ (KDF + AEAD)│   (UNLOCK_FAILED)        │
        │          └──────┬──────┘                          │
        │                 │ ok                              │
        │                 ↓                                 │
        │          ┌─────────────┐                          │
        │          │  unlocked   │ ◄──── Activity() ──┐     │
        │          │ (DEK in mem,│                    │     │
        │          │  idle timer)│                    │     │
        │          └──────┬──────┘                    │     │
        │                 │ LockVault() | idle expiry │     │
        │                 ↓ | ChangeMasterPassword    │     │
        │          ┌─────────────┐                    │     │
        │          │   locking   │ ───────────────────┘     │
        │          │ (zero DEK,  │                          │
        │          │  clear cb)  │                          │
        │          └──────┬──────┘                          │
        └─────────────────┘                                 │
                          │ CloseVault() (or app exit)      │
                          └─────────────────────────────────┘
```

### 5.2 Where Each Transition Is Enforced

| Transition | Frontend Hint | Backend Authoritative Check |
|------------|---------------|----------------------------|
| Show "Unlocked" badge | `sessionStore.lockStatus` | `GetVaultStatus()` is the truth |
| Block secret action | Disable button if locked | **Every secret-touching method calls `session.RequireUnlocked()` first**; returns `VAULT_LOCKED` if not |
| Auto-lock after idle | Frontend pings `Activity()` on user input | Backend timer fires `Lock()` |
| Lock on app exit | Window close handler dispatches | `OnShutdown` hook calls `vault.Lock()` |
| Refuse to unlock already-unlocked | Hide unlock screen | `VAULT_ALREADY_UNLOCKED` error from `UnlockVault` |

**The non-negotiable rule:** the frontend's lock state is *advisory*. The backend's `session` package is the single source of truth. Every Wails-bound method that touches secrets begins with:

```go
func (a *App) GetRecord(id string) (RecordDTO, error) {
    if err := a.session.RequireUnlocked(); err != nil {
        return RecordDTO{}, apperr.ToDTO(err)  // VAULT_LOCKED
    }
    rec, err := a.records.Get(id)
    return rec, apperr.ToDTO(err)
}
```

---

## 6. Idle-Tracking Architecture

### 6.1 The Topology

```
Frontend                                 Backend
────────                                 ───────
[User input]                             ┌─────────────────────┐
   ↓                                     │ session.Service     │
[useIdleTracker]                         │  - autoLockSeconds  │
   ↓ debounced (e.g. 1s)                 │  - timer *time.Timer│
api.session.Activity()      ─────────►   │  - mu sync.Mutex    │
                                         │                     │
                                         │ Activity():         │
                                         │   timer.Reset(d)    │
                                         │                     │
                                         │ on timer fire:      │
                                         │   -> vault.Lock()   │
                                         │   -> emit "locked"  │
                                         └──────────┬──────────┘
                                                    │ Wails event
                                         ◄──────────┘
[sessionStore.lockStatus = "locked"]
[clear local record state]
[redirect to /unlock]
```

### 6.2 Why Frontend-Driven (Plus Backend Authoritative)

**Pure frontend timer:** trusts the renderer; if the renderer dies, no lock.
**Pure backend timer with no input from FE:** can't distinguish "user idle" from "user actively reading a record without keyboard input."
**Hybrid (chosen):** FE detects activity events (mouse, keyboard, navigation, view, edit, copy, generator interaction per PRD §13.2) and pings backend; backend owns the actual countdown and the `Lock()` call.

**Trade-off acknowledged:** This trusts the renderer to faithfully report activity. A compromised renderer could spam `Activity()` to keep the vault unlocked indefinitely. Per PRD §4.2, malware on the user's machine is **explicitly out of scope** — so this trade-off is acceptable.

### 6.3 Key Design Choices

- **Debounce frontend pings to ≥1s.** Otherwise typing is hundreds of IPC calls per minute.
- **Backend emits `vault:locked` event via `runtime.EventsEmit`.** Frontend subscribes once at App root.
- **No FE timer for the actual lock.** FE can show "auto-lock in 2:34" using `unlockedAt + autoLockSeconds - now`, but this is purely advisory.
- **Sleep / lid-close detection** is Should-have (PRD §13.4), deferred past v0.1. Implementation would use OS-specific hooks (e.g., `IOKit` on macOS); skip for MVP and document.

---

## 7. Migration Ordering and Recovery

### 7.1 The Sequence at Vault Open

```
OpenVault(path) →
  1. os.Stat(path)                       // exists?
  2. sql.Open("sqlite3", path)
  3. SELECT application_id FROM pragma_application_id
       expect 0x4E56544E                  // else: UNSUPPORTED_VAULT
  4. PRAGMA journal_mode = DELETE
  5. PRAGMA foreign_keys = ON
  6. migrate.New(embed.FS, sqlite3-driver)
  7. m.Up()                               // apply pending migrations
       on dirty: return MIGRATION_FAILED  // fail-closed; no auto-fix
  8. SELECT vault_id, schema_version FROM vault_metadata
  9. transition: closed → locked
```

### 7.2 Dirty State Handling (PRD §2.5: "fail-closed on dirty migration state")

`golang-migrate` exposes `migrate.ErrDirty`. On detection:

- **Do not auto-force.** Forcing a version on a possibly half-applied migration risks corruption.
- Return `MIGRATION_FAILED` with `details: {schema_version: <v>, dirty: true}` (still safe — no secret leakage).
- Frontend shows: "This vault appears to have an incomplete schema upgrade. Restore from backup or contact support." (No support exists for a learning project; the message is honest.)

A **manual** repair tool could be a future addition (`abyss-repair --force-version=N`), but it is out of MVP.

### 7.3 Migration Tests

In `internal/storage/migrations/migrate_test.go`:
- **Fresh DB:** apply all up migrations; assert tables/PRAGMAs.
- **Round-trip:** up → down → up; assert idempotence.
- **Embed integrity:** `embed.FS` enumerates exactly the expected files (catches missing migration files in releases).
- **Dirty simulation:** force `schema_migrations` to `dirty=1`, assert `MIGRATION_FAILED`.

Run against a real SQLite tmp file (`t.TempDir()`), not an in-memory DB — the production code path uses a file, and `application_id` PRAGMA semantics differ slightly between memory and file.

---

## 8. AAD Discipline (PRD §5.4)

### 8.1 Single Helper, Three Shapes

```go
// internal/aad/aad.go
package aad

import "fmt"

type Kind int

const (
    KindRecordMetadata Kind = iota
    KindRecordSecret
    KindWrappedDEK
)

// Build assembles AAD per PRD §5.4. The format is canonical and tested with golden vectors.
// schema_version is intentionally NOT included (PRD §5.4): schema migrations must
// not force record re-encryption unless record_envelope_version changes.
func Build(k Kind, vaultID, recordID, recordType, dekID, wrappingType string, envVersion int) ([]byte, error) {
    switch k {
    case KindRecordMetadata:
        return []byte(fmt.Sprintf("%s|%d|%s|%s|%s|metadata",
            vaultID, envVersion, recordID, recordType, dekID)), nil
    case KindRecordSecret:
        return []byte(fmt.Sprintf("%s|%d|%s|%s|%s|secret",
            vaultID, envVersion, recordID, recordType, dekID)), nil
    case KindWrappedDEK:
        return []byte(fmt.Sprintf("%s|%s|%s|wrapped_dek",
            vaultID, dekID, wrappingType)), nil
    default:
        return nil, fmt.Errorf("aad: unknown kind %d", k)
    }
}
```

### 8.2 Why a Package, Not a Function

- **Compile-time enforceability.** Other packages can only construct AAD by calling `aad.Build(...)`. A grep for raw `[]byte("...|metadata")` strings in the codebase becomes a CI check that fails the build.
- **One place to test.** `aad_test.go` carries the golden vectors. If a future change accidentally swaps field order, the golden vector breaks before any production data is written.
- **Auditable.** A reviewer reads exactly one file to understand AAD. The PRD §5.4 spec lives next to the code that implements it (also as a doc comment).

### 8.3 Tampering Test (PRD §14.1)

```go
func TestAEAD_AADTampering(t *testing.T) {
    dek := mustGenKey()
    aadA, _ := aad.Build(aad.KindRecordSecret, "vault1", "rec_A", "credential", "dek1", "", 1)
    aadB, _ := aad.Build(aad.KindRecordSecret, "vault1", "rec_B", "credential", "dek1", "", 1)
    ct, nonce := mustEncrypt(dek, []byte("password=hunter2"), aadA)
    _, err := crypto.Decrypt(dek, ct, nonce, aadB)  // wrong AAD
    if err == nil { t.Fatal("expected AAD mismatch failure") }
}
```

This is the canonical "record-swap detection" test (PRD acceptance scenario "AAD Tampering Detection").

---

## 9. Test Pyramid (mapped to architectural layers)

```
                     ┌─────────────────────────────────────┐
                     │ Manual cross-platform matrix (§14.3)│  ← milestone gate
                     │  macOS + Windows, 22 scenarios      │
                     └─────────────────────────────────────┘
                  ┌──────────────────────────────────────────┐
                  │ Integration tests (Go)                   │
                  │  - real SQLite tmp file                  │
                  │  - vault.Service end-to-end              │
                  │  - migrations up/down/dirty              │
                  └──────────────────────────────────────────┘
              ┌────────────────────────────────────────────────┐
              │ Frontend component tests (Vitest + RTL)        │
              │  - form validation                             │
              │  - countdown component                         │
              │  - locked-state action blocking                │
              │  - private key warning modal                   │
              └────────────────────────────────────────────────┘
       ┌──────────────────────────────────────────────────────────┐
       │ Go unit tests (the meat)                                 │
       │  - crypto: KDF, AEAD, nonce, keywrap, AAD vectors        │
       │  - records: encrypt/decrypt, validation, search          │
       │  - generator: password classes, AES sizes, RSA, Ed25519  │
       │  - keyexport: OpenSSH, PEM, PKCS#8 round-trips           │
       │  - apperr: error mapping                                 │
       └──────────────────────────────────────────────────────────┘
```

### 9.1 Coverage Targets by Layer

| Layer | Target | Rationale |
|-------|--------|-----------|
| `internal/crypto`, `internal/aad` | ≥ 95% with golden vectors | Highest-risk code; cheap to test |
| `internal/generator`, `internal/keyexport` | ≥ 90% | Pure functions; easy to test |
| `internal/records`, `internal/vault` | ≥ 80% | Integration via real SQLite |
| `internal/session`, `internal/clipboard` | ≥ 70% | Timer & OS-call boundaries are stub-friendly |
| `app.go` | ≥ 60% | Thin façade; main goal is "errors are mapped, unlocked-check is present" |
| Frontend components | Critical paths only | Per PRD §14.2 list |

### 9.2 What NOT to Unit-Test

- The Wails IPC layer itself (covered by manual cross-platform matrix).
- OS clipboard internals (best-effort by design; document and move on).
- `golang-migrate` internals (trust the library; test our migration files via `m.Up()`).

---

## 10. Build-Order Implications

### 10.1 Mapping PRD §20 (20-step sequence) to Modules

| Step | Modules Created/Touched | Milestone |
|------|-------------------------|-----------|
| 1. Wails + Vite + Mantine skeleton | `main.go`, `app.go` (empty), `frontend/` scaffold | v0.1 |
| 2. SQLite + migrations | `internal/storage/`, `migrations/0001_init.*.sql` | v0.1 |
| 3. Vault metadata, deks, key_wrappings | `migrations/0001_init.up.sql`, `internal/storage/repo_vault.go` | v0.1 |
| 4. Argon2id KDF | `internal/crypto/kdf.go` + tests | v0.1 |
| 5. DEK gen and wrapping | `internal/crypto/keywrap.go`, `internal/crypto/aead.go`, `internal/aad/aad.go` | v0.1 |
| 6. Vault create/unlock/lock | `internal/vault/`, `internal/session/`, `app.go` (vault methods) | v0.1 |
| 7. Credential CRUD | `internal/records/` (credential type only), `app.go` (record methods), `frontend/screens/RecordEdit/` | v0.1 |
| 8. Record encryption with AAD | (already done in step 5; this step wires records through it) | v0.1 |
| 9. Password generator | `internal/generator/password.go`, `frontend/screens/PasswordGenerator/` | v0.1 |
| 10. Clipboard copy/countdown | `internal/clipboard/`, `frontend/components/CountdownBar/`, `frontend/state/clipboardStore.ts` | v0.1 |
| 11. Auto-lock | `internal/session/idle.go`, `frontend/hooks/useIdleTracker.ts` | v0.2 |
| 12. Change master password | `internal/vault/service.go` (extend) | v0.2 |
| 13. API key + secure note records | `internal/records/types.go` (extend), `frontend/components/RecordForm/` (subforms) | v0.3 |
| 14. AES key generation | `internal/generator/aes.go`, `frontend/screens/KeyGenerator/` | v0.4 |
| 15. RSA key generation | `internal/generator/rsa.go`, `internal/keyexport/` | v0.4 |
| 16. Ed25519 key generation | `internal/generator/ed25519.go` | v0.4 |
| 17. Public/private key export | `internal/keyexport/`, `frontend/components/PrivateKeyWarningModal/` | v0.4 |
| 18. Cross-platform hardening | All; manual test matrix | v0.5 |
| 19. README and threat model | `README.md`, `docs/` | v1.0 |
| 20. Final macOS + Windows test pass | All | v1.0 |

### 10.2 Cross-Cutting Concerns That Span Multiple Phases

These must exist from v0.1 even though they're "polished" in v0.2:

| Concern | Stub by | Polish by | Risk if Deferred |
|---------|---------|-----------|------------------|
| `internal/apperr` typed errors | v0.1 (basic codes) | v0.2 (full enum, sanitization) | Refactor every Wails method later if added late |
| `internal/aad` helper | v0.1 (used by step 5) | v0.2 (golden vectors, tampering tests) | If AAD is constructed inline first, every record written has wrong AAD format |
| `internal/logger` redaction | v0.1 (basic redactor) | v0.2 (fuzz tests) | Logged secrets in early dogfooding are unrecoverable |
| Lock-state enforcement | v0.1 (basic `RequireUnlocked()`) | v0.2 (auto-lock integration) | Bound methods that don't check unlock are an audit liability |
| Error normalization in `frontend/src/api/` | v0.1 | v0.2 | Frontend code grows expecting `error.message` strings, painful to refactor |

### 10.3 Module Dependency Order (build this way)

```
apperr  ──┐
logger  ──┤
crypto  ──┤── aad ──┐
        ──┘         │
storage ──┐         ├── records ──┐
          ├── session              ├── vault ── app.go ── main.go
          │                        │
generator ─┘─── keyexport ─────────┤
clipboard ─────────────────────────┘
```

Anything to the left has zero imports from anything to the right. `app.go` imports everything; `main.go` only imports `app` and Wails options.

### 10.4 Frontend Module Dependency Order

```
types/dto.ts ─── errors/codes.ts ─── api/errors.ts ─── api/{vault,records,...}.ts
                                                              ↓
                                              state/* ─── hooks/* ─── components/* ─── screens/* ─── App.tsx ─── main.tsx
```

The `api/` layer must exist before any component uses it. `state/` must exist before screens that subscribe to lock status.

---

## 11. Anti-Patterns

### 11.1 Anti-Pattern: Fat `app.go`

**What people do:** Put SQL queries, crypto calls, and validation directly in `app.go` methods.
**Why it's wrong:** Every method becomes untestable without spinning up Wails. Cross-cutting concerns (error mapping, unlocked check) are duplicated by hand and drift.
**Do this instead:** `app.go` methods are 5–10 lines: validate input, call service, map error, return DTO. Real logic lives in `internal/`.

### 11.2 Anti-Pattern: Inline AAD Construction

**What people do:** Each call site formats `[]byte(vaultID + "|" + recordID + ...)`.
**Why it's wrong:** Field order, separator choice, or version number drift creates AAD that no longer matches what was used at encrypt-time. Records become permanently undecryptable on the next read.
**Do this instead:** `aad.Build(...)` is the *only* way to produce AAD. Lint the codebase for `[]byte(.*|metadata)` patterns and reject in CI.

### 11.3 Anti-Pattern: Caching Decrypted Records on the Backend

**What people do:** Add a "performance" map of `id → decryptedRecord` to speed up search.
**Why it's wrong:** Widens memory exposure (PRD §4.3), complicates lock semantics, masks bugs in the encrypt/decrypt path.
**Do this instead:** Decrypt on demand. If search needs to be faster, build a metadata-only cache in `records/search.go` that is *explicitly* cleared in `Lock()` and is documented.

### 11.4 Anti-Pattern: Frontend Owning Lock State Authoritatively

**What people do:** Keep `isUnlocked` only in a frontend store; backend trusts it.
**Why it's wrong:** A bug or stale state in the frontend allows record reads when the backend should block them. The PRD's threat model explicitly distrusts the renderer for security checks.
**Do this instead:** Frontend `lockStatus` is advisory (for UI). Every secret-touching backend method calls `session.RequireUnlocked()` first.

### 11.5 Anti-Pattern: String-Matching Errors in Frontend

**What people do:** `if (err.message.includes("locked"))`.
**Why it's wrong:** Drift, i18n breakage, security info leakage if Go error strings change.
**Do this instead:** `if (err.code === "VAULT_LOCKED")`. PRD §10.1 mandates this.

### 11.6 Anti-Pattern: Including `schema_version` in AAD

**What people do:** "Why not bind ciphertext to schema version?"
**Why it's wrong:** Ordinary schema migrations (e.g., adding a `last_used_at` column) would force re-encryption of every record. The PRD explicitly forbids this in §5.4.
**Do this instead:** AAD binds to `record_envelope_version`, which only bumps when the *record envelope itself* changes (e.g., adding a new field inside `encrypted_metadata_blob`).

### 11.7 Anti-Pattern: Logging Master Password / KEK / DEK on Error Paths

**What people do:** `log.Printf("unlock failed for password=%s", password)` during debugging.
**Why it's wrong:** Logs are forever. PRD §12.2 forbids this; CI must lint for it.
**Do this instead:** `internal/logger` has a redaction list; any field named `password`, `secret`, `dek`, `kek`, `value`, `private_key`, `api_key` is replaced with `<redacted>`. Code review checks for `slog.Any("...", anything-sensitive)`.

### 11.8 Anti-Pattern: WAL Mode "for Performance"

**What people do:** Enable `PRAGMA journal_mode = WAL` to speed up writes.
**Why it's wrong:** PRD §2.4 forbids WAL: vault portability matters more than throughput. Sidecar `-wal` and `-shm` files break the "single-file portable vault" promise.
**Do this instead:** `journal_mode = DELETE`. Single file, copyable, zip-able.

---

## 12. Integration Points

### 12.1 External Boundaries

| Boundary | Pattern | Notes |
|----------|---------|-------|
| OS Clipboard | `internal/clipboard` wraps Wails runtime or `golang.design/x/clipboard` | Best-effort; document third-party clipboard manager limitations (PRD §19) |
| Filesystem (vault file) | `internal/storage` opens user-chosen path | Validate `application_id` before any other operation |
| Filesystem (key export paths) | `internal/keyexport` writes user-chosen destination | Verify unlock state and warning-confirmation flag before write |
| Wails runtime events | `runtime.EventsEmit("vault:locked", ...)` from session, listened to by frontend | Used for backend-driven lock notifications |

### 12.2 Internal Module Boundaries

| Boundary | Communication | Considerations |
|----------|---------------|----------------|
| `app.go ↔ services` | Direct method calls | `app.go` is the only place service methods are wired together |
| `services ↔ storage` | Repository interfaces | Allows mock storage in tests; real `*sql.DB` in production |
| `services ↔ crypto` | Pure function calls | No state; no DB access from `crypto` |
| `vault ↔ session` | `session` is an injected dependency of `vault.Service` | `vault.Lock()` calls `session.Clear()`; `vault.Unlock()` calls `session.Activate(dek, vaultID, ...)` |
| `records ↔ session` | `records.Service` calls `session.RequireUnlocked()` and reads `session.DEK()` | All record methods are gated; no public DEK accessor outside `session` |
| Frontend `state ↔ api` | `api/` calls update store via Wails events or after method success | Lock state synced via `vault:locked` event, not polling |

---

## 13. Scaling Considerations (Realistic for a Personal Vault)

| Scale | Adjustment |
|-------|-----------|
| 1–500 records | No changes needed. Decrypt-on-read is fast. Search is in-memory linear scan. |
| 500–5,000 records | Add metadata-only cache in `records/search.go`, cleared on lock. Index `record_type` in SQLite. |
| 5,000+ records | Outside MVP intent (this is a *personal* vault). If it happens: paginate the record list UI; consider FTS5 over a metadata-only cache. |

**First bottleneck:** Frontend rendering of large record lists. Mantine's `<Table>` virtualization solves this.
**Second bottleneck:** Argon2id memory profile on a low-RAM machine. PRD §5.3 already addresses with the documented fallback profile.

**The PRD does not envision this app being scaled.** Scaling is out of scope; correctness is in scope.

---

## 14. Summary: Cross-Cutting Concerns Checklist

For the roadmap to wire correctly, these must be addressed *across* phases, not within a single one:

- [ ] **Typed errors (`internal/apperr` + `frontend/src/errors/`)** — established v0.1, exhaustive by v0.2.
- [ ] **AAD helper (`internal/aad`)** — established v0.1 (must exist before first encrypt), golden-vector-tested v0.2.
- [ ] **Sanitized logging (`internal/logger`)** — established v0.1, lint-enforced v0.2.
- [ ] **Lock-state enforcement (`session.RequireUnlocked()`)** — every secret-touching method from v0.1; auto-lock layered in v0.2.
- [ ] **Frontend `api/` wrapper layer** — established v0.1; all components route through it.
- [ ] **Migration fail-closed** — established v0.1; dirty-state simulation test v0.2.
- [ ] **Manual cross-platform smoke** — even at v0.1, do at least one Windows build to catch packaging issues early.
- [ ] **Test golden vectors for AAD and KDF** — added when crypto code is first written, never deferred.

These items are the difference between "a working prototype" and "a vault that doesn't silently corrupt itself or leak secrets."

---

## Sources

- [Wails Application Development Guide](https://wails.io/docs/guides/application-development/) — `app.go` / `main.go` split, `Bind` mechanics, runtime context pattern. (HIGH confidence — official docs.)
- [Wails Discussion #1499 — Multiple bindings v2](https://github.com/wailsapp/wails/discussions/1499) — multi-struct binding pattern, `OnStartup` context distribution. (HIGH — maintainer-confirmed.)
- [Wails Discussion #909 — Complex app structure](https://github.com/wailsapp/wails/discussions/909) — confirms no prescribed structure; flexibility is intentional. (HIGH — maintainer post.)
- [Alex Edwards — The "Fat Service" Pattern for Go](https://www.alexedwards.net/blog/the-fat-service-pattern) — service-layer separation rationale. (HIGH — well-known Go author.)
- [Layered architecture in Go (Shershnev)](https://medium.com/@shershnev/layered-architecture-implementation-in-golang-6318a72c1e10) — dependency direction rules. (MEDIUM — community blog.)
- [golang-migrate v4 — sqlite3 driver](https://pkg.go.dev/github.com/golang-migrate/migrate/v4/database/sqlite3) — `ErrDirty` semantics, fail-closed behavior. (HIGH — official package.)
- [golang-migrate Issue #282 — Dirty version recovery](https://github.com/golang-migrate/migrate/issues/282) — manual recovery guidance for dirty state. (HIGH — issue tracker.)
- [Mastering Database Migrations in Go with golang-migrate and SQLite (DEV.to)](https://dev.to/ouma_ouma/mastering-database-migrations-in-go-with-golang-migrate-and-sqlite-3jhb) — `embed.FS` integration. (MEDIUM — community tutorial; verified pattern.)
- PRD §2 (Locked Architecture), §4 (Threat Model), §5 (Crypto Design), §6 (SQLite), §10 (Wails Contract), §13 (Auto-Lock), §14 (Tests), §15 (Milestones), §18 (Hard Rules), §20 (Sequence). (HIGH — project source of truth.)

---
*Architecture research for: Abyss — local-first desktop password & key vault*
*Researched: 2026-04-29*
