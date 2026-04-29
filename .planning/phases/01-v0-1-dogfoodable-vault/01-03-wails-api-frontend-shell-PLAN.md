---
phase: 01-v0-1-dogfoodable-vault
plan: 03
type: execute
wave: 3
depends_on: ["01-02-crypto-scaffolding-PLAN.md"]
files_modified:
  - app.go
  - main.go
  - internal/vault/service.go
  - internal/vault/service_test.go
  - internal/storage/repo_vault.go
  - internal/appconfig/config.go
  - internal/appconfig/config_test.go
  - frontend/src/types/dto.ts
  - frontend/src/errors/codes.ts
  - frontend/src/errors/messages.ts
  - frontend/src/api/index.ts
  - frontend/src/api/vault.ts
  - frontend/src/api/records.ts
  - frontend/src/api/generator.ts
  - frontend/src/api/clipboard.ts
  - frontend/src/api/errors.ts
  - frontend/src/api/normalizeError.ts
  - frontend/src/state/vaultStore.ts
  - frontend/src/state/clipboardStore.ts
  - frontend/src/App.tsx
  - frontend/src/screens/Welcome/Welcome.tsx
  - frontend/src/screens/CreateVault/CreateVault.tsx
  - frontend/src/screens/UnlockVault/UnlockVault.tsx
  - frontend/src/screens/VaultHome/VaultHome.tsx
  - frontend/src/screens/RecordDetail/RecordDetail.tsx
  - frontend/src/screens/PasswordGenerator/PasswordGenerator.tsx
  - frontend/src/components/LockBadge/LockBadge.tsx
  - frontend/src/router.tsx
  - .github/workflows/ci.yml
  - scripts/check-wails-bindings.sh
autonomous: true
requirements_addressed:
  - API-01
  - API-02
  - API-03
  - API-04
  - API-06
  - API-08
  - API-09
  - UI-01
  - UI-02
  - UI-03
  - UI-13
  - VAULT-01
  - VAULT-02
  - VAULT-03
  - VAULT-05
  - VAULT-06
  - VAULT-07
  - VAULT-11
  - VAULT-12

must_haves:
  truths:
    - "Wails methods on `app.go`: CreateVault, OpenVault, UnlockVault, LockVault, GetVaultStatus, ListRecords, GetRecord, CreateRecord, UpdateRecord, DeleteRecord, SearchRecords, GeneratePassword, CopyToClipboard, ClearClipboard — exactly 14 methods (Phase 1 scope per CONTEXT D-01 plan 3)"
    - "Every secret-touching method calls `session.RequireUnlocked()` first; locked vault returns AppError{VAULT_LOCKED}"
    - "Every error returned from a bound method passes through `apperr.ToDTO()` — raw Go errors NEVER cross the Wails boundary"
    - "Frontend `frontend/src/api/` is the SINGLE entry point for Wails calls; React components MUST NOT import `frontend/wailsjs/go/main/App` directly"
    - "Frontend `normalizeError()` consumes `AppErrorDTO.code` (NOT `.message`) — there is no `error.message.includes(...)` in any screen or component"
    - "Vault create flow: file dialog → SQLite Open → migrations.RunMigrations → VerifyApplicationID → INSERT vault_metadata + DEK + key_wrapping (Argon2id KEK + WrapDEK with aad.Build) → session.Activate → return VaultSessionDTO"
    - "Vault unlock flow: SQLite Open → VerifyApplicationID → SELECT key_wrapping → DeriveKEK(masterPassword, salt, params) → UnwrapDEK with aad.Build → session.Activate → return VaultSessionDTO. Any failure returns generic AppError{UNLOCK_FAILED}"
    - "Vault lock flow: session.Lock zeros DEK + emit `vault:locked` runtime event"
    - "First-launch app config: read `last_vault_path` from `os.UserConfigDir()/Abyss/config.json` (D-11); first launch with no path routes to Welcome focused on Create; subsequent launches route to Unlock with prefilled path"
    - "Welcome / Create / Unlock screens render with Mantine v8.3.18 components per UI-SPEC; all PRD-LOCKED copy strings are verbatim"
    - "wails generate module no-drift CI check rejects PRs that change DTOs without regenerating bindings (D-02 plan-3 gate)"
  artifacts:
    - path: "app.go"
      provides: "App struct with 14 Wails-bound methods; ctx via OnStartup; service handles for vault/records/generator/clipboard"
      exports: ["App", "NewApp"]
    - path: "internal/vault/service.go"
      provides: "Service{} with Create/Open/Unlock/Lock/Status methods used by app.go"
      exports: ["Service", "NewService"]
    - path: "internal/appconfig/config.go"
      provides: "Cross-platform config-file persistence for `last_vault_path` (NOT inside vault — D-11)"
      exports: ["Load", "Save", "ConfigPath", "Config"]
    - path: "frontend/src/types/dto.ts"
      provides: "Hand-curated TS types mirroring PRD §10 DTOs (single source of truth for FE)"
    - path: "frontend/src/api/index.ts"
      provides: "Re-exports all api/*.ts wrappers; SOLE entry point"
    - path: "frontend/src/api/normalizeError.ts"
      provides: "normalizeError(unknown) → AppErrorDTO; never throws"
    - path: "frontend/src/state/vaultStore.ts"
      provides: "Zustand store for {status, vaultPath, vaultId, unlockedAt}"
    - path: "frontend/src/state/clipboardStore.ts"
      provides: "Zustand store for {active, secretKind, timeoutMs, startedAt}"
    - path: "scripts/check-wails-bindings.sh"
      provides: "CI helper: runs `wails generate module` and fails if frontend/wailsjs has uncommitted changes (D-02 plan-3 gate)"
    - path: "frontend/src/App.tsx"
      provides: "Router root + lock-state gate + vault:locked event subscription"
  key_links:
    - from: "app.go::UnlockVault"
      to: "internal/vault/service.go::Unlock"
      via: "service-layer delegation; app.go is thin façade"
      pattern: "vault\\.Service"
    - from: "internal/vault/service.go"
      to: "internal/crypto + internal/aad + internal/session"
      via: "Unlock: DeriveKEK → aad.Build(KindWrappedDEK) → UnwrapDEK → session.Activate"
      pattern: "crypto\\.UnwrapDEK"
    - from: "frontend/src/api/vault.ts"
      to: "frontend/wailsjs/go/main/App"
      via: "Sole importer of generated bindings; wraps every call with normalizeError"
      pattern: "from \"\\.\\./\\.\\./wailsjs"
    - from: "frontend/src/App.tsx"
      to: "internal/session/session.go::Lock event emission"
      via: "runtime.EventsOn('vault:locked', ...) clears stores + navigates to /unlock"
      pattern: "EventsOn.*vault:locked"
    - from: ".github/workflows/ci.yml"
      to: "scripts/check-wails-bindings.sh"
      via: "CI step runs the script and fails on drift"
      pattern: "wails generate module"
---

<objective>
Wire the Wails API surface, the frontend `api/` typed-wrapper layer, the Zustand stores, the React Router, and the 6 Phase-1 screen scaffolds. The vault create / open / unlock / lock end-to-end flows must work — the first time a record is encrypted in this codebase, it goes through `aad.Build` (proven by plan 02; plan 03 wires it into the actual service path). The frontend renders the Welcome / Create / Unlock screens per UI-SPEC and routes between them; Vault Home / Record Detail / Password Generator are scaffolded with placeholder content (plan 04 fills credential CRUD + clipboard). Land the `wails generate module` no-drift CI gate (D-02 plan-3 deliverable).

Purpose: This is the integration plan — it stitches storage + crypto + session + apperr + logger into the Wails-bound App struct, exposes the 14 Phase-1 methods to TS, and stands up the SPA shell. After this plan, the user can `wails dev`, see the Welcome screen, click Create, pick a path, type a master password, and end up at a (mostly empty) Vault Home with the green-shield "Locked"→"Unlocked" pill working. Plans 04 and 05 add credential CRUD + clipboard countdown + the Windows lifecycle smoke.

Output: A running `wails dev` build where Create + Unlock + Lock + Re-Unlock work end-to-end on macOS. The DTO contract between Go and TS is locked. The frontend api wrapper is the only call path. The vault:locked event mechanism is wired.
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
@.planning/research/STACK.md
@.planning/research/ARCHITECTURE.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-01-storage-migrations-PLAN.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-02-crypto-scaffolding-PLAN.md

<interfaces>
<!-- Critical interfaces consumed from plans 01-02 and produced for plan 04+. -->

From plan 01 (`internal/storage`):
```go
func Open(path string) (*sql.DB, error)
func VerifyApplicationID(db *sql.DB) error
var ErrUnsupportedVault, ErrCorruptVault error

func RunMigrations(db *sql.DB) error  // package internal/storage/migrations
var ErrMigrationDirty error
```

From plan 02 (`internal/aad`, `internal/crypto`, `internal/apperr`, `internal/session`, `internal/logger`):
```go
// aad
func Build(k Kind, vaultID, recordID, recordType, dekID, wrappingType string, envVersion int) ([]byte, error)
const KindRecordMetadata, KindRecordSecret, KindWrappedDEK Kind

// crypto
func GenerateDEK() ([]byte, error)
func GenerateSalt(p Argon2idParams) ([]byte, error)
func DeriveKEK(password, salt []byte, p Argon2idParams) ([]byte, error)
func WrapDEK(kek, dek, aad []byte) (wrapped, nonce []byte, err error)
func UnwrapDEK(kek, wrapped, nonce, aad []byte) ([]byte, error)
func Encrypt(key, plaintext, nonce, aad []byte) ([]byte, error)
func Decrypt(key, ciphertext, nonce, aad []byte) ([]byte, error)
func Zero(b []byte)
var DefaultArgon2idParams, FallbackArgon2idParams, MinimumArgon2idParams Argon2idParams

// apperr
func New(code Code, msg string) *AppError
func Wrap(code Code, cause error, msg string) *AppError
func ToDTO(err error) *DTO
const CodeVaultLocked, CodeUnlockFailed, CodeUnsupportedVault, CodeCorruptVault, CodeMigrationFailed, CodeValidationError, CodeRecordNotFound, CodeClipboardFailed, CodeInternalError, CodeVaultAlreadyUnlocked Code

// session
func New() *Session
func (s *Session) SetLockedFor(vaultID, vaultPath string)
func (s *Session) Activate(dek []byte, vaultID, vaultPath string) error
func (s *Session) Lock() error
func (s *Session) RequireUnlocked() error
func (s *Session) Status() Status
func (s *Session) DEK() []byte  // returns defensive copy
func (s *Session) VaultPath() string

// logger
func New() *slog.Logger
```

