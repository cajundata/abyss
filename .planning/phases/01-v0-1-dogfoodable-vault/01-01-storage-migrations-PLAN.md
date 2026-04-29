---
phase: 01-v0-1-dogfoodable-vault
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - go.mod
  - go.sum
  - main.go
  - app.go
  - wails.json
  - .gitignore
  - .golangci.yml
  - migrations/0001_init.up.sql
  - migrations/0001_init.down.sql
  - internal/storage/db.go
  - internal/storage/db_test.go
  - internal/storage/migrations/migrate.go
  - internal/storage/migrations/migrate_test.go
  - internal/storage/repo_vault.go
  - internal/storage/repo_settings.go
  - frontend/package.json
  - frontend/src/main.tsx
  - frontend/src/App.tsx
  - .github/workflows/ci.yml
autonomous: true
requirements_addressed:
  - VAULT-04
  - SEC-09
  - STORE-01
  - STORE-02
  - STORE-03
  - STORE-04
  - STORE-05
  - STORE-06
  - STORE-07
  - STORE-08

must_haves:
  truths:
    - "Wails v2.12.0 (NOT v3 alpha) scaffolds and runs `wails dev` on macOS without errors"
    - "Pure-Go SQLite driver `modernc.org/sqlite` v1.36+ is the ONLY SQLite implementation in `go.sum` — `mattn/go-sqlite3` is absent"
    - "`PRAGMA journal_mode` returns `delete` and `PRAGMA foreign_keys` returns `1` on every newly opened DB connection"
    - "Vault file's SQLite `application_id` PRAGMA equals `1314931790` (decimal of `0x4E56544E`) on Create; Open rejects any file whose `application_id` differs with `UNSUPPORTED_VAULT`"
    - "Migration runner fails closed on `migrate.ErrDirty` — no auto-force, returns `MIGRATION_FAILED`"
    - "All 5 PRD §6.3 tables exist after `0001_init.up.sql`: vault_metadata, deks, key_wrappings, records, settings"
    - "`golangci-lint` baseline runs in CI and gates merges"
    - "`errcheck` rejects unchecked errors in non-test code"
  artifacts:
    - path: "go.mod"
      provides: "Go module declaration with Wails v2.12.0 + modernc.org/sqlite v1.36+ + golang-migrate v4"
      contains: "module"
    - path: "main.go"
      provides: "Wails entrypoint binding the App struct and embedding frontend/dist"
    - path: "app.go"
      provides: "App struct with empty bound methods (filled in plan 03); ctx injection via OnStartup"
    - path: "wails.json"
      provides: "Wails build config with project name, frontend dir, build hooks"
    - path: "migrations/0001_init.up.sql"
      provides: "PRD §6.3 schema: vault_metadata + deks + key_wrappings + records + settings; sets application_id"
    - path: "migrations/0001_init.down.sql"
      provides: "Reverse schema for round-trip migration test"
    - path: "internal/storage/db.go"
      provides: "Open(path) (*sql.DB, error); validates application_id; enforces PRAGMAs"
      exports: ["Open", "ApplicationID", "ErrUnsupportedVault", "ErrCorruptVault"]
    - path: "internal/storage/migrations/migrate.go"
      provides: "RunMigrations(db) error; uses iofs source + embedded migrations FS; fail-closed on dirty"
      exports: ["RunMigrations", "ErrMigrationDirty"]
    - path: ".golangci.yml"
      provides: "errcheck + govet + ineffassign + staticcheck baseline"
    - path: ".github/workflows/ci.yml"
      provides: "Go test + golangci-lint + 'mattn must not appear' grep gate"
  key_links:
    - from: "internal/storage/db.go"
      to: "internal/storage/migrations/migrate.go"
      via: "Open() returns *sql.DB; caller (vault.Service) calls migrate.RunMigrations(db)"
      pattern: "RunMigrations"
    - from: "internal/storage/migrations/migrate.go"
      to: "migrations/*.sql"
      via: "embed.FS via //go:embed migrations/*.sql"
      pattern: "//go:embed migrations"
    - from: ".github/workflows/ci.yml"
      to: "go.sum"
      via: "grep gate that fails CI if mattn/go-sqlite3 appears"
      pattern: "mattn/go-sqlite3"
---

<objective>
Scaffold the Wails v2.12.0 project, wire `modernc.org/sqlite` (pure Go, NO CGO) into `database/sql`, embed the first migration via `golang-migrate/v4` + `iofs`, create the five PRD §6.3 tables, and enforce `application_id = 0x4E56544E` + `journal_mode=DELETE` + `foreign_keys=ON` at every DB open. Land the baseline CI gates (`errcheck`, `golangci-lint`, plus a "mattn must not appear in go.sum" grep gate). Resolve the two open research questions inline (D-07: exact migrate driver import path; D-07: DSN PRAGMA syntax).