This plan PRODUCES (consumed by plan 04):
```go
// app.go
type App struct {
    ctx     context.Context
    log     *slog.Logger
    vault   *vault.Service
    session *session.Session
    appcfg  *appconfig.Config
}

func NewApp() *App
func (a *App) OnStartup(ctx context.Context)

// 14 bound methods (signatures match PRD §10):
func (a *App) CreateVault(req CreateVaultRequest) (*VaultSessionDTO, *apperr.DTO)
func (a *App) OpenVault(req OpenVaultRequest) (*VaultStatusDTO, *apperr.DTO)
func (a *App) UnlockVault(req UnlockVaultRequest) (*VaultSessionDTO, *apperr.DTO)
func (a *App) LockVault() *apperr.DTO
func (a *App) GetVaultStatus() (*VaultStatusDTO, *apperr.DTO)
func (a *App) ListRecords() ([]*RecordSummaryDTO, *apperr.DTO)        // returns empty slice in plan 03; plan 04 implements
func (a *App) GetRecord(id string) (*RecordDTO, *apperr.DTO)           // returns RECORD_NOT_FOUND in plan 03
func (a *App) CreateRecord(req CreateRecordRequest) (*RecordDTO, *apperr.DTO)  // stub returning VAULT_LOCKED if locked, INTERNAL_ERROR if unlocked (plan 04 implements)
func (a *App) UpdateRecord(req UpdateRecordRequest) (*RecordDTO, *apperr.DTO)
func (a *App) DeleteRecord(id string) *apperr.DTO
func (a *App) SearchRecords(req SearchRecordsRequest) ([]*RecordSummaryDTO, *apperr.DTO)
func (a *App) GeneratePassword(req GeneratePasswordRequest) (*GeneratedPasswordDTO, *apperr.DTO)  // stub in plan 03; plan 04 implements
func (a *App) CopyToClipboard(req CopyToClipboardRequest) *apperr.DTO  // stub in plan 03; plan 04 implements
func (a *App) ClearClipboard() *apperr.DTO  // stub in plan 03; plan 04 implements

// vault.Service public surface (consumed by app.go and by plan 04 records.Service):
func (s *Service) Create(ctx context.Context, path, masterPassword string) (vaultID string, err error)
func (s *Service) Open(ctx context.Context, path string) (vaultID string, err error)  // verifies application_id, runs migrations, transitions session to Locked
func (s *Service) Unlock(ctx context.Context, masterPassword string) error            // derive KEK, unwrap DEK, session.Activate; emits vault:locked is NOT here
func (s *Service) Lock(ctx context.Context) error                                      // session.Lock + emit vault:locked event
func (s *Service) Status() vault.StatusInfo                                            // wraps session + db status
func (s *Service) DB() *sql.DB                                                         // for plan 04's records.Service
```