Purpose: Phase 1 cannot encrypt anything until storage works AND the cross-OS dual-machine workflow (D-04/D-06) is unblocked. Pure-Go SQLite is locked because daily macOS-iteration plus end-of-plan Windows verification (D-05) both need a CGO-free toolchain. This plan also lays the lint/CI floor that the rest of Phase 1 builds on.

Output: A `wails dev` build that opens, creates a SQLite vault file with the PRD-specified schema, and rejects non-Abyss files at Open. CI gates green on the first push.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@docs/PRD.md
@CLAUDE.md
@.planning/STATE.md
@.planning/ROADMAP.md
@.planning/REQUIREMENTS.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md
@.planning/research/STACK.md
@.planning/research/ARCHITECTURE.md

<interfaces>
<!-- Key contracts produced by this plan that downstream plans 02-05 consume. -->

From `internal/storage/db.go`:
```go
package storage

import "database/sql"

// ApplicationID is PRD §6.1: 0x4E56544E (decimal 1314931790).
const ApplicationID int64 = 0x4E56544E // 1314931790

// Open opens (or creates) the vault SQLite file. On an existing file it verifies
// application_id and returns ErrUnsupportedVault if mismatched. PRAGMA journal_mode
// and foreign_keys are enforced via DSN.
//
// path: absolute filesystem path. If the file does not exist, it is created and
// application_id is set as part of the migration up step (caller must run
// migrations.RunMigrations(db) before any app-level use).
func Open(path string) (*sql.DB, error)

// Errors returned by Open.
var (
    ErrUnsupportedVault = errors.New("storage: file is not an Abyss vault (application_id mismatch)")
    ErrCorruptVault     = errors.New("storage: vault file appears corrupt")
)
```

From `internal/storage/migrations/migrate.go`:
```go
package migrations

import "database/sql"

// RunMigrations applies all pending up-migrations using the embedded FS.
// Fails closed on dirty schema_migrations state — never auto-forces.
func RunMigrations(db *sql.DB) error

var ErrMigrationDirty = errors.New("migrations: schema_migrations table is dirty; manual repair required")
```