```typescript
// frontend/src/types/dto.ts — mirror of PRD §10
export type AppErrorCode = "VAULT_LOCKED" | "VAULT_ALREADY_UNLOCKED" | "UNLOCK_FAILED"
  | "UNSUPPORTED_VAULT" | "CORRUPT_VAULT" | "MIGRATION_FAILED" | "VALIDATION_ERROR"
  | "RECORD_NOT_FOUND" | "EXPORT_FAILED" | "CLIPBOARD_FAILED" | "PLATFORM_ERROR" | "INTERNAL_ERROR";

export interface AppErrorDTO {
  code: AppErrorCode;
  message: string;
  details?: Record<string, string>;
}

export interface CreateVaultRequest { path: string; masterPassword: string; confirmMasterPassword: string; }
export interface OpenVaultRequest { path: string; }
export interface UnlockVaultRequest { path: string; masterPassword: string; }
export interface VaultStatusDTO { isOpen: boolean; isUnlocked: boolean; vaultPath?: string; vaultId?: string; lockedAt?: string; unlockedAt?: string; }
export interface VaultSessionDTO { vaultId: string; vaultPath: string; unlockedAt: string; autoLockTimeoutSeconds: number; }
export type RecordType = "credential" | "api_key" | "secure_note" | "symmetric_key" | "asymmetric_key_pair";
export interface RecordSummaryDTO { id: string; type: RecordType; title: string; subtitle?: string; tags?: string[]; publicKeyFingerprint?: string; createdAt: string; updatedAt: string; }
export interface RecordDTO { id: string; type: RecordType; metadata: Record<string, unknown>; secret: Record<string, unknown>; publicKeyOpenSSH?: string; publicKeyPEM?: string; publicKeyFingerprint?: string; createdAt: string; updatedAt: string; }
export interface CreateRecordRequest { type: RecordType; metadata: Record<string, unknown>; secret: Record<string, unknown>; publicKeyOpenSSH?: string; publicKeyPEM?: string; publicKeyFingerprint?: string; }
export interface UpdateRecordRequest { id: string; metadata: Record<string, unknown>; secret: Record<string, unknown>; publicKeyOpenSSH?: string; publicKeyPEM?: string; publicKeyFingerprint?: string; }
export interface SearchRecordsRequest { query?: string; recordType?: RecordType; tags?: string[]; }
export interface GeneratePasswordRequest { length: number; includeUppercase: boolean; includeLowercase: boolean; includeNumbers: boolean; includeSymbols: boolean; excludeAmbiguous: boolean; }
export interface GeneratedPasswordDTO { password: string; length: number; }
export type SecretKind = "password" | "api_key" | "aes_key" | "public_key" | "private_key" | "secure_note";
export interface CopyToClipboardRequest { value: string; secretKind: SecretKind; timeoutMs?: number; }
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Implement internal/vault.Service (Create / Open / Unlock / Lock) + internal/appconfig (cross-platform last-vault-path) + service tests</name>
  <files>internal/vault/service.go, internal/vault/service_test.go, internal/storage/repo_vault.go, internal/appconfig/config.go, internal/appconfig/config_test.go</files>
  <read_first>
    - docs/PRD.md §5.2 (Envelope Encryption — vault creation + unlock flows verbatim), §6.3 (vault_metadata, deks, key_wrappings DDL), §9.4 (generic unlock failure)
    - docs/PRD.md §10.2 (Vault DTO TS types — locks the JSON shape we marshal)
    - .planning/research/ARCHITECTURE.md §5.1 (lock state machine), §5.2 (where each transition is enforced), §7.1 (OpenVault sequence)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-11 (last-vault-path stored OUTSIDE the encrypted vault), D-13 (lock routes to Unlock with vault path prefilled), D-20 (UUIDv4 via google/uuid), D-23 (per-record DEK ID is active DEK at creation time)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-01-storage-migrations-PLAN.md (storage.Open + migrations.RunMigrations + repo_vault stubs)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-02-crypto-scaffolding-PLAN.md (aad.Build, crypto.{GenerateDEK,GenerateSalt,DeriveKEK,WrapDEK,UnwrapDEK,Zero}, apperr.*, session.*)
    - internal/storage/db.go (existing from plan 01)
    - internal/storage/repo_vault.go (stubs from plan 01 — fill bodies in this task)
  </read_first>
  <action>
    1. **`internal/storage/repo_vault.go`** — fill bodies of stubs left from plan 01:
       ```go
       package storage

       import (
           "database/sql"
           "encoding/json"
           "errors"
           "fmt"
           "time"
       )

       type VaultMetadataRow struct {
           VaultID                string
           SchemaVersion          int
           RecordEnvelopeVersion  int
           CreatedAt, UpdatedAt   time.Time
       }

       type DEKRow struct {
           ID         string
           VaultID    string
           Status     string // 'active' | 'retired'
           CreatedAt  time.Time
           RetiredAt  sql.NullTime
       }

       type KeyWrappingRow struct {
           ID                string
           VaultID           string
           DEKID             string
           WrappingType      string  // 'master_password'
           KDFAlgorithm      string  // 'argon2id'
           KDFParamsJSON     string  // serialized Argon2idParams
           Salt              []byte
           WrappedDEKNonce   []byte
           WrappedDEK        []byte
           CreatedAt, UpdatedAt time.Time
       }

       func InsertVaultMetadata(db *sql.DB, row VaultMetadataRow) error {
           _, err := db.Exec(`INSERT INTO vault_metadata (id, vault_id, schema_version, record_envelope_version, created_at, updated_at)
               VALUES ('vault', ?, ?, ?, ?, ?)`,
               row.VaultID, row.SchemaVersion, row.RecordEnvelopeVersion,
               row.CreatedAt.Format(time.RFC3339Nano), row.UpdatedAt.Format(time.RFC3339Nano))
           return err
       }

       func GetVaultMetadata(db *sql.DB) (VaultMetadataRow, error) {
           var r VaultMetadataRow
           var ca, ua string
           err := db.QueryRow(`SELECT vault_id, schema_version, record_envelope_version, created_at, updated_at FROM vault_metadata WHERE id='vault'`).
               Scan(&r.VaultID, &r.SchemaVersion, &r.RecordEnvelopeVersion, &ca, &ua)
           if err != nil { return r, err }
           r.CreatedAt, _ = time.Parse(time.RFC3339Nano, ca)
           r.UpdatedAt, _ = time.Parse(time.RFC3339Nano, ua)
           return r, nil
       }

       func InsertDEK(db *sql.DB, row DEKRow) error {
           _, err := db.Exec(`INSERT INTO deks (id, vault_id, status, created_at) VALUES (?,?,?,?)`,
               row.ID, row.VaultID, row.Status, row.CreatedAt.Format(time.RFC3339Nano))
           return err
       }

       func GetActiveDEK(db *sql.DB) (DEKRow, error) {
           var r DEKRow
           var ca string
           err := db.QueryRow(`SELECT id, vault_id, status, created_at FROM deks WHERE status='active' LIMIT 1`).
               Scan(&r.ID, &r.VaultID, &r.Status, &ca)
           if err != nil { return r, err }
           r.CreatedAt, _ = time.Parse(time.RFC3339Nano, ca)
           return r, nil
       }

       func InsertKeyWrapping(db *sql.DB, row KeyWrappingRow) error {
           _, err := db.Exec(`INSERT INTO key_wrappings (id, vault_id, dek_id, wrapping_type, kdf_algorithm, kdf_params_json, salt, wrapped_dek_nonce, wrapped_dek, created_at, updated_at)
               VALUES (?,?,?,?,?,?,?,?,?,?,?)`,
               row.ID, row.VaultID, row.DEKID, row.WrappingType, row.KDFAlgorithm, row.KDFParamsJSON,
               row.Salt, row.WrappedDEKNonce, row.WrappedDEK,
               row.CreatedAt.Format(time.RFC3339Nano), row.UpdatedAt.Format(time.RFC3339Nano))
           return err
       }

       func GetKeyWrappingByType(db *sql.DB, wrappingType string) (KeyWrappingRow, error) {
           var r KeyWrappingRow
           var ca, ua string
           err := db.QueryRow(`SELECT id, vault_id, dek_id, wrapping_type, kdf_algorithm, kdf_params_json, salt, wrapped_dek_nonce, wrapped_dek, created_at, updated_at
               FROM key_wrappings WHERE wrapping_type=? LIMIT 1`, wrappingType).
               Scan(&r.ID, &r.VaultID, &r.DEKID, &r.WrappingType, &r.KDFAlgorithm, &r.KDFParamsJSON,
                   &r.Salt, &r.WrappedDEKNonce, &r.WrappedDEK, &ca, &ua)
           if err != nil { return r, err }
           r.CreatedAt, _ = time.Parse(time.RFC3339Nano, ca)
           r.UpdatedAt, _ = time.Parse(time.RFC3339Nano, ua)
           return r, nil
       }

       // SerializeArgon2idParams turns the strongly-typed param struct into the JSON
       // string stored in key_wrappings.kdf_params_json.
       func SerializeArgon2idParams(p any) (string, error) {
           b, err := json.Marshal(p)
           if err != nil { return "", err }
           return string(b), nil
       }
       ```

    2. **`internal/appconfig/config.go`** — D-11 last-vault-path persistence OUTSIDE vault. Use `os.UserConfigDir()` for cross-platform (returns `~/Library/Application Support` on macOS and `%APPDATA%` on Windows):
       ```go
       package appconfig

       import (
           "encoding/json"
           "errors"
           "fmt"
           "os"
           "path/filepath"
       )

       type Config struct {
           LastVaultPath string `json:"last_vault_path,omitempty"`
       }

       const appDirName = "Abyss"
       const fileName   = "config.json"

       // ConfigPath returns the OS-canonical config file location.
       //   macOS:   ~/Library/Application Support/Abyss/config.json
       //   Windows: %APPDATA%\Abyss\config.json
       func ConfigPath() (string, error) {
           base, err := os.UserConfigDir()
           if err != nil { return "", fmt.Errorf("appconfig: user config dir: %w", err) }
           return filepath.Join(base, appDirName, fileName), nil
       }

       // Load reads the config; returns a zero Config (no error) if file does not exist.
       func Load() (Config, error) {
           p, err := ConfigPath()
           if err != nil { return Config{}, err }
           b, err := os.ReadFile(p)
           if err != nil {
               if errors.Is(err, os.ErrNotExist) { return Config{}, nil }
               return Config{}, fmt.Errorf("appconfig: read: %w", err)
           }
           var c Config
           if err := json.Unmarshal(b, &c); err != nil {
               // Corrupt config — return zero rather than block app launch.
               return Config{}, nil
           }
           return c, nil
       }

       // Save writes the config atomically (temp file + rename).
       func Save(c Config) error {
           p, err := ConfigPath()
           if err != nil { return err }
           if err := os.MkdirAll(filepath.Dir(p), 0o755); err != nil {
               return fmt.Errorf("appconfig: mkdir: %w", err)
           }
           b, err := json.MarshalIndent(c, "", "  ")
           if err != nil { return fmt.Errorf("appconfig: marshal: %w", err) }
           tmp := p + ".tmp"
           if err := os.WriteFile(tmp, b, 0o600); err != nil {
               return fmt.Errorf("appconfig: write: %w", err)
           }
           if err := os.Rename(tmp, p); err != nil {
               return fmt.Errorf("appconfig: rename: %w", err)
           }
           return nil
       }
       ```

    3. **`internal/appconfig/config_test.go`**:
       - `TestSaveLoad_RoundTrip`: override XDG/APPDATA env to a `t.TempDir()`, save Config{LastVaultPath: "/some/path"}, Load → matches.
       - `TestLoad_MissingFile_ReturnsZero`: load when file absent → zero value, no error.
       - `TestLoad_CorruptJSON_ReturnsZero`: write garbage to config path, Load → zero value, no error.
       - `TestSave_AtomicViaRename`: after Save, no `.tmp` file remains in directory.
       - `TestConfigPath_ContainsAbyss`: cross-platform: ConfigPath() contains the substring `Abyss` and either `Application Support` (macOS) or `AppData` (Windows) or `.config` (Linux fallback even though Linux is out of scope for runtime).

    4. **`internal/vault/service.go`** — the orchestrator:
       ```go
       package vault

       import (
           "context"
           "database/sql"
           "errors"
           "fmt"
           "time"

           "github.com/google/uuid"

           "<MODULE>/internal/aad"
           "<MODULE>/internal/apperr"
           "<MODULE>/internal/crypto"
           "<MODULE>/internal/session"
           "<MODULE>/internal/storage"
           "<MODULE>/internal/storage/migrations"
       )

       const (
           SchemaVersionV1         = 1
           RecordEnvelopeVersionV1 = 1
       )

       type Service struct {
           db      *sql.DB
           sess    *session.Session
       }

       func NewService(sess *session.Session) *Service {
           return &Service{sess: sess}
       }

       func (s *Service) DB() *sql.DB { return s.db }

       type StatusInfo struct {
           IsOpen, IsUnlocked   bool
           VaultID, VaultPath   string
           UnlockedAt           time.Time
       }

       func (s *Service) Status() StatusInfo {
           if s.db == nil {
               return StatusInfo{IsOpen: false, IsUnlocked: false}
           }
           return StatusInfo{
               IsOpen:     true,
               IsUnlocked: s.sess.Status() == session.StatusUnlocked,
               VaultID:    s.sess.VaultID(),
               VaultPath:  s.sess.VaultPath(),
           }
       }

       // Create initializes a fresh vault file at path, generates a DEK + KEK, wraps the DEK,
       // inserts the metadata + DEK + key_wrapping rows, and ACTIVATES the session
       // (the vault is unlocked immediately after Create per PRD §16 acceptance scenario).
       func (s *Service) Create(ctx context.Context, path, masterPassword string) (string, error) {
           if len(masterPassword) < 12 {
               return "", apperr.New(apperr.CodeValidationError, "Master password must be at least 12 characters.")
           }
           // Refuse to overwrite an existing file (UX-safe; user picks new name on collision).
           if _, err := os.Stat(path); err == nil {
               return "", apperr.New(apperr.CodeValidationError, "A file already exists at that path. Choose a different name.")
           } else if !errors.Is(err, os.ErrNotExist) {
               return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not check the destination path.")
           }

           db, err := storage.Open(path)
           if err != nil { return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not create the vault file.") }
           if err := migrations.RunMigrations(db); err != nil {
               db.Close()
               return "", apperr.Wrap(apperr.CodeMigrationFailed, err, "Vault could not be opened. Schema migration did not complete cleanly.")
           }
           if err := storage.VerifyApplicationID(db); err != nil {
               db.Close()
               return "", apperr.Wrap(apperr.CodeUnsupportedVault, err, "This file is not an Abyss vault.")
           }

           vaultID := uuid.New().String()
           dekID := uuid.New().String()
           wrappingID := uuid.New().String()
           now := time.Now().UTC()

           // 1. Insert vault_metadata.
           if err := storage.InsertVaultMetadata(db, storage.VaultMetadataRow{
               VaultID: vaultID, SchemaVersion: SchemaVersionV1, RecordEnvelopeVersion: RecordEnvelopeVersionV1,
               CreatedAt: now, UpdatedAt: now,
           }); err != nil {
               db.Close()
               return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not initialize vault metadata.")
           }

           // 2. Generate DEK + insert deks row.
           dek, err := crypto.GenerateDEK()
           if err != nil { db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not generate vault key.") }
           defer crypto.Zero(dek) // local copy zeroed at function return; Activate gets a separate copy below
           if err := storage.InsertDEK(db, storage.DEKRow{
               ID: dekID, VaultID: vaultID, Status: "active", CreatedAt: now,
           }); err != nil {
               db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not store vault key metadata.")
           }

           // 3. Derive KEK from master password + random salt.
           params := crypto.DefaultArgon2idParams
           salt, err := crypto.GenerateSalt(params)
           if err != nil { db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not generate salt.") }
           pwBytes := []byte(masterPassword)  // immutable string -> mutable bytes for KDF
           defer crypto.Zero(pwBytes)
           kek, err := crypto.DeriveKEK(pwBytes, salt, params)
           if err != nil { db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "Key derivation failed.") }
           defer crypto.Zero(kek)

           // 4. Wrap DEK using KEK + wrapped-DEK AAD.
           a, err := aad.Build(aad.KindWrappedDEK, vaultID, "", "", dekID, "master_password", 0)
           if err != nil { db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "AAD construction failed.") }
           wrapped, nonce, err := crypto.WrapDEK(kek, dek, a)
           if err != nil { db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "Key wrapping failed.") }

           // 5. Persist key_wrapping row.
           paramsJSON, err := storage.SerializeArgon2idParams(params)
           if err != nil { db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not serialize KDF params.") }
           if err := storage.InsertKeyWrapping(db, storage.KeyWrappingRow{
               ID: wrappingID, VaultID: vaultID, DEKID: dekID, WrappingType: "master_password",
               KDFAlgorithm: "argon2id", KDFParamsJSON: paramsJSON,
               Salt: salt, WrappedDEKNonce: nonce, WrappedDEK: wrapped,
               CreatedAt: now, UpdatedAt: now,
           }); err != nil {
               db.Close(); return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not store key wrapping.")
           }

           // 6. Activate session — caller now has an unlocked vault.
           //    Pass a FRESH copy because Activate takes ownership and Lock zeros it.
           sessDEK := make([]byte, len(dek)); copy(sessDEK, dek)
           if err := s.sess.Activate(sessDEK, vaultID, path); err != nil {
               db.Close(); return "", err  // already typed
           }
           s.db = db
           return vaultID, nil
       }

       // Open verifies the vault file, runs pending migrations, and transitions the session to Locked
       // (DEK is NOT yet derived). Caller follows up with Unlock(masterPassword).
       func (s *Service) Open(ctx context.Context, path string) (string, error) {
           if _, err := os.Stat(path); errors.Is(err, os.ErrNotExist) {
               return "", apperr.New(apperr.CodeValidationError, "This vault no longer exists at this path.").
                   WithDetails(map[string]string{"reason": "missing_file"})
           }
           db, err := storage.Open(path)
           if err != nil { return "", apperr.Wrap(apperr.CodeInternalError, err, "Could not open the vault file.") }

           if err := storage.VerifyApplicationID(db); err != nil {
               db.Close()
               if errors.Is(err, storage.ErrUnsupportedVault) {
                   return "", apperr.Wrap(apperr.CodeUnsupportedVault, err, "This file is not an Abyss vault.")
               }
               return "", apperr.Wrap(apperr.CodeCorruptVault, err, "This vault file appears to be corrupted. Restore from a backup if you have one.")
           }

           if err := migrations.RunMigrations(db); err != nil {
               db.Close()
               return "", apperr.Wrap(apperr.CodeMigrationFailed, err, "Vault could not be opened. Schema migration did not complete cleanly.")
           }

           md, err := storage.GetVaultMetadata(db)
           if err != nil {
               db.Close()
               return "", apperr.Wrap(apperr.CodeCorruptVault, err, "This vault file appears to be corrupted. Restore from a backup if you have one.")
           }

           s.db = db
           s.sess.SetLockedFor(md.VaultID, path)
           return md.VaultID, nil
       }

       // Unlock derives KEK from master password + stored salt, unwraps the DEK with the
       // wrapped-DEK AAD, and activates the session. Any failure -> generic UNLOCK_FAILED
       // (PRD §9.4 — no oracle).
       func (s *Service) Unlock(ctx context.Context, masterPassword string) error {
           if s.db == nil {
               return apperr.New(apperr.CodeInternalError, "No vault is open.")
           }
           if s.sess.Status() == session.StatusUnlocked {
               return apperr.New(apperr.CodeVaultAlreadyUnlocked, "Vault is already unlocked.")
           }

           wrap, err := storage.GetKeyWrappingByType(s.db, "master_password")
           if err != nil {
               // Treat as generic unlock failure — file may be malformed.
               return apperr.Wrap(apperr.CodeUnlockFailed, err, "Unable to unlock vault. Check your master password and try again.")
           }

           var params crypto.Argon2idParams
           if err := json.Unmarshal([]byte(wrap.KDFParamsJSON), &params); err != nil {
               return apperr.Wrap(apperr.CodeUnlockFailed, err, "Unable to unlock vault. Check your master password and try again.")
           }
           if params.Algorithm != "argon2id" {
               return apperr.New(apperr.CodeUnlockFailed, "Unable to unlock vault. Check your master password and try again.")
           }

           pwBytes := []byte(masterPassword)
           defer crypto.Zero(pwBytes)
           kek, err := crypto.DeriveKEK(pwBytes, wrap.Salt, params)
           if err != nil {
               return apperr.Wrap(apperr.CodeUnlockFailed, err, "Unable to unlock vault. Check your master password and try again.")
           }
           defer crypto.Zero(kek)

           a, err := aad.Build(aad.KindWrappedDEK, s.sess.VaultID(), "", "", wrap.DEKID, "master_password", 0)
           if err != nil {
               return apperr.Wrap(apperr.CodeInternalError, err, "AAD construction failed.")
           }

           dek, err := crypto.UnwrapDEK(kek, wrap.WrappedDEK, wrap.WrappedDEKNonce, a)
           if err != nil {
               // Generic failure — wrong password, AAD mismatch, corrupt blob all look identical.
               return apperr.New(apperr.CodeUnlockFailed, "Unable to unlock vault. Check your master password and try again.")
           }

           // Activate transfers ownership of dek; Lock will Zero it.
           if err := s.sess.Activate(dek, s.sess.VaultID(), s.sess.VaultPath()); err != nil {
               return err
           }
           return nil
       }

       func (s *Service) Lock(ctx context.Context) error {
           return s.sess.Lock()
       }
       ```

    5. **`internal/vault/service_test.go`** — end-to-end:
       - `TestService_Create_NewVault_ProducesUnlockedSession`: tmp path, Create with master password "correctpassword12!" → returns vaultID; sess.Status() == StatusUnlocked.
       - `TestService_Create_RejectsShortPassword`: Create with "short" → returns AppError{CodeValidationError}.
       - `TestService_Create_RejectsExistingFile`: create file at path manually, Create → returns AppError{CodeValidationError, message about existing file}.
       - `TestService_Create_PersistsMetadataDEKWrapping`: after Create, query vault_metadata: 1 row with vaultID; query deks: 1 row status=active with the right dek_id; query key_wrappings: 1 row wrapping_type=master_password with non-empty salt, wrapped_dek_nonce, wrapped_dek.
       - `TestService_OpenUnlock_RoundTrip`: Create vault, sess.Lock(), close service, instantiate fresh service, Open(path), Unlock(correct password) → sess.Status() == StatusUnlocked AND DEK matches the original (same plaintext).
       - `TestService_Unlock_WrongPassword_ReturnsUnlockFailed`: Create, Lock, Unlock("wrongpassword12!") → AppError{CodeUnlockFailed} with message "Unable to unlock vault. Check your master password and try again." (PRD-LOCKED).
       - `TestService_Unlock_NoOracle_TimingDifference`: time Unlock with 10 random wrong passwords vs 10 correct attempts on a fresh vault. Means should differ by less than 50% (Argon2id dominates either way; this is a smoke test that we don't return early on a check that bypasses KDF).
       - `TestService_Open_NonAbyssFile_ReturnsUnsupportedVault`: create a SQLite file with `application_id = 0`, Open → AppError{CodeUnsupportedVault, "This file is not an Abyss vault."}.
       - `TestService_Open_MissingFile_ReturnsValidationError`: Open("/nonexistent/path") → AppError{CodeValidationError, "This vault no longer exists at this path."}, with details.reason == "missing_file".
       - `TestService_Lock_TransitionsAndZerosDEK`: Create, snapshot session.DEK() → non-nil 32 bytes; call Lock; session.DEK() → nil; sess.Status() == StatusLocked.
       - `TestService_Unlock_AAD_Tamper_FailsGenerically`: Create, Lock, **manually** tamper the wrapped_dek_nonce bytes in the DB by 1 bit, Unlock(correct password) → AppError{CodeUnlockFailed} (NOT a different code). Proves AAD-mismatch-as-oracle is closed.
       - `TestService_Create_FirstSealUsesAADBuild`: Create vault. Then read raw `wrapped_dek` from the DB and confirm it cannot be decrypted with `crypto.Decrypt(kek, wrapped, nonce, []byte("wrong-aad"))`. The fact that the wrapped DEK was successfully unwrapped on Unlock with `aad.Build(KindWrappedDEK, vaultID, "", "", dekID, "master_password", 0)` proves the encrypt path used the same Build call (architectural compliance — the FIRST cipher.Seal in the codebase, here in WrapDEK, is fed by aad.Build).
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss &amp;&amp; go test ./internal/vault/... ./internal/appconfig/... ./internal/storage/... -race -count=1 -timeout 180s</automated>
  </verify>
  <acceptance_criteria>
    - `internal/vault/service.go` exports `Service`, `NewService`, methods `Create`, `Open`, `Unlock`, `Lock`, `Status`, `DB`
    - `internal/vault/service.go` calls `aad.Build(aad.KindWrappedDEK, ...)` in BOTH Create (wrap) and Unlock (unwrap) paths (verify: `grep -c 'aad.Build' internal/vault/service.go` returns &gt;= 2)
    - `internal/vault/service.go` defers `crypto.Zero(kek)` and `crypto.Zero(pwBytes)` in both Create and Unlock (verify: `grep -c 'crypto.Zero' internal/vault/service.go` returns &gt;= 4)
    - `internal/appconfig/config.go` exports `Config`, `Load`, `Save`, `ConfigPath`
    - `internal/appconfig/config.go` uses `os.UserConfigDir()` (D-11 cross-platform requirement; verify: `grep -c 'UserConfigDir' internal/appconfig/config.go` returns 1)
    - `go test ./internal/vault -run TestService_Create_NewVault_ProducesUnlockedSession -count=1` exits 0
    - `go test ./internal/vault -run TestService_OpenUnlock_RoundTrip -count=1` exits 0
    - `go test ./internal/vault -run TestService_Unlock_WrongPassword_ReturnsUnlockFailed -count=1` exits 0
    - `go test ./internal/vault -run TestService_Unlock_AAD_Tamper_FailsGenerically -count=1` exits 0
    - `go test ./internal/vault -run TestService_Lock_TransitionsAndZerosDEK -count=1` exits 0
    - `go test ./internal/appconfig -count=1` exits 0
    - `golangci-lint run ./...` exits 0
    - `grep -rn 'math/rand' internal/ --include="*.go"` returns no matches
  </acceptance_criteria>
  <done>vault.Service end-to-end Create/Open/Unlock/Lock works with the AAD helper feeding both encrypt and decrypt paths; PRD §9.4 generic unlock failure proven; cross-platform appconfig persists last_vault_path.</done>
</task>

<task type="auto">
  <name>Task 2: Wire Wails App struct (14 bound methods) + main.go startup + plan-3 stubs for record/generator/clipboard methods + scripts/check-wails-bindings.sh + CI no-drift gate</name>
  <files>app.go, main.go, scripts/check-wails-bindings.sh, .github/workflows/ci.yml</files>
  <read_first>
    - docs/PRD.md §10 (entire section — DTO TS shapes for every method this task implements)
    - docs/PRD.md §10.1 (AppErrorCode enum), §10.2 (Vault DTOs + 6 methods), §10.3 (Record DTOs + 6 methods), §10.4 (GeneratePassword DTO), §10.6 (Clipboard DTOs)
    - .planning/research/ARCHITECTURE.md §1.2 (app.go responsibility — thin façade), §11.1 (anti-pattern: fat app.go)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-02 (wails generate module no-drift check lands in plan 3 alongside first DTO definitions), D-03 (Phase 1 API surface is credential-only — DTO shape lands now, plan 04 fills implementation)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md (AppErrorCode → User-Facing String Map; the 14 method calls that the frontend will make)
    - internal/vault/service.go (just created — Create/Open/Unlock/Lock/Status/DB exported)
    - internal/session/session.go (RequireUnlocked, Status, VaultPath)
    - internal/apperr/error.go (ToDTO, AppError, codes)
    - internal/logger/logger.go (logger.New)
    - internal/appconfig/config.go (Load, Save, Config)
    - app.go (existing scaffold from plan 01 — empty App struct)
    - main.go (existing scaffold from plan 01)
  </read_first>
  <action>
    1. **`app.go`** — replace the scaffold with the full 14-method binding façade:
       ```go
       package main

       import (
           "context"

           "github.com/wailsapp/wails/v2/pkg/runtime"

           "<MODULE>/internal/appconfig"
           "<MODULE>/internal/apperr"
           "<MODULE>/internal/logger"
           "<MODULE>/internal/session"
           "<MODULE>/internal/vault"
       )

       // -------------------- Frontend-facing DTOs --------------------
       // These struct tags drive what `wails generate module` exports to TS.
       // Field names use camelCase JSON tags to match PRD §10 TS contract.

       type CreateVaultRequest struct {
           Path                  string `json:"path"`
           MasterPassword        string `json:"masterPassword"`
           ConfirmMasterPassword string `json:"confirmMasterPassword"`
       }
       type OpenVaultRequest   struct { Path string `json:"path"` }
       type UnlockVaultRequest struct {
           Path           string `json:"path"`
           MasterPassword string `json:"masterPassword"`
       }
       type VaultStatusDTO struct {
           IsOpen     bool   `json:"isOpen"`
           IsUnlocked bool   `json:"isUnlocked"`
           VaultPath  string `json:"vaultPath,omitempty"`
           VaultID    string `json:"vaultId,omitempty"`
           UnlockedAt string `json:"unlockedAt,omitempty"`
       }
       type VaultSessionDTO struct {
           VaultID                 string `json:"vaultId"`
           VaultPath               string `json:"vaultPath"`
           UnlockedAt              string `json:"unlockedAt"`
           AutoLockTimeoutSeconds  int    `json:"autoLockTimeoutSeconds"`
       }

       type RecordSummaryDTO struct {
           ID                   string   `json:"id"`
           Type                 string   `json:"type"` // "credential" | ...
           Title                string   `json:"title"`
           Subtitle             string   `json:"subtitle,omitempty"`
           Tags                 []string `json:"tags,omitempty"`
           PublicKeyFingerprint string   `json:"publicKeyFingerprint,omitempty"`
           CreatedAt            string   `json:"createdAt"`
           UpdatedAt            string   `json:"updatedAt"`
       }
       type RecordDTO struct {
           ID                   string                 `json:"id"`
           Type                 string                 `json:"type"`
           Metadata             map[string]any         `json:"metadata"`
           Secret               map[string]any         `json:"secret"`
           PublicKeyOpenSSH     string                 `json:"publicKeyOpenSSH,omitempty"`
           PublicKeyPEM         string                 `json:"publicKeyPEM,omitempty"`
           PublicKeyFingerprint string                 `json:"publicKeyFingerprint,omitempty"`
           CreatedAt            string                 `json:"createdAt"`
           UpdatedAt            string                 `json:"updatedAt"`
       }
       type CreateRecordRequest struct {
           Type                 string                 `json:"type"`
           Metadata             map[string]any         `json:"metadata"`
           Secret               map[string]any         `json:"secret"`
           PublicKeyOpenSSH     string                 `json:"publicKeyOpenSSH,omitempty"`
           PublicKeyPEM         string                 `json:"publicKeyPEM,omitempty"`
           PublicKeyFingerprint string                 `json:"publicKeyFingerprint,omitempty"`
       }
       type UpdateRecordRequest struct {
           ID                   string                 `json:"id"`
           Metadata             map[string]any         `json:"metadata"`
           Secret               map[string]any         `json:"secret"`
           PublicKeyOpenSSH     string                 `json:"publicKeyOpenSSH,omitempty"`
           PublicKeyPEM         string                 `json:"publicKeyPEM,omitempty"`
           PublicKeyFingerprint string                 `json:"publicKeyFingerprint,omitempty"`
       }
       type SearchRecordsRequest struct {
           Query      string   `json:"query,omitempty"`
           RecordType string   `json:"recordType,omitempty"`
           Tags       []string `json:"tags,omitempty"`
       }

       type GeneratePasswordRequest struct {
           Length            int  `json:"length"`
           IncludeUppercase  bool `json:"includeUppercase"`
           IncludeLowercase  bool `json:"includeLowercase"`
           IncludeNumbers    bool `json:"includeNumbers"`
           IncludeSymbols    bool `json:"includeSymbols"`
           ExcludeAmbiguous  bool `json:"excludeAmbiguous"`
       }
       type GeneratedPasswordDTO struct {
           Password string `json:"password"`
           Length   int    `json:"length"`
       }

       type CopyToClipboardRequest struct {
           Value      string `json:"value"`
           SecretKind string `json:"secretKind"` // "password"|"api_key"|"aes_key"|"public_key"|"private_key"|"secure_note"
           TimeoutMS  int    `json:"timeoutMs,omitempty"`
       }

       type AppErrorDTO = apperr.DTO  // re-exposed under the package's name space

       // -------------------- App --------------------

       type App struct {
           ctx     context.Context
           log     *slog.Logger
           sess    *session.Session
           vault   *vault.Service
           appcfg  appconfig.Config
       }

       func NewApp() *App {
           sess := session.New()
           return &App{
               log:   logger.New(),
               sess:  sess,
               vault: vault.NewService(sess),
           }
       }

       func (a *App) OnStartup(ctx context.Context) {
           a.ctx = ctx
           cfg, err := appconfig.Load()
           if err != nil { a.log.Warn("appconfig: load failed", "err", err) }
           a.appcfg = cfg
           a.log.Info("app started")
       }

       // ----- 14 bound methods (PRD §10) -----

       func (a *App) CreateVault(req CreateVaultRequest) (*VaultSessionDTO, *AppErrorDTO) {
           if req.MasterPassword != req.ConfirmMasterPassword {
               return nil, apperr.ToDTO(apperr.New(apperr.CodeValidationError, "Passwords do not match.").
                   WithDetails(map[string]string{"field": "confirmMasterPassword"}))
           }
           vid, err := a.vault.Create(a.ctx, req.Path, req.MasterPassword)
           if err != nil { return nil, apperr.ToDTO(err) }
           // Persist last_vault_path (D-11).
           a.appcfg.LastVaultPath = req.Path
           if saveErr := appconfig.Save(a.appcfg); saveErr != nil {
               a.log.Warn("appconfig: save failed", "err", saveErr)
           }
           return &VaultSessionDTO{
               VaultID: vid, VaultPath: req.Path,
               UnlockedAt: time.Now().UTC().Format(time.RFC3339Nano),
               AutoLockTimeoutSeconds: 0, // Phase 2 sets this from settings; 0 = disabled in v0.1
           }, nil
       }

       func (a *App) OpenVault(req OpenVaultRequest) (*VaultStatusDTO, *AppErrorDTO) {
           _, err := a.vault.Open(a.ctx, req.Path)
           if err != nil { return nil, apperr.ToDTO(err) }
           // Persist last_vault_path here too (D-11): user opened a different vault from disk.
           a.appcfg.LastVaultPath = req.Path
           _ = appconfig.Save(a.appcfg)
           st := a.vault.Status()
           return &VaultStatusDTO{IsOpen: st.IsOpen, IsUnlocked: st.IsUnlocked, VaultPath: st.VaultPath, VaultID: st.VaultID}, nil
       }

       func (a *App) UnlockVault(req UnlockVaultRequest) (*VaultSessionDTO, *AppErrorDTO) {
           // If a different vault is currently open, close it first.
           cur := a.vault.Status()
           if cur.IsOpen && cur.VaultPath != req.Path {
               _ = a.vault.Lock(a.ctx)
               if a.vault.DB() != nil { _ = a.vault.DB().Close() }
               // Re-open the requested vault.
               if _, err := a.vault.Open(a.ctx, req.Path); err != nil {
                   return nil, apperr.ToDTO(err)
               }
           } else if !cur.IsOpen {
               if _, err := a.vault.Open(a.ctx, req.Path); err != nil {
                   return nil, apperr.ToDTO(err)
               }
           }
           if err := a.vault.Unlock(a.ctx, req.MasterPassword); err != nil {
               return nil, apperr.ToDTO(err)
           }
           a.appcfg.LastVaultPath = req.Path
           _ = appconfig.Save(a.appcfg)
           st := a.vault.Status()
           return &VaultSessionDTO{
               VaultID: st.VaultID, VaultPath: st.VaultPath,
               UnlockedAt: time.Now().UTC().Format(time.RFC3339Nano),
               AutoLockTimeoutSeconds: 0,
           }, nil
       }

       func (a *App) LockVault() *AppErrorDTO {
           if err := a.vault.Lock(a.ctx); err != nil { return apperr.ToDTO(err) }
           // Emit vault:locked event so frontend can clear its stores + redirect to /unlock.
           runtime.EventsEmit(a.ctx, "vault:locked")
           return nil
       }

       func (a *App) GetVaultStatus() (*VaultStatusDTO, *AppErrorDTO) {
           st := a.vault.Status()
           return &VaultStatusDTO{
               IsOpen: st.IsOpen, IsUnlocked: st.IsUnlocked,
               VaultPath: st.VaultPath, VaultID: st.VaultID,
           }, nil
       }

       // --- Records (Phase 1: credential-only, plan 04 fills implementations) ---

       func (a *App) ListRecords() ([]*RecordSummaryDTO, *AppErrorDTO) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, apperr.ToDTO(err) }
           // Plan 04 wires records.Service. v0.1 plan-3 returns empty slice (Vault Home shows empty state).
           return []*RecordSummaryDTO{}, nil
       }

       func (a *App) GetRecord(id string) (*RecordDTO, *AppErrorDTO) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, apperr.ToDTO(err) }
           // Plan 04 fills. For now: not found.
           return nil, apperr.ToDTO(apperr.New(apperr.CodeRecordNotFound, "This record no longer exists. It may have been deleted."))
       }

       func (a *App) CreateRecord(req CreateRecordRequest) (*RecordDTO, *AppErrorDTO) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, apperr.ToDTO(err) }
           // Plan 04 implements; for now signal not-implemented as INTERNAL_ERROR.
           return nil, apperr.ToDTO(apperr.New(apperr.CodeInternalError, "Record creation is not yet implemented."))
       }

       func (a *App) UpdateRecord(req UpdateRecordRequest) (*RecordDTO, *AppErrorDTO) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, apperr.ToDTO(err) }
           return nil, apperr.ToDTO(apperr.New(apperr.CodeInternalError, "Record update is not yet implemented."))
       }

       func (a *App) DeleteRecord(id string) *AppErrorDTO {
           if err := a.sess.RequireUnlocked(); err != nil { return apperr.ToDTO(err) }
           return apperr.ToDTO(apperr.New(apperr.CodeInternalError, "Record delete is not yet implemented."))
       }

       func (a *App) SearchRecords(req SearchRecordsRequest) ([]*RecordSummaryDTO, *AppErrorDTO) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, apperr.ToDTO(err) }
           return []*RecordSummaryDTO{}, nil
       }

       // --- Generator + Clipboard (Phase 1: stubs; plan 04 fills) ---

       func (a *App) GeneratePassword(req GeneratePasswordRequest) (*GeneratedPasswordDTO, *AppErrorDTO) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, apperr.ToDTO(err) }
           return nil, apperr.ToDTO(apperr.New(apperr.CodeInternalError, "Password generator is not yet implemented."))
       }

       func (a *App) CopyToClipboard(req CopyToClipboardRequest) *AppErrorDTO {
           if err := a.sess.RequireUnlocked(); err != nil { return apperr.ToDTO(err) }
           return apperr.ToDTO(apperr.New(apperr.CodeInternalError, "Clipboard is not yet implemented."))
       }

       func (a *App) ClearClipboard() *AppErrorDTO {
           // No unlock guard — clearing should be safe even if unlocked status raced.
           return apperr.ToDTO(apperr.New(apperr.CodeInternalError, "Clipboard is not yet implemented."))
       }
       ```

       Note the IMPLEMENTED methods for plan 03 are: CreateVault, OpenVault, UnlockVault, LockVault, GetVaultStatus. The 9 remaining methods are stubbed with `INTERNAL_ERROR "...not yet implemented"` (or `RECORD_NOT_FOUND` for GetRecord) so the DTO contract / type generator runs and the frontend wrappers compile. Plan 04 fills them.

    2. **`main.go`** — wire Wails Run with the App struct:
       ```go
       package main

       import (
           "embed"

           "github.com/wailsapp/wails/v2"
           "github.com/wailsapp/wails/v2/pkg/options"
           "github.com/wailsapp/wails/v2/pkg/options/assetserver"
       )

       //go:embed all:frontend/dist
       var assets embed.FS

       func main() {
           app := NewApp()
           err := wails.Run(&options.App{
               Title:  "Abyss",
               Width:  960,
               Height: 640,
               MinWidth:  720,
               MinHeight: 480,
               AssetServer: &assetserver.Options{Assets: assets},
               OnStartup:  app.OnStartup,
               Bind: []interface{}{ app },
           })
           if err != nil { println("Error:", err.Error()) }
       }
       ```

    3. **Generate Wails bindings:** run `wails generate module`. This produces `frontend/wailsjs/go/main/App.{js,d.ts}` with all 14 bound methods plus `frontend/wailsjs/runtime/runtime.{js,d.ts}`. Commit these generated files (we untracked them in plan 01's .gitignore — REVISE the .gitignore to TRACK `frontend/wailsjs/` going forward, because the no-drift CI check needs them in git).

    4. **`scripts/check-wails-bindings.sh`** — D-02 plan-3 CI gate:
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       cd "$(dirname "$0")/.."
       echo "Generating Wails bindings..."
       wails generate module
       if ! git diff --quiet -- frontend/wailsjs/; then
         echo "FAIL: 'wails generate module' produced changes to frontend/wailsjs/ that aren't committed."
         echo "Run 'wails generate module' locally and commit the result."
         git diff --stat -- frontend/wailsjs/
         exit 1
       fi
       echo "OK: bindings are in sync."
       ```
       Make executable: `chmod +x scripts/check-wails-bindings.sh`.

    5. **`.github/workflows/ci.yml`** — extend with the no-drift step (after Go test, before frontend build):
       ```yaml
       - name: Install Wails CLI
         run: go install github.com/wailsapp/wails/v2/cmd/wails@v2.12.0
       - name: Wails bindings no-drift check
         run: ./scripts/check-wails-bindings.sh
       ```
       (Run on `ubuntu-latest` only — Wails CLI on Windows requires WebView2 SDK which is heavy for a binding-gen check.)

       Also extend the existing CI grep gate from plan 01: continue rejecting `mattn/go-sqlite3` AND now also reject any direct import of `frontend/wailsjs/go/main/App` outside `frontend/src/api/`:
       ```yaml
       - name: Frontend api/ wrapper boundary check
         run: |
           if grep -rn 'wailsjs/go/main/App' frontend/src --include='*.ts' --include='*.tsx' | grep -v 'frontend/src/api/'; then
             echo "FAIL: Components/screens are importing Wails bindings directly. Use frontend/src/api/ wrappers."
             exit 1
           fi
       ```

    6. Commit: `feat(01-03): wire App struct + 14 Wails-bound methods + bindings no-drift CI gate`.
  </action>

</automated>
  </verify>
  <acceptance_criteria>
    - `app.go` declares struct `App` with field `vault *vault.Service`, `sess *session.Session`, `log *slog.Logger`
    - `app.go` declares EXACTLY 14 exported methods: CreateVault, OpenVault, UnlockVault, LockVault, GetVaultStatus, ListRecords, GetRecord, CreateRecord, UpdateRecord, DeleteRecord, SearchRecords, GeneratePassword, CopyToClipboard, ClearClipboard (verify: `grep -c '^func (a \*App)' app.go` returns `14`)
    - Every secret-touching method calls `a.sess.RequireUnlocked()` before any other work (verify: `grep -c 'RequireUnlocked' app.go` returns at least 9 — the 9 record/generator/clipboard methods)
    - Every error returned passes through `apperr.ToDTO` (verify: `grep -c 'apperr.ToDTO' app.go` returns at least 14)
    - `LockVault` calls `runtime.EventsEmit(a.ctx, "vault:locked")` (verify: `grep -c 'EventsEmit.*vault:locked' app.go` returns 1)
    - `frontend/wailsjs/go/main/App.{js,d.ts}` exists after running `wails generate module`
    - `scripts/check-wails-bindings.sh` is executable (`test -x scripts/check-wails-bindings.sh`)
    - `scripts/check-wails-bindings.sh` exits 0 (no drift between current Go DTOs and committed bindings)
    - `.github/workflows/ci.yml` contains the literal string `wails generate module` (verify: `grep -c 'wails generate module' .github/workflows/ci.yml` returns at least 1)
    - `.github/workflows/ci.yml` contains the api/wrapper boundary check (`grep -c 'wailsjs/go/main/App' .github/workflows/ci.yml` returns at least 1)
    - `go build ./...` exits 0
    - `wails build` exits 0
  </acceptance_criteria>
  <done>14 Wails methods bound; 5 are fully implemented (vault lifecycle), 9 are typed-stubs returning AppError DTOs that the frontend will display; the bindings no-drift CI gate is live; the api-wrapper boundary CI gate is live.</done>