From `migrations/0001_init.up.sql` (signatures):
- 5 tables: `vault_metadata`, `deks`, `key_wrappings`, `records`, `settings`
- `PRAGMA application_id = 1314931790;` set in the up-migration
- All FKs declared (`key_wrappings.dek_id → deks.id`, `records.dek_id → deks.id`)
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Scaffold Wails v2.12.0 project + lock toolchain versions</name>
  <files>go.mod, go.sum, main.go, app.go, wails.json, .gitignore, frontend/package.json, frontend/src/main.tsx, frontend/src/App.tsx</files>
  <read_first>
    - CLAUDE.md (full file — especially "Already Locked by PRD" table and §1, §1a Mantine v8.3.18 lock, §1d clipboard runtime)
    - docs/PRD.md §2 (Locked Architecture Decisions — Wails v2 NOT v3, Vite/React/TS/Mantine NOT Next.js)
    - docs/PRD.md §18 (Hard Rules #10, #11)
    - .planning/research/STACK.md §1 + §2 (recommended stack table; installation commands)
    - .planning/research/ARCHITECTURE.md §2.1 (top-level tree)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-08 (Mantine v8.3.18 lock)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md (Theme bootstrap section — `<MantineProvider defaultColorScheme="auto">`, import order)
  </read_first>
  <action>
    1. Install Wails CLI v2.12.0 if not present: `go install github.com/wailsapp/wails/v2/cmd/wails@v2.12.0`. Verify with `wails version` (must print v2.12.0).

    2. Run `wails init -n abyss -t react-ts` in a sibling temp directory (per .planning/research/STACK.md §2 instructions). The default scaffold creates `main.go`, `app.go`, `wails.json`, `frontend/`. Move the resulting files INTO the existing repo root (`C:\Users\weldo\Projects\abyss\`). Preserve existing `docs/`, `.planning/`, `CLAUDE.md`, `README.md`, `.gitignore`.

    3. Edit `go.mod` to declare Go 1.25 toolchain (per CLAUDE.md §1: Go 1.25.9 conservative; 1.26.2 also acceptable). Pin Wails to `github.com/wailsapp/wails/v2 v2.12.0` exactly. Run `go mod tidy`.

    4. Add backend dependencies (do NOT add `mattn/go-sqlite3` — D-06 forbids it):
       ```bash
       go get modernc.org/sqlite@v1.36.0
       go get github.com/golang-migrate/migrate/v4@latest
       go get github.com/google/uuid@v1.6.0
       go get github.com/stretchr/testify@v1.10.0
       ```
       Note: `modernc.org/sqlite` driver registers under name `"sqlite"` (NOT `"sqlite3"`) per .planning/research/STACK.md line 430.

    5. Edit `frontend/package.json` to pin Mantine v8.3.18 (D-08; v9 forbidden until v1.0 milestone boundary):
       ```json
       "dependencies": {
         "@mantine/core": "^8.3.18",
         "@mantine/hooks": "^8.3.18",
         "@mantine/form": "^8.3.18",
         "@mantine/notifications": "^8.3.18",
         "@mantine/modals": "^8.3.18",
         "@tabler/icons-react": "^3.0.0",
         "react": "^19.2.0",
         "react-dom": "^19.2.0",
         "react-router-dom": "^6.30.0",
         "zustand": "^5.0.0",
         "zod": "^3.23.0"
       },
       "devDependencies": {
         "typescript": "^5.9.2",
         "vite": "^8.0.10"
       }
       ```
       Run `npm install` in `frontend/`.

    6. Edit `frontend/src/main.tsx` per UI-SPEC Theme bootstrap section (import order is REQUIRED for Mantine v8 layered styles):
       ```tsx
       import React from 'react';
       import ReactDOM from 'react-dom/client';
       import { MantineProvider, createTheme } from '@mantine/core';
       import { Notifications } from '@mantine/notifications';
       import '@mantine/core/styles.css';
       import '@mantine/notifications/styles.css';
       import './style.css';
       import App from './App';

       const appTheme = createTheme({
         spacing: { xs: '4px', sm: '8px', md: '16px', lg: '24px', xl: '32px' },
         other: { spacing2xl: '48px', spacing3xl: '64px' },
         fontSizes: { sm: '14px', md: '16px', lg: '22px', xl: '28px' },
         headings: {
           sizes: {
             h1: { fontSize: '28px', lineHeight: '1.2', fontWeight: '600' },
             h2: { fontSize: '22px', lineHeight: '1.3', fontWeight: '600' },
           },
         },
         focusRing: 'auto',
       });

       ReactDOM.createRoot(document.getElementById('root')!).render(
         <React.StrictMode>
           <MantineProvider theme={appTheme} defaultColorScheme="auto">
             <Notifications position="top-right" />
             <App />
           </MantineProvider>
         </React.StrictMode>
       );
       ```

    7. `frontend/src/App.tsx` — placeholder body that just renders `<div>Abyss</div>`. Plan 03 replaces this with the router + screens.

    8. Edit `wails.json` — set `name: "abyss"`, default window size `width: 960, height: 640, minWidth: 720, minHeight: 480` per UI-SPEC §Cross-Platform Rendering Notes.

    9. Run `wails doctor` — must report no missing required deps. Run `wails build` to confirm a clean compile (debug build is fine; we just need it to compile).

    10. Commit: `chore(01-01): scaffold Wails v2.12.0 + Mantine v8.3.18 + frontend toolchain`.
  </action>
  <verify>
    <automated>wails version 2>&amp;1 | grep -F "v2.12.0" &amp;&amp; cd /c/Users/weldo/Projects/abyss &amp;&amp; go build ./... &amp;&amp; cd frontend &amp;&amp; npm run build</automated>
  </verify>
  <acceptance_criteria>
    - `wails version` outputs a string containing `v2.12.0`
    - `go.mod` contains the literal string `github.com/wailsapp/wails/v2 v2.12.0`
    - `go.mod` does NOT contain the string `github.com/mattn/go-sqlite3` (no CGO SQLite anywhere)
    - `go.sum` does NOT contain `github.com/mattn/go-sqlite3` (run `grep -c 'mattn/go-sqlite3' go.sum` returns `0`)
    - `frontend/package.json` `dependencies` pins `@mantine/core` to `^8.3.18` (run `grep -c '"@mantine/core": "\^8\.3\.' frontend/package.json` returns `1`); MUST NOT contain `^9.` for any `@mantine/*` package
    - `frontend/src/main.tsx` contains the literal string `<MantineProvider theme={appTheme} defaultColorScheme="auto">`
    - `wails build` exits 0 and produces a build artifact under `build/bin/`
    - `go build ./...` exits 0
  </acceptance_criteria>
  <done>Wails v2 scaffold compiles and packages on macOS; frontend builds; no CGO SQLite anywhere in the dep graph; Mantine v8.3.18 pinned.</done>
</task>

<task type="auto">
  <name>Task 2: Wire modernc.org/sqlite into database/sql with PRAGMA-enforcing DSN + write the first migration + storage.Open</name>
  <files>internal/storage/db.go, internal/storage/db_test.go, internal/storage/migrations/migrate.go, internal/storage/migrations/migrate_test.go, internal/storage/repo_vault.go, internal/storage/repo_settings.go, migrations/0001_init.up.sql, migrations/0001_init.down.sql</files>
  <read_first>
    - docs/PRD.md §6 (Storage Design — §6.1 application_id, §6.2 journal mode, §6.3 ALL FIVE table DDL — vault_metadata / deks / key_wrappings / records / settings)
    - docs/PRD.md §2.5 (Migration System — fail-closed on dirty)
    - docs/PRD.md §18 Hard Rule #12 (NO WAL mode), #15 (do not couple migrations to record decryption)
    - .planning/research/STACK.md §3 (golang-migrate wiring — the example imports `database/sqlite` not `database/sqlite3`; DSN format `sqlite://file:/abs/path?_pragma=journal_mode(DELETE)&_pragma=foreign_keys(on)`)
    - .planning/research/ARCHITECTURE.md §7.1 (OpenVault sequence; PRAGMA order)
    - .planning/research/ARCHITECTURE.md §7.2 (dirty-state handling — no auto-force)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-06, D-07 (open research: confirm `database/sqlite` import path; verify DSN params)
    - CLAUDE.md "Already Locked by PRD" table (rollback journal, application_id, no SQLCipher)
  </read_first>
  <action>
    Resolve open research questions D-07 inline:

    **Driver import path resolution (D-07):** Per .planning/research/STACK.md §3 line 278 the pure-Go variant is `github.com/golang-migrate/migrate/v4/database/sqlite` (without "3"). Use it. After implementation, the next task verifies via `go mod why github.com/mattn/go-sqlite3` that it does NOT appear.

    **DSN format (D-07):** Per .planning/research/STACK.md line 301: `sqlite://file:/abs/path/to/vault.db?_pragma=journal_mode(DELETE)&_pragma=foreign_keys(on)` for the migrate driver. For raw `sql.Open` use the modernc dataform: `file:/abs/path?_pragma=journal_mode(DELETE)&_pragma=foreign_keys(on)` with driver name `"sqlite"` (NOT `"sqlite3"`). Verify by querying `PRAGMA journal_mode;` and `PRAGMA foreign_keys;` after open in db_test.go.

    1. **`migrations/0001_init.up.sql`** — copy PRD §6.3 DDL exactly. Add `PRAGMA application_id = 1314931790;` (decimal of `0x4E56544E`) as the first statement. Five tables in order: `vault_metadata`, `deks`, `key_wrappings`, `records`, `settings`. Include all CHECK constraints (`vault_metadata.id = 'vault'`, `deks.status IN ('active','retired')`, `key_wrappings.wrapping_type IN ('master_password')`) and all FOREIGN KEY clauses (`key_wrappings.dek_id → deks.id`, `records.dek_id → deks.id`).

       Schema invariants:
       - `vault_metadata`: `id TEXT PRIMARY KEY CHECK (id = 'vault')`, `vault_id TEXT NOT NULL UNIQUE`, `schema_version INTEGER NOT NULL`, `record_envelope_version INTEGER NOT NULL`, `created_at TEXT NOT NULL`, `updated_at TEXT NOT NULL`.
       - `deks`: `id TEXT PRIMARY KEY`, `vault_id TEXT NOT NULL`, `status TEXT NOT NULL CHECK (status IN ('active','retired'))`, `created_at TEXT NOT NULL`, `retired_at TEXT`.
       - `key_wrappings`: `id TEXT PRIMARY KEY`, `vault_id TEXT NOT NULL`, `dek_id TEXT NOT NULL`, `wrapping_type TEXT NOT NULL CHECK (wrapping_type IN ('master_password'))`, `kdf_algorithm TEXT NOT NULL`, `kdf_params_json TEXT NOT NULL`, `salt BLOB NOT NULL`, `wrapped_dek_nonce BLOB NOT NULL`, `wrapped_dek BLOB NOT NULL`, `created_at TEXT NOT NULL`, `updated_at TEXT NOT NULL`, `FOREIGN KEY (dek_id) REFERENCES deks(id)`.
       - `records`: `id TEXT PRIMARY KEY`, `vault_id TEXT NOT NULL`, `dek_id TEXT NOT NULL`, `record_type TEXT NOT NULL`, `public_key_openssh TEXT`, `public_key_pem TEXT`, `public_key_fingerprint TEXT`, `encrypted_metadata_blob BLOB NOT NULL`, `metadata_nonce BLOB NOT NULL`, `encrypted_secret_blob BLOB NOT NULL`, `secret_nonce BLOB NOT NULL`, `created_at TEXT NOT NULL`, `updated_at TEXT NOT NULL`, `FOREIGN KEY (dek_id) REFERENCES deks(id)`.
       - `settings`: `key TEXT PRIMARY KEY`, `value TEXT NOT NULL`, `updated_at TEXT NOT NULL`.

    2. **`migrations/0001_init.down.sql`** — `DROP TABLE` statements in reverse FK order: `settings`, `records`, `key_wrappings`, `deks`, `vault_metadata`. Round-trip is required for migrate_test.

    3. **`internal/storage/migrations/migrate.go`** — implements:
       ```go
       package migrations

       import (
           "database/sql"
           "embed"
           "errors"
           "fmt"

           "github.com/golang-migrate/migrate/v4"
           sqlitemigrate "github.com/golang-migrate/migrate/v4/database/sqlite"
           "github.com/golang-migrate/migrate/v4/source/iofs"
       )

       //go:embed all:migrations
       var migrationsFS embed.FS

       var ErrMigrationDirty = errors.New("migrations: schema_migrations table is dirty; manual repair required")

       func RunMigrations(db *sql.DB) error {
           src, err := iofs.New(migrationsFS, "migrations")
           if err != nil { return fmt.Errorf("migrations: load fs: %w", err) }
           drv, err := sqlitemigrate.WithInstance(db, &sqlitemigrate.Config{})
           if err != nil { return fmt.Errorf("migrations: driver: %w", err) }
           m, err := migrate.NewWithInstance("iofs", src, "sqlite", drv)
           if err != nil { return fmt.Errorf("migrations: new: %w", err) }
           if err := m.Up(); err != nil &amp;&amp; !errors.Is(err, migrate.ErrNoChange) {
               // Detect dirty state (PRD §2.5 fail-closed)
               if errors.Is(err, migrate.ErrDirty) {
                   return ErrMigrationDirty
               }
               // Some migrate versions wrap dirty errors in ErrDirty struct value, not Is-comparable.
               // Defensive check on string form:
               if strings.Contains(err.Error(), "dirty") {
                   return ErrMigrationDirty
               }
               return fmt.Errorf("migrations: up: %w", err)
           }
           return nil
       }
       ```
       Embed comment is `//go:embed all:migrations` (the SQL files live in repo-root `migrations/`, but they need to be relative to this file — the actual layout is described below in the build wiring). Two options:
       (a) Move SQL files to `internal/storage/migrations/migrations/0001_init.{up,down}.sql` (sibling subdir).
       (b) Keep SQL at repo-root `migrations/` and use `//go:embed` from a Go file in the same package at repo root. Per .planning/research/ARCHITECTURE.md §2.2, SQL stays at `/migrations/` repo root and the runner is in `/internal/storage/migrations/migrate.go`. Use option (a) — copy the SQL files into a subdirectory `internal/storage/migrations/migrations/` so `//go:embed migrations/*.sql` works without filesystem-tricks. ARCHITECTURE.md's stated layout (SQL at `/migrations/`) is the mental model; the physical embed source is in the package.

       Practical resolution: keep SQL files BOTH at repo-root `migrations/` (canonical, human-discoverable, plan-source-of-truth) AND have the migrations package use `//go:embed all:./migrations` referencing the SAME files via the build's relative path. Since `//go:embed` cannot reference parent directories, copy the SQL files into `internal/storage/migrations/migrations/` and have CI verify they match repo-root `migrations/` byte-for-byte (Task 3 below adds a hash check).

       Simpler resolution adopted: place SQL files canonically at `internal/storage/migrations/migrations/0001_init.{up,down}.sql` and DO NOT keep a duplicate at repo root. Update `.gitignore` accordingly. The repo-root `migrations/` directory referenced in ARCHITECTURE.md becomes documentation, not code — note this in the SUMMARY.

    4. **`internal/storage/db.go`** — implements:
       ```go
       package storage

       import (
           "database/sql"
           "errors"
           "fmt"

           _ "modernc.org/sqlite" // registers driver name "sqlite"
       )

       const ApplicationID int64 = 0x4E56544E // 1314931790 decimal — PRD §6.1

       var (
           ErrUnsupportedVault = errors.New("storage: file is not an Abyss vault (application_id mismatch)")
           ErrCorruptVault     = errors.New("storage: vault file appears corrupt")
       )

       // Open opens a SQLite vault file. Caller must run migrations.RunMigrations(db) before
       // application use. PRAGMA journal_mode=DELETE and foreign_keys=ON are enforced via DSN.
       //
       // For an EXISTING file, application_id is checked AFTER open and ErrUnsupportedVault
       // is returned on mismatch. For a NEW file, application_id is set by migration 0001 up.
       func Open(path string) (*sql.DB, error) {
           dsn := fmt.Sprintf("file:%s?_pragma=journal_mode(DELETE)&_pragma=foreign_keys(on)", path)
           db, err := sql.Open("sqlite", dsn)
           if err != nil { return nil, fmt.Errorf("storage: open: %w", err) }
           if err := db.Ping(); err != nil {
               db.Close()
               return nil, fmt.Errorf("storage: ping: %w", err)
           }
           return db, nil
       }

       // VerifyApplicationID returns ErrUnsupportedVault if the application_id doesn't match.
       // Caller invokes this AFTER Open AND after RunMigrations on existing-file path.
       // For new files (post-migration), this is also called and must succeed because
       // 0001_init.up.sql sets application_id.
       func VerifyApplicationID(db *sql.DB) error {
           var id int64
           if err := db.QueryRow("PRAGMA application_id").Scan(&id); err != nil {
               return fmt.Errorf("storage: read application_id: %w", err)
           }
           if id != ApplicationID {
               return ErrUnsupportedVault
           }
           return nil
       }
       ```

    5. **`internal/storage/repo_vault.go`** — minimal stubs returning `error("not implemented")` for the calls that plan 03/04 will fill: `InsertVaultMetadata`, `GetVaultMetadata`, `InsertDEK`, `GetActiveDEK`, `InsertKeyWrapping`, `GetKeyWrappingByType`. These are STUBS — keep them tiny so plan 03/04 can fill the bodies. The interfaces lock here so downstream plans don't redesign them.

    6. **`internal/storage/repo_settings.go`** — minimal `Get(key) (string, error)` and `Set(key, value) error` against `settings` table. Used only by plan 03 for non-sensitive prefs.

    7. **`internal/storage/db_test.go`** — tests:
       - `TestOpen_NewFile_Pragmas`: open temp path, call `RunMigrations`, then `db.QueryRow("PRAGMA journal_mode").Scan(&mode)` → expect `"delete"` (lowercase). `db.QueryRow("PRAGMA foreign_keys").Scan(&fk)` → expect `1`.
       - `TestOpen_NewFile_ApplicationID`: same setup → `VerifyApplicationID(db)` returns nil; raw query `PRAGMA application_id` returns `1314931790`.
       - `TestOpen_NewFile_FiveTables`: query `SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%' AND name NOT LIKE 'schema_migrations' ORDER BY name`. Expect rows: `deks, key_wrappings, records, settings, vault_metadata`.
       - `TestVerifyApplicationID_WrongID_ReturnsUnsupportedVault`: open a fresh SQLite file via raw `sql.Open` (don't run migrations), set `PRAGMA application_id = 12345`, then call `VerifyApplicationID` → expect `ErrUnsupportedVault`.
       - `TestVerifyApplicationID_BlankFile_ReturnsUnsupportedVault`: open a fresh SQLite file with NO PRAGMA set (default app ID is 0) → expect `ErrUnsupportedVault`.
       - All tests use `t.TempDir()` for vault paths (per .planning/research/ARCHITECTURE.md §6 test isolation).

    8. **`internal/storage/migrations/migrate_test.go`** — tests:
       - `TestRunMigrations_FreshDB_AllTablesExist`: open tmp DB, run migrations, query sqlite_master, assert 5 tables + `schema_migrations`.
       - `TestRunMigrations_RoundTrip_UpDownUp`: apply up, then call `m.Down()` (or use `migrate.Migrate` directly), then up again. Assert no errors and final state has all 5 tables. (Idempotence check.)
       - `TestRunMigrations_DirtyState_ReturnsErrMigrationDirty`: open tmp DB, run migrations, then manually `UPDATE schema_migrations SET dirty = 1`, then call `RunMigrations` → expect `errors.Is(err, ErrMigrationDirty)`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss &amp;&amp; go test ./internal/storage/... -v -count=1</automated>
  </verify>
  <acceptance_criteria>
    - `migrations/...0001_init.up.sql` (located at `internal/storage/migrations/migrations/0001_init.up.sql`) contains the literal `PRAGMA application_id = 1314931790;`
    - The up-migration contains `CREATE TABLE vault_metadata`, `CREATE TABLE deks`, `CREATE TABLE key_wrappings`, `CREATE TABLE records`, `CREATE TABLE settings` (5 distinct creates)
    - The up-migration contains `FOREIGN KEY (dek_id) REFERENCES deks(id)` at least twice (key_wrappings AND records)
    - The up-migration contains `CHECK (id = 'vault')` (vault_metadata constraint)
    - `internal/storage/db.go` declares `const ApplicationID int64 = 0x4E56544E`
    - `internal/storage/db.go` imports `_ "modernc.org/sqlite"` and uses `sql.Open("sqlite", dsn)` (NOT `"sqlite3"`)
    - `internal/storage/db.go` builds DSN containing the literal substrings `_pragma=journal_mode(DELETE)` AND `_pragma=foreign_keys(on)`
    - `internal/storage/migrations/migrate.go` imports `github.com/golang-migrate/migrate/v4/database/sqlite` (without "3")
    - `go test ./internal/storage/... -count=1 -run TestOpen_NewFile_Pragmas` exits 0
    - `go test ./internal/storage/... -count=1 -run TestOpen_NewFile_ApplicationID` exits 0
    - `go test ./internal/storage/... -count=1 -run TestOpen_NewFile_FiveTables` exits 0
    - `go test ./internal/storage/migrations -count=1 -run TestRunMigrations_DirtyState` exits 0
    - `go mod why github.com/mattn/go-sqlite3` exits NON-ZERO (i.e., the dep is absent — D-06 lock)
  </acceptance_criteria>
  <done>SQLite opens with correct PRAGMAs; migrations apply cleanly; dirty-state fails closed; application_id check rejects non-Abyss files; pure-Go SQLite confirmed.</done>
</task>

<task type="auto">
  <name>Task 3: Land golangci-lint + errcheck + CI baseline + "no mattn" grep gate</name>
  <files>.golangci.yml, .github/workflows/ci.yml, .gitignore</files>
  <read_first>
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-02 (CI gates land WITH the code they protect; errcheck/golangci-lint baseline lands in plan 1)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-04, D-05, D-06 (cross-platform discipline; pure-Go SQLite mandatory)
    - .planning/research/ARCHITECTURE.md §11 (anti-patterns — informs lint rule selection)
    - go.mod, go.sum (so the grep gate references real paths)
    - CLAUDE.md §1 development tools (golangci-lint v1.62+)
  </read_first>
  <action>
    1. **`.golangci.yml`** — baseline config. NO `forbidigo` rule for `math/rand` here (D-02: that lands in plan 02 with the first crypto code). Enable:
       ```yaml
       run:
         go: "1.25"
         timeout: 5m
       linters:
         enable:
           - errcheck     # D-02: required baseline
           - govet
           - ineffassign
           - staticcheck
           - unused
           - gosimple
           - typecheck
       linters-settings:
         errcheck:
           check-type-assertions: true
           check-blank: true
       issues:
         exclude-dirs:
           - frontend
           - build
       ```

    2. **`.github/workflows/ci.yml`** — GitHub Actions matrix on `ubuntu-latest`, `macos-latest`, `windows-latest`. Steps:
       - Set up Go 1.25 (`actions/setup-go@v5` with `go-version: '1.25.x'`).
       - Set up Node 22 LTS (`actions/setup-node@v4` with `node-version: '22'`).
       - `go mod download`.
       - **Grep gate (D-06):** `if grep -q 'github.com/mattn/go-sqlite3' go.sum; then echo "FAIL: mattn/go-sqlite3 detected; D-06 forbids CGO SQLite"; exit 1; fi`. NOTE: the gate uses `grep -q` (silent), separate from the no-plaintext gate that lands in plan 04.
       - `go test ./... -race -count=1` (race detection on; `-count=1` defeats test cache).
       - golangci-lint via `golangci/golangci-lint-action@v6` with `version: v1.62.0`.
       - `cd frontend &amp;&amp; npm ci &amp;&amp; npm run build` (verifies frontend compiles).
       - DO NOT run `wails build` in CI for this plan — that's Windows-smoke material in plan 05. CI here just verifies Go + frontend compile in isolation.

    3. **`.gitignore`** — add Wails-typical entries if not present from scaffold: `build/bin/`, `frontend/dist/`, `frontend/wailsjs/` (auto-generated; downstream plans toggle this depending on team policy — for v0.1 keep wailsjs ignored to avoid generated-file diff noise; the no-drift CI gate in plan 03 regenerates it on demand). Also: `*.abyssvault`, `*.abyssvault-journal`.

    4. Run locally: `golangci-lint run ./...` (must exit 0). Run `go test ./... -race`. Push and verify CI green on Linux + macOS + Windows (Windows is the cheapest pre-flight for D-05).

    5. Commit: `chore(01-01): land CI baseline (errcheck, golangci-lint, no-mattn grep gate)`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss &amp;&amp; golangci-lint run ./... &amp;&amp; ! grep -q 'github.com/mattn/go-sqlite3' go.sum &amp;&amp; go test ./... -race -count=1</automated>
  </verify>
  <acceptance_criteria>
    - `.golangci.yml` exists and contains a `linters.enable` list including `errcheck`
    - `.golangci.yml` does NOT contain `forbidigo` (that's plan 02's deliverable)
    - `.github/workflows/ci.yml` contains the literal grep gate `grep -q 'github.com/mattn/go-sqlite3' go.sum`
    - `.github/workflows/ci.yml` matrix runs on ubuntu-latest AND macos-latest AND windows-latest (`grep -c 'os:' .github/workflows/ci.yml` reflects matrix usage)
    - `golangci-lint run ./...` exits 0
    - `go test ./... -race -count=1` exits 0
    - First CI workflow run on the plan-1 branch is green on all 3 OSes (verified by `gh run list --workflow=ci.yml --limit 1` showing `success`)
  </acceptance_criteria>
  <done>CI gates baseline locked. Future plans extend (forbidigo in 02, generate-no-drift in 03, no-plaintext-grep in 04). The "no mattn" gate is the D-06 enforcement floor.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Filesystem ↔ storage.Open | User-controlled file path; could point to any file (intentional renaming, malware-substituted file). application_id check is the file-identity gate. |
| Build pipeline ↔ go.sum | Transitive deps could pull mattn/go-sqlite3 silently; CI gate catches it. |
| Migration runner ↔ schema_migrations | Crash mid-migration leaves dirty=1; auto-force would silently corrupt vaults. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-01 | Spoofing | storage.Open on existing file | mitigate | `VerifyApplicationID` returns `ErrUnsupportedVault` if `PRAGMA application_id != 1314931790`; PRD §6.1 file-identity gate is enforced at every Open before any other operation. |
| T-01-02 | Tampering | SQLite vault file (rollback journal artifacts) | accept | Rollback journal mode is locked by PRD §2.4 + §18 hard rule #12 for vault portability. Journal file (`*.abyssvault-journal`) is transient and contains the same encrypted blobs — no plaintext leak. |
| T-01-03 | Tampering | schema_migrations dirty after crash | mitigate | `RunMigrations` returns `ErrMigrationDirty` on `dirty=1` per PRD §2.5; no auto-force; user is told to restore from backup. CI test `TestRunMigrations_DirtyState_ReturnsErrMigrationDirty` enforces. |
| T-01-04 | Information Disclosure | go.sum supply chain — mattn/go-sqlite3 sneaks in via transitive dep | mitigate | CI grep gate fails the build on `mattn/go-sqlite3` in `go.sum` (D-06 lock; .planning/research/STACK.md §3 line 299 directive). |
| T-01-05 | Denial of Service | Foreign-key violation on bad insert | mitigate | `PRAGMA foreign_keys = ON` enforced via DSN at every connection open; verified by `TestOpen_NewFile_Pragmas`. Without it, FK declarations are decorative in SQLite. |
| T-01-06 | Tampering | WAL sidecar file leakage | mitigate | `PRAGMA journal_mode = DELETE` enforced via DSN; PRD §2.4 + §6.2 + §18 hard rule #12 forbid WAL. Verified by `TestOpen_NewFile_Pragmas`. |
| T-01-07 | Spoofing | Blank/0-application_id SQLite file (random non-Abyss DB) | mitigate | Default `application_id` is 0; `VerifyApplicationID` returns `ErrUnsupportedVault` for any value != `0x4E56544E`. Test `TestVerifyApplicationID_BlankFile_ReturnsUnsupportedVault` enforces. |
</threat_model>

<verification>
**End-of-plan checks (run from repo root):**

1. `wails version` shows `v2.12.0`
2. `go build ./...` exits 0
3. `go test ./internal/storage/... -race -count=1` exits 0 with all 5+ tests passing
4. `golangci-lint run ./...` exits 0
5. `grep -c 'github.com/mattn/go-sqlite3' go.sum` returns `0`
6. `grep -c '@mantine/core": "\^8\.3' frontend/package.json` returns `1` (exactly v8.3.x pinned)
7. `grep -c '@mantine/.*": "\^9\.' frontend/package.json` returns `0` (no v9 anywhere)
8. `wails build` exits 0
9. `cd frontend &amp;&amp; npm run build` exits 0
10. CI workflow latest run on this branch reports `success` on ubuntu-latest, macos-latest, AND windows-latest (`gh run list --workflow=ci.yml --limit 1 --json conclusion,status` returns success)
11. **D-05 cross-OS smoke (manual, end-of-plan):** on Windows machine, `git pull`, `wails dev`, app launches and shows the placeholder `<div>Abyss</div>`. Sign off in `.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md` with date + commit SHA.
</verification>

<success_criteria>
- Wails v2.12.0 + Mantine v8.3.18 + React 19.2 + TypeScript 5.9 + Vite 8 frontend toolchain compiles and runs `wails dev` on macOS
- `modernc.org/sqlite` v1.36+ is the ONLY SQLite driver (verified by go.sum grep gate); CGO not required
- `application_id = 0x4E56544E` set on Create, validated on Open with `ErrUnsupportedVault` on mismatch
- `PRAGMA journal_mode=DELETE` and `foreign_keys=ON` enforced via DSN at every Open
- All 5 PRD §6.3 tables (vault_metadata, deks, key_wrappings, records, settings) created by migration 0001 with all CHECK constraints + FK declarations
- Migration runner fails closed on `dirty=1` with `ErrMigrationDirty`; no auto-force
- CI gates pass on ubuntu/macos/windows: `go test -race`, `golangci-lint`, `frontend npm run build`, `no mattn in go.sum`
- D-05 Windows smoke signed off in `01-VERIFICATION.md`
</success_criteria>

<output>
After completion, create `.planning/phases/01-v0-1-dogfoodable-vault/01-01-SUMMARY.md` documenting:
- Resolved D-07: confirmed `github.com/golang-migrate/migrate/v4/database/sqlite` (no "3") is the pure-Go-compatible driver path. Confirmed DSN params `_pragma=journal_mode(DELETE)&_pragma=foreign_keys(on)` set both PRAGMAs at every connection.
- Resolved D-06 enforcement: `go mod why github.com/mattn/go-sqlite3` exits non-zero; CI grep gate active.
- ARCHITECTURE.md §2.2 deviation: SQL files placed at `internal/storage/migrations/migrations/` instead of repo-root `migrations/` to satisfy `//go:embed` parent-directory restriction. Documented for plan 02+ (no impact on cross-cutting concerns).
- Final commit SHAs for the plan; Windows smoke signoff date.
</output>