</task>

<task type="auto">
  <name>Task 3: Frontend api/ typed-wrapper layer + Zustand stores + 6 Phase-1 screens with React Router + vault:locked event subscription</name>
  <files>frontend/src/types/dto.ts, frontend/src/errors/codes.ts, frontend/src/errors/messages.ts, frontend/src/api/index.ts, frontend/src/api/vault.ts, frontend/src/api/records.ts, frontend/src/api/generator.ts, frontend/src/api/clipboard.ts, frontend/src/api/errors.ts, frontend/src/api/normalizeError.ts, frontend/src/state/vaultStore.ts, frontend/src/state/clipboardStore.ts, frontend/src/App.tsx, frontend/src/router.tsx, frontend/src/screens/Welcome/Welcome.tsx, frontend/src/screens/CreateVault/CreateVault.tsx, frontend/src/screens/UnlockVault/UnlockVault.tsx, frontend/src/screens/VaultHome/VaultHome.tsx, frontend/src/screens/RecordDetail/RecordDetail.tsx, frontend/src/screens/PasswordGenerator/PasswordGenerator.tsx, frontend/src/components/LockBadge/LockBadge.tsx</files>
  <read_first>
    - .planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md (FULL FILE — every PRD-LOCKED copy string, the 12-row AppErrorCode → User-Facing String Map, focal points table, spacing/typography/color tokens, interaction contracts #1-10, accessibility contract, component inventory)
    - docs/PRD.md §9.2-§9.7 (screen-by-screen requirements; especially §9.4 generic unlock failure, §9.5 lock-state visibility)
    - docs/PRD.md §10 (DTO TS shapes — locked by app.go but mirrored here)
    - docs/PRD.md §18 hard rule #14 (no string-matching errors in frontend)
    - .planning/research/ARCHITECTURE.md §4.2 (api/ wrapper layer pattern with normalizeError), §4.3 (hand-curated dto.ts)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-10 (first launch focus on Create), D-11 (subsequent launches → Unlock prefilled), D-13 (Lock routes to Unlock with vault path), D-14 (per-copy timeout menu — DTO contract for plan 04), D-15 (theme = auto, no toggle), D-16 (length-only master pw validator)
    - app.go (just produced — DTO struct field names with json tags)
    - frontend/src/main.tsx (existing from plan 01 — MantineProvider already configured)
    - frontend/wailsjs/go/main/App.d.ts (just generated — the source of truth for what TS sees)
  </read_first>
  <action>
    1. **`frontend/src/types/dto.ts`** — hand-curated TS types mirroring PRD §10 (matching the JSON tags in `app.go`). Export every type listed in the `<interfaces>` block at the top of this plan: `AppErrorCode` (union of 12 strings), `AppErrorDTO`, `CreateVaultRequest`, `OpenVaultRequest`, `UnlockVaultRequest`, `VaultStatusDTO`, `VaultSessionDTO`, `RecordType`, `RecordSummaryDTO`, `RecordDTO`, `CreateRecordRequest`, `UpdateRecordRequest`, `SearchRecordsRequest`, `GeneratePasswordRequest`, `GeneratedPasswordDTO`, `SecretKind`, `CopyToClipboardRequest`. Field names use camelCase (matching the JSON tags on the Go side).

    2. **`frontend/src/errors/codes.ts`** — re-export the AppErrorCode union plus a runtime `KNOWN_CODES` ReadonlySet for exhaustiveness checks.

    3. **`frontend/src/errors/messages.ts`** — locks the 12-row map from UI-SPEC §AppErrorCode → User-Facing String Map verbatim. Critical strings (must be byte-exact):
       - `UNLOCK_FAILED` → `Unable to unlock vault. Check your master password and try again.` (PRD-LOCKED)
       - `UNSUPPORTED_VAULT` → `This file is not an Abyss vault.`
       - `CORRUPT_VAULT` → `This vault file appears to be corrupted. Restore from a backup if you have one.`
       - `MIGRATION_FAILED` → `Vault could not be opened. Schema migration did not complete cleanly.`
       - `VAULT_LOCKED` → `Vault is locked. Unlock to continue.`
       - `VAULT_ALREADY_UNLOCKED` → `Vault is already unlocked.`
       - `RECORD_NOT_FOUND` → `This record no longer exists. It may have been deleted.`
       - `EXPORT_FAILED` → `Could not export. Try again or check the destination folder.`
       - `CLIPBOARD_FAILED` → `Could not copy to clipboard. Try again.`
       - `PLATFORM_ERROR` → `Something went wrong on your system. Try again.`
       - `INTERNAL_ERROR` → `Something went wrong. Try again.`
       - `VALIDATION_ERROR` → `Some fields need attention.` (default; if `details.reason` or `details.message` is present, use that instead)

       Export `userFacingMessage(err: AppErrorDTO): string` that does the lookup. The function MUST NOT consult `err.message` for any code other than reading `details.reason` / `details.message` for VALIDATION_ERROR.

    4. **`frontend/src/api/normalizeError.ts`** — single sanitization point that NEVER throws:
       - If `e` is an object with `code` property AND that code is in `KNOWN_CODES`, return `{code, message: e.message ?? "", details: e.details}`.
       - Otherwise return `{code: "INTERNAL_ERROR", message: "Something went wrong. Try again."}`.

    5. **`frontend/src/api/vault.ts`** — typed wrappers for the 5 vault methods using the pattern: import the raw method from `../../wailsjs/go/main/App` aliased as `RawX`, define an exported async function with the proper request/response types, wrap the call in try/catch and rethrow `normalizeError(e)`. Repeat for createVault, openVault, unlockVault, lockVault, getVaultStatus.

    6. **`frontend/src/api/records.ts`** — wrappers for ListRecords, GetRecord, CreateRecord, UpdateRecord, DeleteRecord, SearchRecords. Same try/normalize pattern.

    7. **`frontend/src/api/generator.ts`** — wraps `GeneratePassword`.

    8. **`frontend/src/api/clipboard.ts`** — wraps `CopyToClipboard`, `ClearClipboard`.

    9. **`frontend/src/api/errors.ts`** — re-exports `normalizeError` and `userFacingMessage` for screens that handle errors (most screens import `userFacingMessage` from here).

    10. **`frontend/src/api/index.ts`** — `export * from "./vault"; export * from "./records"; export * from "./generator"; export * from "./clipboard"; export * from "./errors";`. Single import point for screens.

    11. **`frontend/src/state/vaultStore.ts`** — Zustand store with fields `status: "closed"|"locked"|"unlocked"`, `vaultId?: string`, `vaultPath?: string`, `unlockedAt?: string`, plus actions `setFromStatus(s: VaultStatusDTO)`, `setUnlocked(vaultId, vaultPath, unlockedAt)`, `reset()`.

    12. **`frontend/src/state/clipboardStore.ts`** — Zustand store with fields `active: boolean`, `secretKind?: SecretKind`, `startedAt?: number`, `timeoutMs?: number`, plus actions `start(secretKind, timeoutMs)`, `stop()`. Plan 04 wires the actual countdown UI.

    13. **`frontend/src/components/LockBadge/LockBadge.tsx`** — UI-SPEC lock-state pill: Mantine `<Badge size="sm" leftSection={<IconShield/>}>` with `color="green"` for Locked, `color="yellow"` for Unlocked. Always rendered with `role="status" aria-live="polite"` per UI-SPEC accessibility contract. Subscribes to `useVaultStore`.

    14. **`frontend/src/screens/Welcome/Welcome.tsx`** — per UI-SPEC §Welcome Screen copy table (verbatim):
        - Heading `<Title order={1}>Abyss</Title>`
        - Subhead: `Local password and key vault. Stays on your machine.`
        - Primary CTA `Create new vault` (filled blue) → routes to `/create`
        - Secondary CTA `Open existing vault` → opens file dialog (use Wails runtime `OpenFileDialog`) → routes to `/unlock?path=<chosen>`
        - Tertiary link `View security limitations` → opens minimal Mantine `<Modal>` titled `Security limitations` with the 3 verbatim PRD §4.4 bullets from UI-SPEC §"View security limitations" copy table. Footer CTA: `Got it`
        - Footer note: `Learning-grade. Not a substitute for a hardened password manager.` (Label, dimmed)
        - Auto-focus per D-10: on first launch (when `vaultStore.status === "closed"` AND no last_vault_path was returned by GetVaultStatus), focus the `Create new vault` button via `useEffect` + ref.

    15. **`frontend/src/screens/CreateVault/CreateVault.tsx`** — per UI-SPEC §Create Vault Screen + PRD §9.3:
        - Heading `Create your vault`
        - Vault file location: `<TextInput>` with helper `Default: ~/Documents/Abyss/vault.abyssvault` (D-12). For v0.1, also support a `Choose…` button that opens Wails `SaveFileDialog` and pre-fills the field.
        - Master password: `<PasswordInput>` (Mantine built-in reveal toggle — UI-SPEC §Component Inventory)
        - Counter `{n} / 12 minimum` rendered in `<Text size="sm" c={n < 12 ? "red" : "dimmed"} style={{fontVariantNumeric: "tabular-nums"}}>` (D-16: length-only validator, NO zxcvbn)
        - Confirm master password: `<PasswordInput>`; inline error `Passwords do not match.`
        - PRD-LOCKED no-recovery warning displayed verbatim above the checkbox: `There is no password recovery. If you lose your master password, the vault cannot be recovered.`
        - Acknowledgement checkbox: `I understand there is no password recovery.`
        - Primary CTA `Create vault` — disabled until pw≥12 AND match AND ack-checked
        - Cancel link `Back` → `/welcome`
        - On submit: `await api.createVault({ path, masterPassword, confirmMasterPassword })`. On success: `vaultStore.setUnlocked(...)` + `navigate("/vault")`. On error: render `userFacingMessage(err)` in an inline alert (Mantine `<Alert color="red">`).

    16. **`frontend/src/screens/UnlockVault/UnlockVault.tsx`** — per UI-SPEC §Unlock Screen + PRD §9.4:
        - Heading `Unlock vault`
        - Vault path display (Label, dimmed): the path from URL query param OR vaultStore.vaultPath
        - "Open a different vault" link → file dialog → re-route with new path
        - Master password: `<PasswordInput>` autofocused on mount (UI-SPEC interaction contract #7)
        - Primary CTA `Unlock vault`
        - On submit: `await api.unlockVault({ path, masterPassword })`. On success: `vaultStore.setUnlocked(...)` + `navigate("/vault")`.
        - **Error handling (CRITICAL — UI-SPEC §Unlock Screen rule):** for ANY `AppErrorCode` returned by UnlockVault that is NOT `VAULT_ALREADY_UNLOCKED` and NOT `UNSUPPORTED_VAULT` and NOT `CORRUPT_VAULT` and NOT `MIGRATION_FAILED`, display the PRD-LOCKED string `Unable to unlock vault. Check your master password and try again.` Do NOT display `err.message`.
        - For `UNSUPPORTED_VAULT`, `CORRUPT_VAULT`, `MIGRATION_FAILED` → display the matching `userFacingMessage(err)` in a banner (these are NOT wrong-password failures).
        - For `VALIDATION_ERROR` with `err.details?.reason === "missing_file"` → display info banner `This vault is no longer at this path. Open a different one.`

    17. **`frontend/src/screens/VaultHome/VaultHome.tsx`** — per UI-SPEC §Vault Home + PRD §9.5:
        - Header: `<Group justify="space-between">` with title `Vault` (left), `<LockBadge/>` + `Lock vault` button (right).
        - Search input placeholder `Search records…` (renders, no behavior in plan 03).
        - Record-type filter (Phase 1: only `All types` and `Credentials` per UI-SPEC).
        - Record list: empty state copy `No records yet` / `Create your first credential, or generate a password to get started.` + primary CTA `New record` + secondary CTA `Generate password`.
        - Toolbar buttons `New record` / `Generate password`.
        - Lock button calls `api.lockVault()`; the `vault:locked` event handler in App.tsx will redirect.
        - Lock guard: if `vaultStore.status !== "unlocked"`, navigate to `/unlock` (UI-SPEC interaction contract #6).

    18. **`frontend/src/screens/RecordDetail/RecordDetail.tsx`** — placeholder for plan 04 fill. Calls `api.getRecord(id)` which returns RECORD_NOT_FOUND in plan 03; displays the user-facing string and a `Back to Vault Home` link. Includes header with `<LockBadge/>`.

    19. **`frontend/src/screens/PasswordGenerator/PasswordGenerator.tsx`** — per UI-SPEC §Password Generator copy table. Renders the form (length slider 8-128 default 24, class checkboxes with UI-SPEC labels, exclude-ambiguous toggle with UI-SPEC label `Exclude ambiguous characters (O 0 l 1 I)`, generated output field in monospace, `Generate again` button, primary `Copy` button, secondary `Save as credential…` button). Generate button calls `api.generatePassword(req)` which returns INTERNAL_ERROR in plan 03 — display the toast `Something went wrong. Try again.` (this proves the wiring is correct; plan 04 implements the backend). Includes header with `<LockBadge/>`.

    20. **`frontend/src/router.tsx`** — React Router v6 `createBrowserRouter` with routes:
        - `/` → redirect to `/welcome`
        - `/welcome` → `<Welcome/>`
        - `/create` → `<CreateVault/>`
        - `/unlock` → `<UnlockVault/>`
        - `/vault` → `<VaultHome/>`
        - `/record/:id` → `<RecordDetail/>`
        - `/generator` → `<PasswordGenerator/>`

    21. **`frontend/src/App.tsx`** — replace the plan-01 placeholder. Render `<RouterProvider router={router}/>`. In a `useEffect`:
        - Call `getVaultStatus()` once on mount; pipe result into `vaultStore.setFromStatus`. If status is `"locked"` AND a vaultPath is set, navigate to `/unlock`. Otherwise stay on `/welcome`.
        - Subscribe to backend `vault:locked` event via `runtime.EventsOn("vault:locked", () => { vaultStore.reset(); clipboardStore.stop(); router.navigate("/unlock"); })`. The runtime import is from `../wailsjs/runtime/runtime` (allowed by the api-wrapper boundary check; only `wailsjs/go/main/App` is gated).
        - On unmount, call the unsubscribe function returned by `EventsOn`.

    22. Run `cd frontend &amp;&amp; npm run build` — must compile with zero errors.

    23. Manual smoke on macOS (`wails dev`):
        - App launches → Welcome screen renders with "Abyss" hero, three CTAs, footer note.
        - Click `Create new vault` → routes to `/create`.
        - Type a path (e.g. `/tmp/test.abyssvault`), master password `correctpassword12!`, confirm same, check ack box → `Create vault` button enables → click.
        - Vault Home renders with yellow `Unlocked` pill (vault is unlocked immediately after Create per PRD §16 acceptance scenario).
        - Click `Lock vault` → routes back to `/unlock` (vault:locked event fires; vaultStore reset; clipboardStore stopped).
        - Type the same password → `Unlock vault` → routes back to Vault Home with yellow `Unlocked` pill.
        - Type wrong password → inline error renders the PRD-LOCKED string `Unable to unlock vault. Check your master password and try again.`

    24. **D-05 cross-OS smoke for plan 03:** on Windows, `git pull`, `wails dev`, repeat the create/unlock/lock cycle. Sign off in `.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md` with date + commit SHA.

    25. Commit: `feat(01-03): frontend api/ wrappers + Zustand stores + 6 Phase-1 screens with vault:locked event`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss/frontend &amp;&amp; npm run build</automated>
  </verify>
  <acceptance_criteria>
    - `frontend/src/types/dto.ts` declares `type AppErrorCode` as a union of EXACTLY 12 string literals (each of the 12 codes appears at least once: `grep -c '"VAULT_LOCKED"' frontend/src/types/dto.ts` &gt;= 1, same pattern for the other 11)
    - `frontend/src/errors/messages.ts` `STATIC_MESSAGES` map declares all 12 codes (verify: `grep -cE '(VAULT_LOCKED|VAULT_ALREADY_UNLOCKED|UNLOCK_FAILED|UNSUPPORTED_VAULT|CORRUPT_VAULT|MIGRATION_FAILED|VALIDATION_ERROR|RECORD_NOT_FOUND|EXPORT_FAILED|CLIPBOARD_FAILED|PLATFORM_ERROR|INTERNAL_ERROR):' frontend/src/errors/messages.ts` returns 12)
    - `frontend/src/errors/messages.ts` UNLOCK_FAILED entry contains EXACTLY the PRD-LOCKED string (verify: `grep -c 'Unable to unlock vault\. Check your master password and try again\.' frontend/src/errors/messages.ts` returns 1)
    - `frontend/src/api/normalizeError.ts` exports `normalizeError`
    - `grep -rn 'wailsjs/go/main/App' frontend/src --include='*.ts' --include='*.tsx' | grep -v 'frontend/src/api/'` returns no matches (api-wrapper boundary holds across the codebase)
    - `grep -rEn '\.message\.(includes|match|indexOf)\(' frontend/src --include='*.ts' --include='*.tsx'` returns no matches (no string-matching errors per PRD §18 hard rule #14)
    - `frontend/src/state/vaultStore.ts` exports `useVaultStore` (Zustand)
    - `frontend/src/state/clipboardStore.ts` exports `useClipboardStore` (Zustand)
    - `frontend/src/App.tsx` calls `EventsOn` with the literal event name `vault:locked` (verify: `grep -c 'EventsOn.*vault:locked' frontend/src/App.tsx` returns 1)
    - All 6 Phase-1 screens exist as files: Welcome.tsx, CreateVault.tsx, UnlockVault.tsx, VaultHome.tsx, RecordDetail.tsx, PasswordGenerator.tsx
    - `frontend/src/screens/CreateVault/CreateVault.tsx` contains the literal PRD-LOCKED no-recovery warning (verify: `grep -c 'There is no password recovery\. If you lose your master password, the vault cannot be recovered\.' frontend/src/screens/CreateVault/CreateVault.tsx` returns 1)
    - `frontend/src/screens/Welcome/Welcome.tsx` contains the literal `Local password and key vault. Stays on your machine.` (UI-SPEC subhead)
    - `frontend/src/screens/Welcome/Welcome.tsx` contains the literal `Learning-grade. Not a substitute for a hardened password manager.` (UI-SPEC footer note)
    - `cd frontend &amp;&amp; npm run build` exits 0
    - `wails build` exits 0
    - Manual smoke: `wails dev` on macOS, full Create/Lock/Unlock cycle works end-to-end (sign-off recorded in 01-VERIFICATION.md with commit SHA)
    - D-05 Windows smoke: same lifecycle works on Windows (sign-off recorded in 01-VERIFICATION.md)
  </acceptance_criteria>
  <done>End-to-end Create/Open/Unlock/Lock works on macOS AND Windows; the api-wrapper boundary CI gate is enforced; PRD-LOCKED copy strings present verbatim in CreateVault and UnlockVault and Welcome; vault:locked event triggers store reset + navigation; the frontend has zero string-matching of error messages; the 14-method DTO contract between Go and TS is locked.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Frontend SPA (renderer) ↔ Wails IPC ↔ Go backend | Frontend lock state is advisory; backend `session.RequireUnlocked()` is authoritative. |
| App method body ↔ Wails IPC | Errors transit `apperr.ToDTO`; raw Cause never crosses. |
| `frontend/src/` ↔ `frontend/wailsjs/go/main/App` | Only `frontend/src/api/` is allowed to import generated bindings (CI grep gate enforces). |
| `internal/appconfig/config.json` (last_vault_path) ↔ filesystem | Path leaks the existence + location of an Abyss vault but is not a secret per PRD §9.2. Stored under `os.UserConfigDir()` with 0o600 file permissions. |
| Wails ctx ↔ runtime.EventsEmit/On | `vault:locked` event flows backend → frontend; renderer cannot spoof it. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-03-01 | Spoofing | Frontend pretends `isUnlocked: true` to bypass UI gates | mitigate | Frontend lock state is advisory only (.planning/research/ARCHITECTURE.md §11.4); every secret-touching backend method calls `session.RequireUnlocked()` (acceptance criterion: `grep -c 'RequireUnlocked' app.go` &gt;= 9). PRD §10. |
| T-03-02 | Information Disclosure | Internal Go errors with sensitive context cross the Wails boundary | mitigate | All bound methods end with `return ..., apperr.ToDTO(err)`. ToDTO drops Cause; unknown errors map to generic INTERNAL_ERROR. Acceptance criterion: `grep -c 'apperr.ToDTO' app.go` &gt;= 14. PRD §12.1. |
| T-03-03 | Information Disclosure | Frontend string-matches error messages and leaks raw error text to UI | mitigate | UI-SPEC AppErrorCode → User-Facing String Map locks the 12 user-facing strings; `userFacingMessage(err)` reads `err.code` only (with the single exception of `details.reason` for VALIDATION_ERROR); CI grep gate forbids `.message.includes(`. PRD §18 hard rule #14. |
| T-03-04 | Spoofing | Frontend component imports `wailsjs/go/main/App` directly, bypassing api/ wrapper's normalizeError | mitigate | CI grep gate forbids any import of `wailsjs/go/main/App` outside `frontend/src/api/`. Acceptance criterion enforces. |
| T-03-05 | Tampering | Wails-generated TS bindings drift from Go DTO struct tags (silent contract break) | mitigate | `scripts/check-wails-bindings.sh` runs `wails generate module` in CI and `git diff --quiet` fails on any drift. D-02 plan-3 gate. |
| T-03-06 | Information Disclosure | UnlockVault returns distinguishable errors for wrong-password vs corrupt-vault, enabling oracle | mitigate | `vault.Service.Unlock` collapses ALL failure modes (wrong password, AAD mismatch, GetKeyWrappingByType failure, JSON unmarshal failure) into AppError{CodeUnlockFailed} with identical PRD-LOCKED message. UNSUPPORTED_VAULT and CORRUPT_VAULT come only from `Open` (file-identity check), distinguishable from wrong-password but not disclosing it. Test `TestService_Unlock_AAD_Tamper_FailsGenerically` enforces. PRD §9.4. |
| T-03-07 | Information Disclosure | last_vault_path file readable by malware on user's machine | accept | PRD §4.2 explicitly excludes malware from threat scope. File permissions 0o600 limit casual access. |
| T-03-08 | Information Disclosure | Master password string lingers in renderer process memory before reaching Go | accept | PRD §4.4 documents this Go memory limitation. Renderer process is not under our control; mitigation requires native dialog (out of scope for v0.1). README documents it (Phase 6). |
| T-03-09 | Denial of Service | Concurrent LockVault + UnlockVault race | mitigate | `session.Session` mutex serializes all state transitions; `TestConcurrent_Activate_Lock_RequireUnlocked` (plan 02) covers under -race. |
| T-03-10 | Tampering | A different vault file is swapped at the same path between Open and Unlock | mitigate | `Open` opens and pins the `*sql.DB` handle; `Unlock` operates against that handle. A swap would open an entirely different DB but the existing handle still points at the original file via the OS file descriptor. |
| T-03-11 | Spoofing | Renderer sends arbitrary `path` to UnlockVault to make backend open an attacker-controlled file | mitigate | `storage.VerifyApplicationID` rejects non-Abyss files; `migrations.RunMigrations` rejects dirty schema; both surface distinct error codes BEFORE any KEK derivation. The renderer cannot escalate via path manipulation alone. |
</threat_model>

<verification>
**End-of-plan checks (run from repo root):**

1. `go build ./...` exits 0
2. `go test ./internal/vault/... ./internal/appconfig/... -race -count=1 -timeout 180s` exits 0
3. `golangci-lint run ./...` exits 0
4. `scripts/check-wails-bindings.sh` exits 0
5. `cd frontend &amp;&amp; npm run build` exits 0
6. `grep -rn 'wailsjs/go/main/App' frontend/src --include='*.ts' --include='*.tsx' | grep -v 'frontend/src/api/'` returns no matches
7. `grep -rEn '\.message\.(includes|match|indexOf)\(' frontend/src --include='*.ts' --include='*.tsx'` returns no matches
8. `grep -c '^func (a \*App)' app.go` returns `14`
9. `grep -c 'RequireUnlocked' app.go` returns at least 9
10. `grep -c 'apperr.ToDTO' app.go` returns at least 14
11. `wails build` exits 0
12. **D-05 cross-OS smoke:** macOS + Windows manual lifecycle (Welcome → Create → Vault Home unlocked → Lock → Unlock screen → Re-Unlock → Vault Home), both signed off in `.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md` with commit SHA
13. CI workflow latest run reports `success` on ubuntu-latest + macos-latest + windows-latest including the new `wails generate module` no-drift step and the api-wrapper boundary check
</verification>

<success_criteria>
- 14 Wails methods bound on `App` matching PRD §10 contract; 5 vault-lifecycle methods fully implemented; 9 record/generator/clipboard methods stubbed with INTERNAL_ERROR (filled in plan 04)
- Every secret-touching method invokes `session.RequireUnlocked()` first
- Every error path passes through `apperr.ToDTO()` — verified by grep counts
- `vault.Service.Create` and `Unlock` invoke `aad.Build(KindWrappedDEK, ...)` for both wrap and unwrap — proven by `service_test.go::TestService_Create_FirstSealUsesAADBuild` and round-trip Open/Unlock test
- `vault.Service.Unlock` collapses all failure modes to generic AppError{CodeUnlockFailed} with PRD-LOCKED message — proven by tests
- `internal/appconfig` persists `last_vault_path` outside the encrypted vault using `os.UserConfigDir()` cross-platform (D-11)
- Frontend `frontend/src/api/` is the SOLE entry point for Wails calls; CI grep gate enforces
- Frontend `userFacingMessage()` consumes `code` only; CI grep gate forbids `.message.includes`
- 6 Phase-1 screens render with Mantine v8.3.18 components per UI-SPEC; all PRD-LOCKED copy strings verbatim
- `vault:locked` event subscribed in App.tsx; on emit, vaultStore + clipboardStore reset + router navigates to /unlock
- `wails generate module` no-drift CI gate active; api-wrapper boundary CI gate active
- D-05 Windows smoke for full Create/Unlock/Lock cycle signed off in `01-VERIFICATION.md`
</success_criteria>

<output>
After completion, create `.planning/phases/01-v0-1-dogfoodable-vault/01-03-SUMMARY.md` documenting:
- The 14 Wails-bound methods with one-line summary of each + which are fully implemented vs stubbed for plan 04
- Resolution of D-11 cross-platform config-file location: `os.UserConfigDir()/Abyss/config.json` — confirmed paths are `~/Library/Application Support/Abyss/config.json` (macOS) and `%APPDATA%\Abyss\config.json` (Windows), both verified in `TestConfigPath_ContainsAbyss`
- Confirmed: the FIRST `cipher.Seal` in the production flow (vault.Service.Create → crypto.WrapDEK) uses `aad.Build(KindWrappedDEK, ...)`. Test `TestService_Create_FirstSealUsesAADBuild` enforces.
- Confirmed: PRD-LOCKED user-facing strings are verbatim in `frontend/src/errors/messages.ts` and the CreateVault / UnlockVault / Welcome screens
- Manual smoke result + commit SHAs for macOS AND Windows D-05 sign-off
- Open question forwarded to plan 04: confirm Mantine `<TagsInput>` chip behavior matches the encrypted-tag UX from D-17; confirm `runtime.ClipboardSetText` returns `error` cleanly on macOS Tahoe (Wails v2.12.0 fix per CLAUDE.md §1d)
</output>
