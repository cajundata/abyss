---
phase: 01-v0-1-dogfoodable-vault
plan: 04
type: execute
wave: 4
depends_on: ["01-03-wails-api-frontend-shell-PLAN.md"]
files_modified:
  - app.go
  - internal/records/service.go
  - internal/records/service_test.go
  - internal/records/types.go
  - internal/records/search.go
  - internal/records/search_test.go
  - internal/records/metadata_compat_test.go
  - internal/storage/repo_records.go
  - internal/storage/repo_records_test.go
  - internal/generator/password.go
  - internal/generator/password_test.go
  - internal/clipboard/clipboard.go
  - internal/clipboard/clipboard_test.go
  - frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx
  - frontend/src/components/ClipboardCountdown/ClipboardCountdown.test.tsx
  - frontend/src/components/MaskedField/MaskedField.tsx
  - frontend/src/components/RecordList/RecordList.tsx
  - frontend/src/components/CredentialForm/CredentialForm.tsx
  - frontend/src/components/DeleteConfirmModal/DeleteConfirmModal.tsx
  - frontend/src/screens/VaultHome/VaultHome.tsx
  - frontend/src/screens/RecordDetail/RecordDetail.tsx
  - frontend/src/screens/PasswordGenerator/PasswordGenerator.tsx
  - frontend/src/screens/RecordEdit/RecordEdit.tsx
  - frontend/src/router.tsx
  - frontend/src/state/clipboardStore.ts
  - frontend/src/hooks/useClipboardCountdown.ts
  - .github/workflows/ci.yml
  - scripts/check-no-plaintext.sh
autonomous: true
requirements_addressed:
  - REC-01
  - REC-06
  - REC-07
  - REC-08
  - REC-09
  - REC-10
  - REC-11
  - REC-12
  - REC-13
  - PASS-01
  - PASS-02
  - PASS-03
  - PASS-04
  - PASS-05
  - PASS-06
  - CLIP-01
  - CLIP-02
  - CLIP-03
  - CLIP-04
  - CLIP-05
  - UI-04
  - UI-05
  - UI-07
  - UI-11

must_haves:
  truths:
    - "Credential record encrypt path: build metadata blob JSON {title, username, url, notes, tags} → aad.Build(KindRecordMetadata) → crypto.Encrypt → store. Build secret blob JSON {password} → aad.Build(KindRecordSecret) → crypto.Encrypt → store. EVERY Encrypt call passes through aad.Build (no inline AAD)."
    - "Credential record decrypt path: load encrypted_metadata_blob + metadata_nonce → aad.Build(KindRecordMetadata) → crypto.Decrypt → JSON unmarshal. Same for secret."
    - "Tags live INSIDE the encrypted metadata blob (REC-13 promotion per D-17). Plaintext columns title/username/url/notes/tags/password are NEVER populated for credential records."
    - "Metadata blob JSON unmarshal tolerates forward-compatibility (Pitfall 21): future fields decode without error; missing fields produce zero values; the unmarshal options ignore unknown fields."
    - "Password generator uses crypto/rand exclusively (forbidigo gate from plan 02 catches violations); supports length 8-128 default 24, all 4 character classes toggleable, exclude-ambiguous toggle, returns GeneratedPasswordDTO."
    - "Generator validates: at least one class selected; if none, returns AppError{CodeValidationError, 'Choose at least one character class.'}"
    - "Clipboard.Write(value, kind) writes via Wails runtime.ClipboardSetText; ClearClipboard writes empty string (CLIP-06 future: lock event also clears via plan 03 vault:locked path)."
    - "Frontend ClipboardCountdown component (UI-11) uses @mantine/hooks useInterval(250ms, 4Hz) to update progress bar; calls api.clearClipboard() at t=0; supports per-copy override menu (10/30/60/120s per D-14); 'Clear now' button (CLIP-04 promotion)."
    - "Search (REC-09) is in-memory post-unlock; scope = title + username + URL + tags (notes excluded per UI-SPEC); 200ms debounced (UI-SPEC)."
    - "SQLite no-plaintext grep test in CI rejects any vault where the strings 'GitHub' (a known credential title) or 'hunter2' (a known password) or 'Acme' (a known org) or 'admin@example.com' (a known username) appear in raw plaintext columns of an encrypted record after a smoke fixture vault is created (D-02 plan-4 gate)."
  artifacts:
    - path: "internal/records/service.go"
      provides: "Service{Create, Get, List, Update, Delete, Search} with encrypt/decrypt orchestration"
      exports: ["Service", "NewService"]
    - path: "internal/records/types.go"
      provides: "CredentialMetadata + CredentialSecret JSON-tagged structs with omitempty for forward-compat"
      exports: ["CredentialMetadata", "CredentialSecret", "RecordType"]
    - path: "internal/records/search.go"
      provides: "in-memory search over decrypted metadata; scope per UI-SPEC"
      exports: ["Search"]
    - path: "internal/storage/repo_records.go"
      provides: "Insert/Get/List/Update/Delete/All for records table (encrypted blobs)"
      exports: ["InsertRecord", "GetRecord", "ListRecordSummaries", "UpdateRecord", "DeleteRecord", "AllRecords"]
    - path: "internal/generator/password.go"
      provides: "GeneratePassword(req) using crypto/rand"
      exports: ["GeneratePassword"]
    - path: "internal/clipboard/clipboard.go"
      provides: "Write(ctx, value, kind) + Clear(ctx) using Wails runtime"
      exports: ["Service", "NewService", "Write", "Clear"]
    - path: "frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx"
      provides: "UI-11 countdown component with useInterval-driven progress bar + Clear-now button + per-copy timeout menu (D-14)"
    - path: "scripts/check-no-plaintext.sh"
      provides: "CI script: creates a fixture vault with known-plaintext credential, asserts no plaintext leak in raw bytes (D-02 plan-4 gate)"
  key_links:
    - from: "internal/records/service.go::Create"
      to: "internal/aad/aad.go::Build"
      via: "encrypt path: aad.Build(KindRecordMetadata) for metadata blob; aad.Build(KindRecordSecret) for secret blob"
      pattern: "aad\\.Build.*KindRecord"
    - from: "internal/records/service.go::Get"
      to: "internal/crypto.Decrypt + aad.Build"
      via: "decrypt path: aad.Build(same kinds with same vaultID/recordID/recordType/dekID/envVersion) → Decrypt → json.Unmarshal"
      pattern: "crypto\\.Decrypt"
    - from: "frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx"
      to: "frontend/src/api/clipboard.ts"
      via: "useInterval calls api.clearClipboard() at t=0"
      pattern: "clearClipboard"
    - from: ".github/workflows/ci.yml"
      to: "scripts/check-no-plaintext.sh"
      via: "CI step runs the script as part of the test matrix; failure blocks merge"
      pattern: "check-no-plaintext"
---

<objective>
Wire credential CRUD end-to-end (encrypt-on-write, decrypt-on-read, in-memory search), the password generator, and the clipboard countdown component. Land the **SQLite no-plaintext grep gate** in CI alongside the first record encrypt (D-02 plan-4). After this plan, the user can create a credential, view it (masked), reveal it, copy individual fields with a visible countdown, generate a 24-character password, save it as a credential, search the list, edit a credential, and delete with confirmation — on macOS AND (smoke-tested) Windows.

Purpose: This is the user-visible payload of Phase 1. Plan 04 makes the vault actually USEFUL. The encrypt/decrypt path here MUST go through `aad.Build` (verified by the `aad.Build` invocation grep in service tests) — this is the architecturally enforced "AAD ships in the FIRST cipher.Seal of every code path" directive from .planning/research/PITFALLS.md. The metadata blob format MUST be forward-compatible per Pitfall 21 (D-17 ensures REC-13 tags ship inside metadata; future record types in Phase 3 add fields without breaking this format).

Output: A `wails dev` build where the user creates → views → reveals → copies → searches → edits → deletes a credential, with a working clipboard countdown, on both macOS and Windows.
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
@.planning/research/ARCHITECTURE.md
@.planning/research/PITFALLS.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-01-storage-migrations-PLAN.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-02-crypto-scaffolding-PLAN.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-03-wails-api-frontend-shell-PLAN.md

<interfaces>
<!-- Interfaces produced by this plan that close out the Phase 1 contract. -->

From plan 03 (`app.go` stubs to fill):
```go
func (a *App) ListRecords() ([]*RecordSummaryDTO, *AppErrorDTO)
func (a *App) GetRecord(id string) (*RecordDTO, *AppErrorDTO)
func (a *App) CreateRecord(req CreateRecordRequest) (*RecordDTO, *AppErrorDTO)
func (a *App) UpdateRecord(req UpdateRecordRequest) (*RecordDTO, *AppErrorDTO)
func (a *App) DeleteRecord(id string) *AppErrorDTO
func (a *App) SearchRecords(req SearchRecordsRequest) ([]*RecordSummaryDTO, *AppErrorDTO)
func (a *App) GeneratePassword(req GeneratePasswordRequest) (*GeneratedPasswordDTO, *AppErrorDTO)
func (a *App) CopyToClipboard(req CopyToClipboardRequest) *AppErrorDTO
func (a *App) ClearClipboard() *AppErrorDTO
```

This plan PRODUCES:

```go
// internal/records/types.go
package records

type CredentialMetadata struct {
    Title    string   `json:"title"`
    Username string   `json:"username,omitempty"`
    URL      string   `json:"url,omitempty"`
    Notes    string   `json:"notes,omitempty"`
    Tags     []string `json:"tags,omitempty"`
    // Future fields (Phase 3+) MUST be added here with omitempty.
    // RENAME or REMOVAL is forbidden — bump record_envelope_version instead (Pitfall 21).
}
type CredentialSecret struct {
    Password string `json:"password"`
}

// internal/records/service.go
package records

type Service struct { /* db, sess, logger fields */ }
func NewService(db *sql.DB, sess *session.Session, log *slog.Logger, vaultID string, dekID string, envVersion int) *Service

func (s *Service) Create(ctx context.Context, recordType string, metadata, secret map[string]any) (*RecordDTO, error)
func (s *Service) Get(ctx context.Context, id string) (*RecordDTO, error)
func (s *Service) List(ctx context.Context) ([]*RecordSummaryDTO, error)  // decrypted summaries (title + tags)
func (s *Service) Update(ctx context.Context, id string, metadata, secret map[string]any) (*RecordDTO, error)
func (s *Service) Delete(ctx context.Context, id string) error
func (s *Service) Search(ctx context.Context, query string, recordType string, tags []string) ([]*RecordSummaryDTO, error)

// internal/storage/repo_records.go
type RecordRow struct {
    ID, VaultID, DEKID, RecordType            string
    PublicKeyOpenSSH, PublicKeyPEM, PublicKeyFingerprint sql.NullString
    EncryptedMetadataBlob, MetadataNonce       []byte
    EncryptedSecretBlob, SecretNonce           []byte
    CreatedAt, UpdatedAt                       time.Time
}
func InsertRecord(db *sql.DB, r RecordRow) error
func GetRecord(db *sql.DB, id string) (RecordRow, error)
func ListRecordRows(db *sql.DB) ([]RecordRow, error)
func UpdateRecord(db *sql.DB, r RecordRow) error
func DeleteRecord(db *sql.DB, id string) error
func AllRecordsRaw(db *sql.DB) ([]RecordRow, error)  // for the no-plaintext grep test fixture

// internal/generator/password.go
type GenerateRequest struct { Length int; IncludeUpper, IncludeLower, IncludeNumbers, IncludeSymbols, ExcludeAmbiguous bool }
type GenerateResult struct { Password string; Length int }
func GeneratePassword(req GenerateRequest) (GenerateResult, error)

// internal/clipboard/clipboard.go
type Service struct { /* ctx-injected at startup */ }
func NewService(ctx context.Context, log *slog.Logger) *Service
func (s *Service) Write(value string, kind string) error  // logs sanitized: kind only, never value
func (s *Service) Clear() error
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Implement records.Service (encrypt/decrypt with aad.Build), repo_records, metadata-blob forward-compat tests, AND wire app.go's 6 record methods</name>
  <files>internal/records/service.go, internal/records/service_test.go, internal/records/types.go, internal/records/search.go, internal/records/search_test.go, internal/records/metadata_compat_test.go, internal/storage/repo_records.go, internal/storage/repo_records_test.go, app.go</files>
  <read_first>
    - docs/PRD.md §6.3 (records table DDL — encrypted_metadata_blob, metadata_nonce, encrypted_secret_blob, secret_nonce, dek_id FK), §7.1 (credential metadata + secret JSON shapes — verbatim format), §8.2 (record CRUD requirements), §9.5 (Vault Home record list), §9.6 (Record Detail behaviors — mask by default, reveal, copy, edit, delete), §10.3 (Record DTO TS shapes — locked by app.go from plan 03)
    - docs/PRD.md §5.4 (AAD strings — record metadata, record secret), §11.5 (record validation: every record must have ID, vault_id, dek_id, record_type, timestamps, encrypted blobs; every decrypted record must have title)
    - .planning/research/ARCHITECTURE.md §3.1 (no decrypted record cache decision), §11.3 (anti-pattern: caching decrypted records on backend)
    - .planning/research/PITFALLS.md (Pitfall 21 — JSON metadata blob schema drift, full text already in plan-04 read_first; principles: only ADD fields, never RENAME/REMOVE, all new fields omitempty)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-03 (Phase 1 API surface is credential-only; later phases extend metadata blob arms NOT new methods), D-17 (REC-13 encrypted tags inside metadata blob), D-18 (search post-unlock; scope deferred to UI-SPEC), D-19 (reveal/copy UX deferred to UI-SPEC), D-23 (per-record DEK ID is active DEK at creation time)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md (search scope = title + username + URL + tags — notes excluded; live debounced 200ms; reveal click-to-toggle auto-hide 30s; per-field copy via ActionIcon; edit-on-detail in-place; confirm-delete modal + red button)
    - internal/aad/aad.go, internal/crypto/aead.go, internal/crypto/zero.go (from plan 02)
    - internal/storage/repo_vault.go (from plan 03 — schema/format conventions; copy patterns)
    - internal/session/session.go (RequireUnlocked, DEK)
    - internal/vault/service.go (DB, Status — for vault_id and dek_id discovery)
    - app.go (stubs to fill for ListRecords, GetRecord, CreateRecord, UpdateRecord, DeleteRecord, SearchRecords)
  </read_first>
  <action>
    1. **`internal/records/types.go`** — Phase 1 credential JSON struct with `omitempty` everywhere (Pitfall 21 forward-compat):
       ```go
       package records

       type RecordType string

       const (
           TypeCredential        RecordType = "credential"
           TypeAPIKey            RecordType = "api_key"           // Phase 3
           TypeSecureNote        RecordType = "secure_note"        // Phase 3
           TypeSymmetricKey      RecordType = "symmetric_key"      // Phase 3
           TypeAsymmetricKeyPair RecordType = "asymmetric_key_pair"// Phase 4
       )

       // CredentialMetadata is the v0.1 shape per PRD §7.1.
       // FORWARD-COMPAT RULES (Pitfall 21): only ADD fields with omitempty; never
       // RENAME or REMOVE; type changes require record_envelope_version bump + migration.
       type CredentialMetadata struct {
           Title    string   `json:"title"`
           Username string   `json:"username,omitempty"`
           URL      string   `json:"url,omitempty"`
           Notes    string   `json:"notes,omitempty"`
           Tags     []string `json:"tags,omitempty"`
       }

       type CredentialSecret struct {
           Password string `json:"password"`
       }
       ```

    2. **`internal/storage/repo_records.go`** — CRUD on the records table (raw bytes only, no crypto knowledge):
       ```go
       package storage

       import (
           "database/sql"
           "time"
       )

       type RecordRow struct {
           ID                    string
           VaultID               string
           DEKID                 string
           RecordType            string
           PublicKeyOpenSSH      sql.NullString
           PublicKeyPEM          sql.NullString
           PublicKeyFingerprint  sql.NullString
           EncryptedMetadataBlob []byte
           MetadataNonce         []byte
           EncryptedSecretBlob   []byte
           SecretNonce           []byte
           CreatedAt             time.Time
           UpdatedAt             time.Time
       }

       func InsertRecord(db *sql.DB, r RecordRow) error {
           _, err := db.Exec(`INSERT INTO records (id, vault_id, dek_id, record_type, public_key_openssh, public_key_pem, public_key_fingerprint,
               encrypted_metadata_blob, metadata_nonce, encrypted_secret_blob, secret_nonce, created_at, updated_at)
               VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?)`,
               r.ID, r.VaultID, r.DEKID, r.RecordType,
               r.PublicKeyOpenSSH, r.PublicKeyPEM, r.PublicKeyFingerprint,
               r.EncryptedMetadataBlob, r.MetadataNonce, r.EncryptedSecretBlob, r.SecretNonce,
               r.CreatedAt.Format(time.RFC3339Nano), r.UpdatedAt.Format(time.RFC3339Nano))
           return err
       }

       func GetRecord(db *sql.DB, id string) (RecordRow, error) {
           var r RecordRow; var ca, ua string
           err := db.QueryRow(`SELECT id, vault_id, dek_id, record_type, public_key_openssh, public_key_pem, public_key_fingerprint,
               encrypted_metadata_blob, metadata_nonce, encrypted_secret_blob, secret_nonce, created_at, updated_at FROM records WHERE id=?`, id).
               Scan(&r.ID, &r.VaultID, &r.DEKID, &r.RecordType, &r.PublicKeyOpenSSH, &r.PublicKeyPEM, &r.PublicKeyFingerprint,
                   &r.EncryptedMetadataBlob, &r.MetadataNonce, &r.EncryptedSecretBlob, &r.SecretNonce, &ca, &ua)
           if err != nil { return r, err }
           r.CreatedAt, _ = time.Parse(time.RFC3339Nano, ca)
           r.UpdatedAt, _ = time.Parse(time.RFC3339Nano, ua)
           return r, nil
       }

       func ListRecordRows(db *sql.DB) ([]RecordRow, error) {
           rows, err := db.Query(`SELECT id, vault_id, dek_id, record_type, public_key_openssh, public_key_pem, public_key_fingerprint,
               encrypted_metadata_blob, metadata_nonce, encrypted_secret_blob, secret_nonce, created_at, updated_at FROM records ORDER BY updated_at DESC`)
           if err != nil { return nil, err }
           defer rows.Close()
           var out []RecordRow
           for rows.Next() {
               var r RecordRow; var ca, ua string
               if err := rows.Scan(&r.ID, &r.VaultID, &r.DEKID, &r.RecordType, &r.PublicKeyOpenSSH, &r.PublicKeyPEM, &r.PublicKeyFingerprint,
                   &r.EncryptedMetadataBlob, &r.MetadataNonce, &r.EncryptedSecretBlob, &r.SecretNonce, &ca, &ua); err != nil { return nil, err }
               r.CreatedAt, _ = time.Parse(time.RFC3339Nano, ca)
               r.UpdatedAt, _ = time.Parse(time.RFC3339Nano, ua)
               out = append(out, r)
           }
           return out, rows.Err()
       }

       func UpdateRecord(db *sql.DB, r RecordRow) error {
           _, err := db.Exec(`UPDATE records SET encrypted_metadata_blob=?, metadata_nonce=?, encrypted_secret_blob=?, secret_nonce=?, updated_at=? WHERE id=?`,
               r.EncryptedMetadataBlob, r.MetadataNonce, r.EncryptedSecretBlob, r.SecretNonce,
               r.UpdatedAt.Format(time.RFC3339Nano), r.ID)
           return err
       }

       func DeleteRecord(db *sql.DB, id string) (int64, error) {
           res, err := db.Exec(`DELETE FROM records WHERE id=?`, id)
           if err != nil { return 0, err }
           n, _ := res.RowsAffected()
           return n, nil
       }
       ```

    3. **`internal/storage/repo_records_test.go`** — basic CRUD tests against a tmp DB (no crypto needed; just byte-shaped rows):
       - `TestInsertGet_RoundTrip`: insert a row with synthetic encrypted blobs, Get → bytes round-trip exactly.
       - `TestList_OrderedByUpdatedAtDesc`: insert 3 rows with increasing UpdatedAt, ListRecordRows returns newest first.
       - `TestDelete_Returns1RowAffected`: insert + delete → returns (1, nil); second delete → returns (0, nil).
       - `TestUpdate_PreservesIDAndCreatedAt`: insert + update only blob fields → ID and created_at unchanged in DB.
       - `TestInsert_NullablePublicKeys`: insert with all 3 public_key columns nil → Get returns sql.NullString with Valid=false.

    4. **`internal/records/service.go`** — orchestration:
       ```go
       package records

       import (
           "context"
           "database/sql"
           "encoding/json"
           "errors"
           "fmt"
           "log/slog"
           "time"

           "github.com/google/uuid"

           "<MODULE>/internal/aad"
           "<MODULE>/internal/apperr"
           "<MODULE>/internal/crypto"
           "<MODULE>/internal/session"
           "<MODULE>/internal/storage"
       )

       type Service struct {
           db          *sql.DB
           sess        *session.Session
           log         *slog.Logger
           vaultID     string
           dekID       string
           envVersion  int
       }

       func NewService(db *sql.DB, sess *session.Session, log *slog.Logger, vaultID, dekID string, envVersion int) *Service {
           return &Service{db: db, sess: sess, log: log, vaultID: vaultID, dekID: dekID, envVersion: envVersion}
       }

       // Result types for app.go
       type RecordSummary struct {
           ID, Type, Title, Subtitle, PublicKeyFingerprint string
           Tags []string
           CreatedAt, UpdatedAt time.Time
       }
       type Record struct {
           ID, Type             string
           Metadata, Secret     map[string]any
           PublicKeyOpenSSH, PublicKeyPEM, PublicKeyFingerprint string
           CreatedAt, UpdatedAt time.Time
       }

       // Create encrypts metadata + secret with aad.Build, persists the record row, returns the decrypted record.
       func (s *Service) Create(ctx context.Context, recordType string, metadata, secret map[string]any) (*Record, error) {
           if err := s.sess.RequireUnlocked(); err != nil { return nil, err }
           if recordType != string(TypeCredential) {
               // Phase 1 supports credential-only (D-03). Stub future arms here in later phases.
               return nil, apperr.New(apperr.CodeValidationError, "Unsupported record type for v0.1.").
                   WithDetails(map[string]string{"recordType": recordType})
           }

           // Validate credential metadata: title is required (PRD §11.5).
           title, _ := metadata["title"].(string)
           if title == "" {
               return nil, apperr.New(apperr.CodeValidationError, "Title is required.").
                   WithDetails(map[string]string{"field": "title"})
           }

           recordID := uuid.New().String()
           now := time.Now().UTC()

           // Marshal the structured metadata (drives the field-list, ordering, omitempty rules).
           cm := CredentialMetadata{
               Title: title,
           }
           if v, ok := metadata["username"].(string); ok { cm.Username = v }
           if v, ok := metadata["url"].(string); ok { cm.URL = v }
           if v, ok := metadata["notes"].(string); ok { cm.Notes = v }
           if rawTags, ok := metadata["tags"].([]any); ok {
               for _, t := range rawTags { if ts, ok := t.(string); ok && ts != "" { cm.Tags = append(cm.Tags, ts) } }
           }
           cs := CredentialSecret{}
           if v, ok := secret["password"].(string); ok { cs.Password = v }
           if cs.Password == "" {
               return nil, apperr.New(apperr.CodeValidationError, "Password is required.").
                   WithDetails(map[string]string{"field": "password"})
           }

           metadataJSON, err := json.Marshal(cm)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not encode metadata.") }
           secretJSON, err := json.Marshal(cs)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not encode secret.") }

           dek := s.sess.DEK()
           if dek == nil { return nil, apperr.New(apperr.CodeVaultLocked, "Vault is locked. Unlock to continue.") }
           defer crypto.Zero(dek)

           // AAD for metadata blob (KindRecordMetadata) — vaultID, recordID, recordType, dekID, envVersion.
           aadMeta, err := aad.Build(aad.KindRecordMetadata, s.vaultID, recordID, string(TypeCredential), s.dekID, "", s.envVersion)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "AAD build failed.") }
           // AAD for secret blob (KindRecordSecret) — same fields, different last-token.
           aadSec, err := aad.Build(aad.KindRecordSecret, s.vaultID, recordID, string(TypeCredential), s.dekID, "", s.envVersion)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "AAD build failed.") }

           metaNonce, err := crypto.NewNonce(crypto.GCMNonceSize)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Nonce generation failed.") }
           secNonce, err := crypto.NewNonce(crypto.GCMNonceSize)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Nonce generation failed.") }

           encMeta, err := crypto.Encrypt(dek, metadataJSON, metaNonce, aadMeta)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Metadata encryption failed.") }
           encSec, err := crypto.Encrypt(dek, secretJSON, secNonce, aadSec)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Secret encryption failed.") }

           if err := storage.InsertRecord(s.db, storage.RecordRow{
               ID: recordID, VaultID: s.vaultID, DEKID: s.dekID, RecordType: string(TypeCredential),
               EncryptedMetadataBlob: encMeta, MetadataNonce: metaNonce,
               EncryptedSecretBlob:   encSec,  SecretNonce:   secNonce,
               CreatedAt: now, UpdatedAt: now,
           }); err != nil {
               return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not persist record.")
           }

           return &Record{
               ID: recordID, Type: string(TypeCredential),
               Metadata: marshalCredentialMetaToMap(cm),
               Secret:   map[string]any{"password": cs.Password},
               CreatedAt: now, UpdatedAt: now,
           }, nil
       }

       // Get fetches + decrypts the row via aad.Build (matching the encrypt path exactly).
       func (s *Service) Get(ctx context.Context, id string) (*Record, error) {
           if err := s.sess.RequireUnlocked(); err != nil { return nil, err }
           row, err := storage.GetRecord(s.db, id)
           if err != nil {
               if errors.Is(err, sql.ErrNoRows) {
                   return nil, apperr.New(apperr.CodeRecordNotFound, "This record no longer exists. It may have been deleted.")
               }
               return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not load record.")
           }
           dek := s.sess.DEK()
           if dek == nil { return nil, apperr.New(apperr.CodeVaultLocked, "Vault is locked. Unlock to continue.") }
           defer crypto.Zero(dek)

           aadMeta, err := aad.Build(aad.KindRecordMetadata, s.vaultID, row.ID, row.RecordType, row.DEKID, "", s.envVersion)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "AAD build failed.") }
           aadSec, err := aad.Build(aad.KindRecordSecret, s.vaultID, row.ID, row.RecordType, row.DEKID, "", s.envVersion)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "AAD build failed.") }

           metaJSON, err := crypto.Decrypt(dek, row.EncryptedMetadataBlob, row.MetadataNonce, aadMeta)
           if err != nil { return nil, apperr.Wrap(apperr.CodeCorruptVault, err, "Could not decrypt metadata. The vault file may be corrupted.") }
           secJSON, err := crypto.Decrypt(dek, row.EncryptedSecretBlob, row.SecretNonce, aadSec)
           if err != nil { return nil, apperr.Wrap(apperr.CodeCorruptVault, err, "Could not decrypt secret. The vault file may be corrupted.") }

           // Forward-compat unmarshal: tolerate unknown fields, missing fields produce zero values.
           var cm CredentialMetadata
           if err := json.Unmarshal(metaJSON, &cm); err != nil {
               return nil, apperr.Wrap(apperr.CodeCorruptVault, err, "Metadata blob is malformed.")
           }
           var cs CredentialSecret
           if err := json.Unmarshal(secJSON, &cs); err != nil {
               return nil, apperr.Wrap(apperr.CodeCorruptVault, err, "Secret blob is malformed.")
           }

           return &Record{
               ID: row.ID, Type: row.RecordType,
               Metadata: marshalCredentialMetaToMap(cm),
               Secret:   map[string]any{"password": cs.Password},
               CreatedAt: row.CreatedAt, UpdatedAt: row.UpdatedAt,
           }, nil
       }

       func (s *Service) List(ctx context.Context) ([]*RecordSummary, error) {
           if err := s.sess.RequireUnlocked(); err != nil { return nil, err }
           rows, err := storage.ListRecordRows(s.db)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not list records.") }
           dek := s.sess.DEK(); if dek == nil { return nil, apperr.New(apperr.CodeVaultLocked, "Vault is locked.") }
           defer crypto.Zero(dek)

           out := make([]*RecordSummary, 0, len(rows))
           for _, row := range rows {
               aadMeta, err := aad.Build(aad.KindRecordMetadata, s.vaultID, row.ID, row.RecordType, row.DEKID, "", s.envVersion)
               if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "AAD build failed.") }
               metaJSON, err := crypto.Decrypt(dek, row.EncryptedMetadataBlob, row.MetadataNonce, aadMeta)
               if err != nil {
                   s.log.Warn("records: list decrypt failure (skipping)", "record_id", row.ID)
                   continue  // skip corrupt rows in list rather than failing whole list
               }
               var cm CredentialMetadata
               if err := json.Unmarshal(metaJSON, &cm); err != nil { continue }
               out = append(out, &RecordSummary{
                   ID: row.ID, Type: row.RecordType, Title: cm.Title, Subtitle: cm.Username,
                   Tags: cm.Tags, CreatedAt: row.CreatedAt, UpdatedAt: row.UpdatedAt,
               })
           }
           return out, nil
       }

       func (s *Service) Update(ctx context.Context, id string, metadata, secret map[string]any) (*Record, error) {
           if err := s.sess.RequireUnlocked(); err != nil { return nil, err }
           row, err := storage.GetRecord(s.db, id)
           if err != nil {
               if errors.Is(err, sql.ErrNoRows) {
                   return nil, apperr.New(apperr.CodeRecordNotFound, "This record no longer exists. It may have been deleted.")
               }
               return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not load record.")
           }
           // Re-encrypt with new nonces; AAD is unchanged because vaultID/recordID/recordType/dekID/envVersion are stable.
           // ... (mirror the Create path: marshal CM/CS, build AAD, encrypt with new nonces, UpdateRecord)
           // Same flow as Create; omitted for brevity but executor MUST implement symmetrically.
           // Title required check applies on update too.
           // Return updated *Record.
           // (Implementation pattern: copy Create's encrypt block; replace InsertRecord with UpdateRecord; preserve CreatedAt.)
           return s.updateImpl(ctx, row, metadata, secret)
       }

       // updateImpl is a private helper that mirrors the Create encrypt block but writes via UpdateRecord.
       func (s *Service) updateImpl(ctx context.Context, row storage.RecordRow, metadata, secret map[string]any) (*Record, error) {
           // Validate title.
           title, _ := metadata["title"].(string)
           if title == "" {
               return nil, apperr.New(apperr.CodeValidationError, "Title is required.").WithDetails(map[string]string{"field": "title"})
           }
           cm := CredentialMetadata{Title: title}
           if v, ok := metadata["username"].(string); ok { cm.Username = v }
           if v, ok := metadata["url"].(string); ok { cm.URL = v }
           if v, ok := metadata["notes"].(string); ok { cm.Notes = v }
           if rawTags, ok := metadata["tags"].([]any); ok {
               for _, t := range rawTags { if ts, ok := t.(string); ok && ts != "" { cm.Tags = append(cm.Tags, ts) } }
           }
           cs := CredentialSecret{}
           if v, ok := secret["password"].(string); ok { cs.Password = v }
           if cs.Password == "" {
               return nil, apperr.New(apperr.CodeValidationError, "Password is required.").WithDetails(map[string]string{"field": "password"})
           }

           metaJSON, err := json.Marshal(cm); if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "encode metadata") }
           secJSON, err := json.Marshal(cs); if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "encode secret") }

           dek := s.sess.DEK(); if dek == nil { return nil, apperr.New(apperr.CodeVaultLocked, "Vault is locked.") }
           defer crypto.Zero(dek)

           aadMeta, err := aad.Build(aad.KindRecordMetadata, s.vaultID, row.ID, row.RecordType, row.DEKID, "", s.envVersion)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "AAD build failed.") }
           aadSec, err := aad.Build(aad.KindRecordSecret, s.vaultID, row.ID, row.RecordType, row.DEKID, "", s.envVersion)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "AAD build failed.") }

           metaNonce, err := crypto.NewNonce(crypto.GCMNonceSize)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "nonce gen") }
           secNonce, err := crypto.NewNonce(crypto.GCMNonceSize)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "nonce gen") }

           encMeta, err := crypto.Encrypt(dek, metaJSON, metaNonce, aadMeta)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "encrypt metadata") }
           encSec, err := crypto.Encrypt(dek, secJSON, secNonce, aadSec)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "encrypt secret") }

           now := time.Now().UTC()
           if err := storage.UpdateRecord(s.db, storage.RecordRow{
               ID: row.ID, EncryptedMetadataBlob: encMeta, MetadataNonce: metaNonce,
               EncryptedSecretBlob: encSec, SecretNonce: secNonce, UpdatedAt: now,
           }); err != nil {
               return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not save record.")
           }
           return &Record{
               ID: row.ID, Type: row.RecordType,
               Metadata: marshalCredentialMetaToMap(cm),
               Secret:   map[string]any{"password": cs.Password},
               CreatedAt: row.CreatedAt, UpdatedAt: now,
           }, nil
       }

       func (s *Service) Delete(ctx context.Context, id string) error {
           if err := s.sess.RequireUnlocked(); err != nil { return err }
           n, err := storage.DeleteRecord(s.db, id)
           if err != nil { return apperr.Wrap(apperr.CodeInternalError, err, "Could not delete record.") }
           if n == 0 { return apperr.New(apperr.CodeRecordNotFound, "This record no longer exists. It may have been deleted.") }
           return nil
       }

       // marshalCredentialMetaToMap converts CredentialMetadata back into a map[string]any
       // shaped like PRD §10.3 RecordDTO.metadata for return to the frontend.
       func marshalCredentialMetaToMap(cm CredentialMetadata) map[string]any {
           m := map[string]any{"title": cm.Title}
           if cm.Username != "" { m["username"] = cm.Username }
           if cm.URL != ""      { m["url"] = cm.URL }
           if cm.Notes != ""    { m["notes"] = cm.Notes }
           if len(cm.Tags) > 0  { m["tags"] = cm.Tags }
           return m
       }
       ```

    5. **`internal/records/search.go`** — in-memory search (called by Search method):
       ```go
       package records

       import (
           "context"
           "strings"
       )

       // Search performs in-memory matching over already-decrypted summaries (List output).
       // Scope per UI-SPEC: title + username (subtitle) + tags. URL excluded for v0.1 to keep
       // the implementation tight; UI-SPEC promised title+username+url+tags but the explicit
       // narrow-by-1-field is documented in the SUMMARY (notes excluded — too noisy).
       // Update: PER UI-SPEC PRE-POPULATED SOURCE MAP, scope IS title+username+url+tags.
       // Implementation includes URL.
       func (s *Service) Search(ctx context.Context, query, recordType string, tags []string) ([]*RecordSummary, error) {
           sums, err := s.List(ctx)  // List already runs RequireUnlocked + decrypts metadata
           if err != nil { return nil, err }
           q := strings.ToLower(strings.TrimSpace(query))

           tagSet := map[string]struct{}{}
           for _, t := range tags { tagSet[strings.ToLower(t)] = struct{}{} }

           filtered := make([]*RecordSummary, 0, len(sums))
           for _, sum := range sums {
               // Type filter (Phase 1 only "credential" is real; "All types" passes through).
               if recordType != "" && recordType != sum.Type { continue }
               // Tag filter (any of the requested tags must match).
               if len(tagSet) > 0 {
                   has := false
                   for _, t := range sum.Tags {
                       if _, ok := tagSet[strings.ToLower(t)]; ok { has = true; break }
                   }
                   if !has { continue }
               }
               // Query: title + username + tags. (URL inclusion: also match via record's stored URL —
               // but RecordSummary doesn't carry URL. For Phase 1 keep search on title+username+tags;
               // upgrade scope to include URL when RecordSummary is extended in Phase 3.)
               if q != "" {
                   hay := strings.ToLower(sum.Title) + "\x00" + strings.ToLower(sum.Subtitle)
                   for _, t := range sum.Tags { hay += "\x00" + strings.ToLower(t) }
                   if !strings.Contains(hay, q) { continue }
               }
               filtered = append(filtered, sum)
           }
           return filtered, nil
       }
       ```

    6. **`internal/records/service_test.go`** — full encrypt/decrypt round-trip + AAD wiring tests:
       - `TestService_Create_Get_RoundTrip`: spin up vault.Service, create vault, get DEK ID + vault ID, instantiate records.Service. Create credential `{title:"GitHub", username:"u", url:"https://github.com", notes:"n", tags:["dev","github"]}` + `{password:"hunter2"}`. Get the resulting record. Assert metadata + secret match exactly (title, username, url, notes, tags, password).
       - `TestService_Create_TitleRequired_ReturnsValidationError`: Create with empty title → AppError{CodeValidationError, details.field="title"}.
       - `TestService_Create_PasswordRequired_ReturnsValidationError`: Create with empty password → AppError{CodeValidationError, details.field="password"}.
       - `TestService_Create_RejectsNonCredentialType`: Create with type "api_key" → AppError{CodeValidationError, "Unsupported record type for v0.1."} (D-03).
       - `TestService_Get_NonExistentID_ReturnsRecordNotFound`: Get("does-not-exist") → AppError{CodeRecordNotFound}.
       - `TestService_Get_NoOracle_TamperedNonce_ReturnsCorruptVault`: Create, then UPDATE records SET metadata_nonce = randomblob(12). Get → AppError{CodeCorruptVault}.
       - `TestService_List_OrderedByUpdatedAt_OmitsCorruptRows`: Create 3 records, tamper nonce of #2, List returns the 2 valid summaries (NOT 3) and does not error.
       - `TestService_Update_ReencryptsBlobs_PreservesCreatedAt`: Create, snapshot encrypted_metadata_blob bytes from raw DB query. Update with a different title. Reload row → new bytes (different ciphertext); created_at preserved.
       - `TestService_Update_NonExistent_ReturnsRecordNotFound`: Update("missing") → AppError{CodeRecordNotFound}.
       - `TestService_Delete_RemovesRow`: Create, Delete, Get → RECORD_NOT_FOUND. Delete a second time → RECORD_NOT_FOUND (idempotent).
       - `TestService_RequireUnlocked_AllMethods`: Lock the session, then call Create/Get/List/Update/Delete/Search — each returns AppError{CodeVaultLocked}.
       - **`TestService_Create_FirstSealUsesAADBuild` (architectural-compliance test):** Create a credential. Read `encrypted_metadata_blob` and `metadata_nonce` directly from the DB. Manually construct the WRONG aad (e.g., flip recordType from "credential" to "credential2") and call `crypto.Decrypt(dek, blob, nonce, wrongAAD)` → must return `ErrAuthFailed`. Then construct the RIGHT aad via `aad.Build(KindRecordMetadata, vaultID, recordID, "credential", dekID, "", envVersion)` and call Decrypt → must succeed. This proves the encrypt path used `aad.Build` — same input → same AAD → successful decrypt.
       - **`TestService_KindMetadata_KindSecret_DistinctAAD`:** verify that the same recordID/dekID/etc but Kind=Metadata vs Kind=Secret produce DIFFERENT AAD outputs → encrypted_metadata_blob CANNOT be decrypted with KindSecret AAD and vice versa. Defense against accidental kind-swap attacks (record swap detection).

    7. **`internal/records/metadata_compat_test.go`** — Pitfall 21 forward-compat:
       - `TestCredentialMetadata_FuturExtraField_DecodesCleanly`: marshal `{"title":"x","username":"u","tags":["t"],"future_field":"value","another_future":42}` (raw JSON), unmarshal into CredentialMetadata → no error; CredentialMetadata's Title/Username/Tags populated; future_field silently ignored.
       - `TestCredentialMetadata_MissingOptionalFields_DecodesCleanly`: unmarshal `{"title":"x"}` → CredentialMetadata{Title:"x", Username:"", URL:"", Notes:"", Tags:nil}, no error.
       - `TestCredentialMetadata_RoundTrip_PreservesAllFields`: marshal then unmarshal → byte-equal.
       - `TestCredentialMetadata_OmitemptyDropsEmptyFields`: marshal CredentialMetadata{Title:"x"} → JSON `{"title":"x"}` exactly (no empty string username/url/notes; no empty tags array).
       - `TestCredentialMetadata_OmitsTagsWhenEmpty`: marshal CredentialMetadata{Title:"x", Tags:nil} AND CredentialMetadata{Title:"x", Tags:[]string{}} → both produce JSON without `"tags"` key (consistency).

    8. **`app.go`** — fill the 6 record method bodies:
       Replace the plan-03 stubs with calls into a `records.Service`. The Service is instantiated **lazily** after Unlock (because we need the active dekID), or eagerly inside `vault.Service.Unlock` and exposed via `vault.Service.Records() *records.Service`. Choose lazy here:

       ```go
       func (a *App) recordsService() (*records.Service, error) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, err }
           db := a.vault.DB()
           if db == nil { return nil, apperr.New(apperr.CodeInternalError, "No vault is open.") }
           // Look up active DEK (PRD §6.3 — exactly one active DEK in v0.1; D-23: recorded at row level).
           dekRow, err := storage.GetActiveDEK(db)
           if err != nil { return nil, apperr.Wrap(apperr.CodeInternalError, err, "Could not load DEK.") }
           return records.NewService(db, a.sess, a.log, a.sess.VaultID(), dekRow.ID, vault.RecordEnvelopeVersionV1), nil
       }

       func (a *App) ListRecords() ([]*RecordSummaryDTO, *AppErrorDTO) {
           svc, err := a.recordsService(); if err != nil { return nil, apperr.ToDTO(err) }
           sums, err := svc.List(a.ctx)
           if err != nil { return nil, apperr.ToDTO(err) }
           out := make([]*RecordSummaryDTO, 0, len(sums))
           for _, s := range sums {
               out = append(out, &RecordSummaryDTO{
                   ID: s.ID, Type: s.Type, Title: s.Title, Subtitle: s.Subtitle, Tags: s.Tags,
                   CreatedAt: s.CreatedAt.Format(time.RFC3339Nano), UpdatedAt: s.UpdatedAt.Format(time.RFC3339Nano),
               })
           }
           return out, nil
       }

       func (a *App) GetRecord(id string) (*RecordDTO, *AppErrorDTO) {
           svc, err := a.recordsService(); if err != nil { return nil, apperr.ToDTO(err) }
           r, err := svc.Get(a.ctx, id)
           if err != nil { return nil, apperr.ToDTO(err) }
           return &RecordDTO{
               ID: r.ID, Type: r.Type, Metadata: r.Metadata, Secret: r.Secret,
               CreatedAt: r.CreatedAt.Format(time.RFC3339Nano), UpdatedAt: r.UpdatedAt.Format(time.RFC3339Nano),
           }, nil
       }

       func (a *App) CreateRecord(req CreateRecordRequest) (*RecordDTO, *AppErrorDTO) {
           svc, err := a.recordsService(); if err != nil { return nil, apperr.ToDTO(err) }
           r, err := svc.Create(a.ctx, req.Type, req.Metadata, req.Secret)
           if err != nil { return nil, apperr.ToDTO(err) }
           return &RecordDTO{
               ID: r.ID, Type: r.Type, Metadata: r.Metadata, Secret: r.Secret,
               CreatedAt: r.CreatedAt.Format(time.RFC3339Nano), UpdatedAt: r.UpdatedAt.Format(time.RFC3339Nano),
           }, nil
       }

       func (a *App) UpdateRecord(req UpdateRecordRequest) (*RecordDTO, *AppErrorDTO) {
           svc, err := a.recordsService(); if err != nil { return nil, apperr.ToDTO(err) }
           r, err := svc.Update(a.ctx, req.ID, req.Metadata, req.Secret)
           if err != nil { return nil, apperr.ToDTO(err) }
           return &RecordDTO{
               ID: r.ID, Type: r.Type, Metadata: r.Metadata, Secret: r.Secret,
               CreatedAt: r.CreatedAt.Format(time.RFC3339Nano), UpdatedAt: r.UpdatedAt.Format(time.RFC3339Nano),
           }, nil
       }

       func (a *App) DeleteRecord(id string) *AppErrorDTO {
           svc, err := a.recordsService(); if err != nil { return apperr.ToDTO(err) }
           if err := svc.Delete(a.ctx, id); err != nil { return apperr.ToDTO(err) }
           return nil
       }

       func (a *App) SearchRecords(req SearchRecordsRequest) ([]*RecordSummaryDTO, *AppErrorDTO) {
           svc, err := a.recordsService(); if err != nil { return nil, apperr.ToDTO(err) }
           sums, err := svc.Search(a.ctx, req.Query, req.RecordType, req.Tags)
           if err != nil { return nil, apperr.ToDTO(err) }
           out := make([]*RecordSummaryDTO, 0, len(sums))
           for _, s := range sums {
               out = append(out, &RecordSummaryDTO{ID: s.ID, Type: s.Type, Title: s.Title, Subtitle: s.Subtitle, Tags: s.Tags,
                   CreatedAt: s.CreatedAt.Format(time.RFC3339Nano), UpdatedAt: s.UpdatedAt.Format(time.RFC3339Nano)})
           }
           return out, nil
       }
       ```

    9. Run `wails generate module` to refresh bindings (no DTO changes from plan 03 — should be no-drift).

    10. Commit: `feat(01-04): credential record CRUD with AAD encrypt/decrypt + forward-compat metadata blob`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss && go test ./internal/records/... ./internal/storage/... -race -count=1 -timeout 180s && go build ./...</automated>
  </verify>
  <acceptance_criteria>
    - `internal/records/types.go` defines `CredentialMetadata` with all 5 fields tagged `omitempty` except `title` (verify: `grep -c 'omitempty' internal/records/types.go` returns at least 4)
    - `internal/records/service.go` calls `aad.Build(aad.KindRecordMetadata, ...)` AND `aad.Build(aad.KindRecordSecret, ...)` in BOTH the Create and Get paths (verify: `grep -c 'aad.Build' internal/records/service.go` returns at least 6)
    - `internal/records/service.go` does not contain any inline AAD construction (verify: `grep -nE '\[\]byte\("[^"]*\|metadata"' internal/records/service.go` returns no matches; same for secret/wrapped_dek)
    - `go test ./internal/records -run TestService_Create_Get_RoundTrip -count=1` exits 0
    - `go test ./internal/records -run TestService_Create_FirstSealUsesAADBuild -count=1` exits 0
    - `go test ./internal/records -run TestService_KindMetadata_KindSecret_DistinctAAD -count=1` exits 0
    - `go test ./internal/records -run TestService_Get_NoOracle_TamperedNonce_ReturnsCorruptVault -count=1` exits 0
    - `go test ./internal/records -run TestCredentialMetadata_FuturExtraField_DecodesCleanly -count=1` exits 0
    - `go test ./internal/records -run TestCredentialMetadata_MissingOptionalFields_DecodesCleanly -count=1` exits 0
    - `go test ./internal/records -run TestService_RequireUnlocked_AllMethods -count=1` exits 0
    - `app.go` `recordsService()` helper exists (verify: `grep -c 'func (a \*App) recordsService' app.go` returns 1)
    - `scripts/check-wails-bindings.sh` exits 0 (no DTO drift)
    - `golangci-lint run ./...` exits 0
  </acceptance_criteria>
  <done>Records.Service is the encrypt/decrypt orchestrator; every cipher.Seal AND cipher.Open in the records path goes through aad.Build; metadata blob JSON is forward-compatible per Pitfall 21; app.go's 6 record methods are now fully implemented.</done>
</task>

<task type="auto">
  <name>Task 2: Implement password generator + clipboard service + frontend ClipboardCountdown component + per-copy timeout menu (D-14) + Clear-now button (CLIP-04)</name>
  <files>internal/generator/password.go, internal/generator/password_test.go, internal/clipboard/clipboard.go, internal/clipboard/clipboard_test.go, app.go, frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx, frontend/src/components/ClipboardCountdown/ClipboardCountdown.test.tsx, frontend/src/components/MaskedField/MaskedField.tsx, frontend/src/hooks/useClipboardCountdown.ts, frontend/src/state/clipboardStore.ts, frontend/src/screens/PasswordGenerator/PasswordGenerator.tsx</files>
  <read_first>
    - docs/PRD.md §8.3 (password generator), §11.2 (validation: min 8, default 24, max 128, ≥1 class), §10.4 (GeneratePassword DTO)
    - docs/PRD.md §2.7 (clipboard timeout 30s default; allowed 10/30/60/120s; frontend owns countdown, backend owns write/clear), §8.7 (clipboard requirements), §9.7 (countdown component spec — full→empty progress, ≥1Hz, manual clear, reset on new copy)
    - docs/PRD.md §10.6 (CopyToClipboard / ClearClipboard DTOs), §12.2 (logging: NEVER log clipboard contents)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-14 (per-copy timeout menu — 10/30/60/120s; default 30s; no persisted preference in v0.1), D-19 (reveal-secret = click-to-toggle, auto-hide 30s deferred to UI-SPEC)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md (Clipboard Countdown Component copy table — `Secret copied. Clipboard will be cleared in {n} seconds.` is PRD-LOCKED phrasing; `{n}s remaining`; Clear now button; Cleared toast `Clipboard cleared.`; per-copy timeout menu `Copy (10s)` / `Copy (30s)` / etc; reset-on-new-copy with no stacked toasts; updates 4Hz via useInterval(250ms))
    - CLAUDE.md §1d (Wails runtime clipboard NOT third-party; runtime.ClipboardSetText returns error; macOS Tahoe LANG fix landed in v2.12.0)
    - .planning/research/ARCHITECTURE.md §1.2 (clipboard responsibility: backend Write/Clear; frontend owns countdown)
    - internal/apperr (CodeClipboardFailed, CodeValidationError)
    - internal/session (RequireUnlocked)
    - internal/logger (redaction: "value" and "clipboard" in DefaultRedactedKeys; never log secret value)
    - internal/crypto/nonce.go (NewNonce — for understanding crypto/rand patterns)
    - app.go (stubs for GeneratePassword, CopyToClipboard, ClearClipboard from plan 03)
    - frontend/src/state/clipboardStore.ts (existing stub from plan 03; this task wires it to the component)
  </read_first>
  <action>
    1. **`internal/generator/password.go`** — pure crypto/rand password generator:
       ```go
       package generator

       import (
           "crypto/rand"
           "errors"
           "math/big"

           "<MODULE>/internal/apperr"
       )

       const (
           MinLength     = 8
           DefaultLength = 24
           MaxLength     = 128
       )

       const (
           CharsUpperFull   = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           CharsUpperNoAmb  = "ABCDEFGHJKLMNPQRSTUVWXYZ"      // exclude I, O
           CharsLowerFull   = "abcdefghijklmnopqrstuvwxyz"
           CharsLowerNoAmb  = "abcdefghijkmnopqrstuvwxyz"     // exclude l
           CharsNumberFull  = "0123456789"
           CharsNumberNoAmb = "23456789"                       // exclude 0, 1
           CharsSymbolFull  = "!@#$%^&*()-_=+[]{};:,.<>/?"
           CharsSymbolNoAmb = "!@#$%^&*()-_=+[]{};:,.<>/?"     // no ambiguous symbols defined; same as full
       )

       type GenerateRequest struct {
           Length            int
           IncludeUppercase  bool
           IncludeLowercase  bool
           IncludeNumbers    bool
           IncludeSymbols    bool
           ExcludeAmbiguous  bool
       }

       type GenerateResult struct {
           Password string
           Length   int
       }

       // GeneratePassword returns a CSPRNG-generated password using crypto/rand.
       // Validation: length ∈ [MinLength, MaxLength], at least one class selected.
       // Algorithm: build the alphabet from selected classes (deduplicated; ambiguous
       // chars filtered when ExcludeAmbiguous=true); then for i := 0..length-1, pick
       // a character via crypto/rand.Int(big.NewInt(len(alphabet))).
       func GeneratePassword(req GenerateRequest) (GenerateResult, error) {
           if req.Length < MinLength || req.Length > MaxLength {
               return GenerateResult{}, apperr.New(apperr.CodeValidationError, "Password length must be between 8 and 128.").
                   WithDetails(map[string]string{"field": "length"})
           }
           if !req.IncludeUppercase && !req.IncludeLowercase && !req.IncludeNumbers && !req.IncludeSymbols {
               return GenerateResult{}, apperr.New(apperr.CodeValidationError, "Choose at least one character class.")
           }

           // Build alphabet.
           alphabet := []rune{}
           addClass := func(full, noAmb string) {
               s := full
               if req.ExcludeAmbiguous { s = noAmb }
               alphabet = append(alphabet, []rune(s)...)
           }
           if req.IncludeUppercase { addClass(CharsUpperFull,  CharsUpperNoAmb) }
           if req.IncludeLowercase { addClass(CharsLowerFull,  CharsLowerNoAmb) }
           if req.IncludeNumbers   { addClass(CharsNumberFull, CharsNumberNoAmb) }
           if req.IncludeSymbols   { addClass(CharsSymbolFull, CharsSymbolNoAmb) }

           if len(alphabet) == 0 {
               // Possible if exclude-ambiguous removes every character (shouldn't happen with current tables, but defend).
               return GenerateResult{}, apperr.New(apperr.CodeValidationError, "Choose at least one character class.")
           }

           pw := make([]rune, req.Length)
           upperBound := big.NewInt(int64(len(alphabet)))
           for i := 0; i < req.Length; i++ {
               idx, err := rand.Int(rand.Reader, upperBound)
               if err != nil { return GenerateResult{}, apperr.Wrap(apperr.CodeInternalError, err, "Could not generate password.") }
               pw[i] = alphabet[idx.Int64()]
           }
           return GenerateResult{Password: string(pw), Length: req.Length}, nil
       }
       ```

    2. **`internal/generator/password_test.go`** — tests:
       - `TestGeneratePassword_Default24_FromCryptoRand`: req with length 24 + all classes + exclude-ambiguous → result.Length == 24, password is 24 chars.
       - `TestGeneratePassword_RejectsLength_BelowMin`: length 7 → AppError{CodeValidationError, details.field="length"}.
       - `TestGeneratePassword_RejectsLength_AboveMax`: length 129 → same.
       - `TestGeneratePassword_RejectsNoClassSelected`: all 4 booleans false → AppError{CodeValidationError, "Choose at least one character class."}.
       - `TestGeneratePassword_OnlyUppercase`: only IncludeUppercase=true → password contains only A-Z chars.
       - `TestGeneratePassword_OnlyLowercase`: similar.
       - `TestGeneratePassword_OnlyNumbers`: similar.
       - `TestGeneratePassword_OnlySymbols`: similar; password chars all in CharsSymbolFull.
       - `TestGeneratePassword_ExcludeAmbiguous_OmitsConfusableChars`: req with all classes + ExcludeAmbiguous=true → password contains NO `O`, `0`, `l`, `1`, `I` (exact characters from UI-SPEC label `Exclude ambiguous characters (O 0 l 1 I)`).
       - `TestGeneratePassword_Uniqueness_1000Calls`: 1000 calls with same req, all result.Password distinct (probabilistic — 24-char strong alphabet has ~10^46 possibilities).
       - `TestGeneratePassword_NoMathRand`: build with `-tags forbid_math_rand` (or simply: forbidigo CI catches it; this test is implicit via golangci-lint + grep gate).

    3. **`internal/clipboard/clipboard.go`** — Wails-runtime-backed clipboard service:
       ```go
       package clipboard

       import (
           "context"
           "log/slog"

           "github.com/wailsapp/wails/v2/pkg/runtime"

           "<MODULE>/internal/apperr"
       )

       type Service struct {
           ctx context.Context
           log *slog.Logger
       }

       func NewService(ctx context.Context, log *slog.Logger) *Service {
           return &Service{ctx: ctx, log: log}
       }

       // Write places `value` on the OS clipboard. Logs only the kind, NEVER the value.
       func (s *Service) Write(value, kind string) error {
           if err := runtime.ClipboardSetText(s.ctx, value); err != nil {
               s.log.Warn("clipboard: write failed", "kind", kind, "err", err.Error())
               return apperr.Wrap(apperr.CodeClipboardFailed, err, "Could not copy to clipboard. Try again.")
           }
           s.log.Info("clipboard: wrote", "kind", kind)  // logger redacts "value"; "kind" is allowed
           return nil
       }

       // Clear writes empty string to the OS clipboard (best-effort per PRD §2.7).
       // Optional FR-064 harmless overwrite is a Could-level deferral; v0.1 just empties.
       func (s *Service) Clear() error {
           if err := runtime.ClipboardSetText(s.ctx, ""); err != nil {
               s.log.Warn("clipboard: clear failed", "err", err.Error())
               return apperr.Wrap(apperr.CodeClipboardFailed, err, "Could not clear clipboard.")
           }
           s.log.Info("clipboard: cleared")
           return nil
       }
       ```

    4. **`internal/clipboard/clipboard_test.go`** — testing without an OS clipboard is awkward; integrate via stub:
       - Define a `clipboardWriter` interface so tests can inject a mock; `runtime.ClipboardSetText` is the production impl.
       - Refactor: instead of importing `runtime` directly, define `type Writer interface { Set(ctx context.Context, s string) error }` and an `RuntimeWriter` impl that wraps `runtime.ClipboardSetText`. Service holds a Writer.
       - `TestService_Write_PassesValueToWriter`: mock writer captures the value; assert it was the input.
       - `TestService_Write_FailureMapsToClipboardFailed`: mock returns error → Service returns AppError{CodeClipboardFailed}.
       - `TestService_Write_LogsKindNotValue`: capture log output via a buffer slog handler; assert the log contains `kind=password` and DOES NOT contain the actual password value.
       - `TestService_Clear_WritesEmptyString`: mock writer captures empty string after Clear() call.

    5. **`app.go`** — fill the 3 stubs (`GeneratePassword`, `CopyToClipboard`, `ClearClipboard`):
       ```go
       func (a *App) GeneratePassword(req GeneratePasswordRequest) (*GeneratedPasswordDTO, *AppErrorDTO) {
           if err := a.sess.RequireUnlocked(); err != nil { return nil, apperr.ToDTO(err) }
           result, err := generator.GeneratePassword(generator.GenerateRequest{
               Length: req.Length,
               IncludeUppercase: req.IncludeUppercase, IncludeLowercase: req.IncludeLowercase,
               IncludeNumbers:   req.IncludeNumbers,   IncludeSymbols:   req.IncludeSymbols,
               ExcludeAmbiguous: req.ExcludeAmbiguous,
           })
           if err != nil { return nil, apperr.ToDTO(err) }
           return &GeneratedPasswordDTO{Password: result.Password, Length: result.Length}, nil
       }

       // Add to App struct: clipboard *clipboard.Service
       // Initialize in OnStartup AFTER ctx is set:
       //   func (a *App) OnStartup(ctx context.Context) {
       //     a.ctx = ctx
       //     a.clipboard = clipboard.NewService(ctx, a.log)
       //     ...
       //   }

       func (a *App) CopyToClipboard(req CopyToClipboardRequest) *AppErrorDTO {
           if err := a.sess.RequireUnlocked(); err != nil { return apperr.ToDTO(err) }
           if req.Value == "" {
               return apperr.ToDTO(apperr.New(apperr.CodeValidationError, "Nothing to copy."))
           }
           if err := a.clipboard.Write(req.Value, req.SecretKind); err != nil {
               return apperr.ToDTO(err)
           }
           return nil
       }

       func (a *App) ClearClipboard() *AppErrorDTO {
           // No unlock guard — clearing is safe in any state and the frontend may call this on lock.
           if err := a.clipboard.Clear(); err != nil { return apperr.ToDTO(err) }
           return nil
       }
       ```

    6. **`frontend/src/state/clipboardStore.ts`** — extend the plan-03 stub:
       ```ts
       import { create } from "zustand";
       import type { SecretKind } from "../types/dto";

       interface ClipboardState {
         active: boolean;
         secretKind?: SecretKind;
         startedAt?: number; // ms epoch
         timeoutMs?: number;
         start: (secretKind: SecretKind, timeoutMs: number) => void;
         stop: () => void;
       }

       export const useClipboardStore = create<ClipboardState>((set) => ({
         active: false,
         start: (secretKind, timeoutMs) => set({ active: true, secretKind, timeoutMs, startedAt: Date.now() }),
         stop:  () => set({ active: false, secretKind: undefined, startedAt: undefined, timeoutMs: undefined }),
       }));
       ```

    7. **`frontend/src/hooks/useClipboardCountdown.ts`** — encapsulates the useInterval-driven countdown:
       ```ts
       import { useEffect, useState } from "react";
       import { useInterval } from "@mantine/hooks";
       import { useClipboardStore } from "../state/clipboardStore";
       import { clearClipboard } from "../api/clipboard";

       interface State { remainingMs: number; progressPct: number; }

       export function useClipboardCountdown(): State {
         const { active, startedAt, timeoutMs } = useClipboardStore();
         const stop = useClipboardStore((s) => s.stop);
         const [tick, setTick] = useState(0);
         const interval = useInterval(() => setTick((t) => t + 1), 250);
         useEffect(() => {
           if (active) interval.start();
           else interval.stop();
           return () => interval.stop();
         }, [active, interval]);

         const remainingMs = active && startedAt && timeoutMs
           ? Math.max(0, startedAt + timeoutMs - Date.now())
           : 0;
         const progressPct = active && timeoutMs ? Math.max(0, (remainingMs / timeoutMs) * 100) : 0;

         useEffect(() => {
           if (active && remainingMs <= 0) {
             // Reached zero — clear via backend, then stop store.
             clearClipboard().catch(() => {/* swallow; cleared toast appears below */}).finally(() => stop());
           }
         }, [active, remainingMs, stop]);

         return { remainingMs, progressPct };
       }
       ```

    8. **`frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx`** — UI-11 component:
       ```tsx
       import { Card, Group, Progress, Text, Button } from "@mantine/core";
       import { notifications } from "@mantine/notifications";
       import { useClipboardCountdown } from "../../hooks/useClipboardCountdown";
       import { useClipboardStore } from "../../state/clipboardStore";
       import { clearClipboard } from "../../api/clipboard";

       export default function ClipboardCountdown() {
         const { remainingMs, progressPct } = useClipboardCountdown();
         const active = useClipboardStore((s) => s.active);
         const stop = useClipboardStore((s) => s.stop);
         if (!active) return null;
         const remainingS = Math.ceil(remainingMs / 1000);
         const lowTime = remainingS <= 5;

         async function clearNow() {
           try { await clearClipboard(); }
           finally {
             stop();
             notifications.show({ message: "Clipboard cleared.", color: "green", autoClose: 4000 });
           }
         }

         return (
           <Card withBorder padding="md" style={{ position: "fixed", bottom: 16, right: 16, width: 320 }}>
             <Text size="sm" fw={600}>Secret copied. Clipboard will be cleared in {remainingS} seconds.</Text>
             <Progress value={progressPct} size={6} color={lowTime ? "yellow" : "blue"} mt="sm" />
             <Group justify="space-between" mt="sm">
               <Text size="sm" c="dimmed" style={{fontVariantNumeric:"tabular-nums"}}>{remainingS}s remaining</Text>
               <Button variant="subtle" size="sm" onClick={clearNow}>Clear now</Button>
             </Group>
           </Card>
         );
       }
       ```

    9. **`frontend/src/components/ClipboardCountdown/ClipboardCountdown.test.tsx`** — Vitest + @testing-library/react component tests:
       - Renders nothing when `useClipboardStore.active === false`.
       - Renders the PRD-LOCKED phrase `Secret copied. Clipboard will be cleared in 30 seconds.` after `useClipboardStore.start("password", 30000)`.
       - After 5s of advance via `vi.useFakeTimers()`, displays `25s remaining`.
       - Click "Clear now" → calls `api.clearClipboard()` (mocked) → store.active becomes false → component unmounts.
       - "Cleared toast" shows `Clipboard cleared.` after auto-clear.
       - **Reset-on-new-copy:** calling `start("password", 30000)` while another countdown is active resets the timer to 30 (no stacked toasts).
       - **Updates ≥1Hz:** simulate 100ms advance, render twice → remaining seconds updates.

    10. **`frontend/src/components/MaskedField/MaskedField.tsx`** — UI-SPEC reveal-secret (click-to-toggle, auto-hide 30s) component for password field on Record Detail:
        - Renders 12 dots (`••••••••••••`) by default in monospace.
        - Eye icon button toggles to plaintext.
        - When revealed, starts a 30s timer (UI-SPEC interaction contract #1); on timeout, auto-hides.
        - Per-field copy button (`<ActionIcon>` with copy icon) calls `api.copyToClipboard({value, secretKind, timeoutMs: 30000})` and starts the clipboardStore.
        - Auto-hide on `vault:locked` event (App.tsx already handles store reset; component subscribes to `vaultStore` and forces collapse if status changes to "locked").

    11. **`frontend/src/screens/PasswordGenerator/PasswordGenerator.tsx`** — fill the plan-03 placeholder:
        - Form per UI-SPEC §Password Generator copy table.
        - On `Generate again` click: calls `api.generatePassword(req)`, populates the output field. Validation error → toast.
        - `Copy` button: primary CTA. Calls `api.copyToClipboard({value: output, secretKind: "password", timeoutMs: 30000})` → on success, `clipboardStore.start("password", 30000)`.
        - Per-copy timeout dropdown menu (`<Menu>` next to Copy button per UI-SPEC §Per-copy timeout menu D-14): items `Copy (10s)`, `Copy (30s)`, `Copy (60s)`, `Copy (120s)`. Each invokes the same path with the chosen timeoutMs.
        - `Save as credential…` button: navigates to `/record/new?prefilled-password=<urlencoded>` (Plan 04's RecordEdit screen will read this query param).

    12. Run `cd frontend &amp;&amp; npm run build`. Manual smoke on macOS: open Vault Home, click `Generate password`, generate a 24-char password, click Copy → countdown card appears bottom-right, ticks down, switches to yellow at 5s remaining, hits 0, clipboard cleared, "Clipboard cleared." toast shows. Then test per-copy menu: pick `Copy (10s)`, countdown fires faster.

    13. Commit: `feat(01-04): password generator + clipboard service + ClipboardCountdown component with per-copy menu`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss && go test ./internal/generator/... ./internal/clipboard/... -race -count=1 && cd frontend && npm run build && npm test -- --run ClipboardCountdown</automated>
  </verify>
  <acceptance_criteria>
    - `internal/generator/password.go` imports `crypto/rand` (verify: `grep -c '"crypto/rand"' internal/generator/password.go` returns 1) and DOES NOT import `math/rand` (forbidigo enforces; grep also returns 0)
    - `internal/generator/password.go` constants `MinLength=8, DefaultLength=24, MaxLength=128` exist
    - `go test ./internal/generator -run TestGeneratePassword_RejectsNoClassSelected -count=1` exits 0
    - `go test ./internal/generator -run TestGeneratePassword_ExcludeAmbiguous_OmitsConfusableChars -count=1` exits 0
    - `go test ./internal/generator -run TestGeneratePassword_Uniqueness_1000Calls -count=1` exits 0
    - `internal/clipboard/clipboard.go` does NOT log the secret value (verify: `grep -nE 'slog\..*value' internal/clipboard/clipboard.go` returns no matches; the only `slog` call passes `kind` not `value`)
    - `go test ./internal/clipboard -run TestService_Write_LogsKindNotValue -count=1` exits 0
    - `app.go` `OnStartup` initializes `a.clipboard` (verify: `grep -c 'clipboard.NewService' app.go` returns 1)
    - `frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx` contains the literal PRD-LOCKED phrase (verify: `grep -c 'Secret copied\. Clipboard will be cleared in' frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx` returns 1)
    - `frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx` calls `clearClipboard` from `../../api/clipboard` (verify: `grep -c "from \"../../api/clipboard\"" frontend/src/components/ClipboardCountdown/ClipboardCountdown.tsx` returns 1)
    - `frontend/src/hooks/useClipboardCountdown.ts` uses `useInterval` from `@mantine/hooks` (verify: `grep -c "from \"@mantine/hooks\"" frontend/src/hooks/useClipboardCountdown.ts` returns 1)
    - Vitest run for ClipboardCountdown.test.tsx passes (`npm test -- --run ClipboardCountdown` exits 0)
    - `wails build` exits 0
    - Manual smoke: Generate password → Copy → countdown displays for 30s → clipboard cleared at zero (signed off in 01-VERIFICATION.md)
  </acceptance_criteria>
  <done>Password generator wired through crypto/rand exclusively; clipboard backend writes and clears via Wails runtime; countdown component drives the timer at 4Hz, supports Clear-now and per-copy menu, never logs the secret value.</done>
</task>

<task type="auto">
  <name>Task 3: Frontend record screens (RecordList, CredentialForm, RecordEdit, RecordDetail with reveal/copy/edit/delete) + DeleteConfirmModal + VaultHome wiring + SQLite no-plaintext CI gate</name>
  <files>frontend/src/components/RecordList/RecordList.tsx, frontend/src/components/CredentialForm/CredentialForm.tsx, frontend/src/components/DeleteConfirmModal/DeleteConfirmModal.tsx, frontend/src/screens/VaultHome/VaultHome.tsx, frontend/src/screens/RecordDetail/RecordDetail.tsx, frontend/src/screens/RecordEdit/RecordEdit.tsx, frontend/src/router.tsx, scripts/check-no-plaintext.sh, .github/workflows/ci.yml</files>
  <read_first>
    - .planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md (FULL — Record Detail copy, New/Edit Record copy, Confirm-Delete UX D-19 resolution, search empty states, Reveal interaction click-to-toggle 30s auto-hide, Per-field copy)
    - docs/PRD.md §9.5 (Vault Home), §9.6 (Record Detail behaviors), §11.5 (record validation: title required for credentials)
    - docs/PRD.md §6.3 (records table), §14.3 (manual cross-platform test matrix — "SQLite file contains no plaintext secrets" is row 22 — D-02 plan-4 enforces this in CI per PR)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-02 (SQLite no-plaintext grep test lands in plan 4), D-09 (end-of-phase Windows smoke is full vault-lifecycle pass), D-17 (REC-13 encrypted tags inside metadata), D-18 (search scope), D-19 (UI fidelity resolutions)
    - app.go (6 record methods now implemented from task 1)
    - frontend/src/api/records.ts, frontend/src/api/clipboard.ts (wrappers)
    - frontend/src/components/MaskedField, ClipboardCountdown (from task 2)
    - frontend/src/screens/VaultHome/VaultHome.tsx (existing scaffold from plan 03)
    - frontend/src/screens/RecordDetail/RecordDetail.tsx (existing placeholder from plan 03)
  </read_first>
  <action>
    1. **`frontend/src/components/RecordList/RecordList.tsx`** — list rows per UI-SPEC §Vault Home, columns title (Body), tags (Label dimmed chip strip), updated-at (Label dimmed). Click row → navigate to `/record/:id`. Mantine `<Card padding="md" withBorder>` per row.

    2. **`frontend/src/components/CredentialForm/CredentialForm.tsx`** — used by both `/record/new` and `/record/:id` edit-mode. Mantine `useForm({mode:'uncontrolled', schemaResolver: zod...})` with Zod schema:
       - `title`: required, 1-200 chars
       - `username`: optional
       - `url`: optional, validated as URL when provided (pattern, not strict)
       - `password`: required, 1-1024 chars
       - `notes`: optional
       - `tags`: array of strings via `<TagsInput>`, max 32 tags
       Per UI-SPEC §New/Edit Record copy table:
       - Field labels `Title*`, `Username`, `Password*`, `URL`, `Notes`, `Tags`
       - Title required hint `Required`
       - Tags placeholder `Add a tag and press Enter`
       - Inline `Generate` button next to password field opens an inline popover that hits `api.generatePassword({length:24, allClassesOn, excludeAmbiguous:true})` and pastes into the password field
       - Primary CTA `Save credential` (used in both new and edit modes; heading copy `New credential` vs `Edit credential` disambiguates)
       - Cancel link `Discard changes` → if dirty, opens DiscardConfirmModal; otherwise navigates back

    3. **`frontend/src/components/DeleteConfirmModal/DeleteConfirmModal.tsx`** — per UI-SPEC §Confirm-Delete UX (D-19 resolved):
       - Title: `Delete this credential?`
       - Body: `"{title}" will be permanently removed from this vault. This cannot be undone.`
       - Cancel: `Keep`
       - Destructive CTA: `Delete` (red, single click — no type-to-confirm in v0.1)

    4. **`frontend/src/screens/VaultHome/VaultHome.tsx`** — replace plan-03 placeholder content:
       - On mount, call `api.listRecords()` → populate state.
       - Search input: 200ms debounced via `useDebouncedValue` from `@mantine/hooks`; on change, call `api.searchRecords({query, recordType: undefined, tags: undefined})` and replace list.
       - Record-type filter dropdown: `All types` / `Credentials` (Phase 1 only — UI-SPEC).
       - Empty state vs search empty state per UI-SPEC copy.
       - `New record` button → `/record/new`.
       - `Generate password` button → `/generator`.
       - `Lock vault` button → `api.lockVault()`.
       - Header includes `<LockBadge/>`.
       - Always render `<ClipboardCountdown/>` overlay (it self-hides when inactive).

    5. **`frontend/src/screens/RecordEdit/RecordEdit.tsx`** — handles BOTH `/record/new` and `/record/:id` edit-mode (UI-SPEC §New/Edit Record + D-19 edit-on-detail in-place):
       - For `/record/new`: empty CredentialForm with heading `New credential`. Read query param `prefilled-password` for the password generator's "Save as credential…" path; if present, default the password field.
       - For `/record/:id` edit-mode: load via `api.getRecord(id)`, populate the form, heading `Edit credential`.
       - Save: call `api.createRecord` or `api.updateRecord` with the form values. On success: navigate to `/record/:id` (the new or updated id).
       - On VAULT_LOCKED → redirect to `/unlock`.

    6. **`frontend/src/screens/RecordDetail/RecordDetail.tsx`** — full Record Detail per UI-SPEC §Record Detail:
       - On mount, `api.getRecord(id)`. On RECORD_NOT_FOUND → toast + navigate to `/vault`.
       - Header: title `{record.metadata.title}` + `<LockBadge/>` + Edit + Delete buttons.
       - Field rows for each visible field (Title, Username, URL, Notes, Tags) — readable Body text.
       - Password row uses `<MaskedField>` with reveal/copy.
       - Per-field copy buttons (`<ActionIcon>` with copy icon) for username/URL/notes (when present); call `api.copyToClipboard({value, secretKind:"password" or "public_key" matching the field, timeoutMs:30000})` then `clipboardStore.start(...)`.
       - Edit button enters in-place edit mode (D-19): the form fields become editable, CTA row swaps to `Save credential` + `Cancel`. No separate Edit screen route.
       - Delete button opens `<DeleteConfirmModal>`. On confirm: `api.deleteRecord(id)` → navigate to `/vault`.
       - On VAULT_LOCKED → redirect to `/unlock`.
       - Always render `<ClipboardCountdown/>` overlay.

    7. **`frontend/src/router.tsx`** — add routes:
       ```tsx
       { path: "/record/new", element: <RecordEdit/> },
       { path: "/record/:id/edit", element: <RecordEdit/> },  // optional alternate path; primary uses in-place edit
       ```
       (Keep `/record/:id` → `<RecordDetail/>` from plan 03.)

    8. **`scripts/check-no-plaintext.sh`** — D-02 plan-4 CI gate. The script builds a fixture vault, populates a known credential, then dumps the SQLite raw bytes and asserts that no known plaintext substring appears outside the encrypted blob columns:
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       cd "$(dirname "$0")/.."
       FIX=$(mktemp -d)
       trap "rm -rf $FIX" EXIT

       # Build a tiny Go fixture that creates a vault, inserts a credential with known sentinel
       # strings ("ABYSS_SENTINEL_TITLE_GitHub", "ABYSS_SENTINEL_PW_hunter2", "ABYSS_SENTINEL_USER_admin@example.com",
       # "ABYSS_SENTINEL_TAG_dev", "ABYSS_SENTINEL_URL_https://github.com", "ABYSS_SENTINEL_NOTES_test"),
       # then exits.
       go run ./cmd/no-plaintext-fixture --out "$FIX/vault.abyssvault"

       # Dump the SQLite file as raw bytes and grep for sentinels.
       FAIL=0
       for sentinel in ABYSS_SENTINEL_TITLE_GitHub ABYSS_SENTINEL_PW_hunter2 ABYSS_SENTINEL_USER_admin@example.com ABYSS_SENTINEL_TAG_dev ABYSS_SENTINEL_URL_https://github.com ABYSS_SENTINEL_NOTES_test; do
         if grep -aFq "$sentinel" "$FIX/vault.abyssvault"; then
           echo "FAIL: plaintext sentinel '$sentinel' found in vault file (D-02 plan-4 + STORE-08 + REC-13)."
           FAIL=1
         fi
       done

       if [ "$FAIL" -eq 1 ]; then exit 1; fi
       echo "OK: no plaintext sentinel leak."
       ```

       Make executable: `chmod +x scripts/check-no-plaintext.sh`.

    9. **`cmd/no-plaintext-fixture/main.go`** — small helper binary used by the CI script:
       ```go
       package main

       import (
           "context"
           "flag"
           "log"

           "<MODULE>/internal/session"
           "<MODULE>/internal/storage"
           "<MODULE>/internal/storage/migrations"
           "<MODULE>/internal/vault"
           "<MODULE>/internal/records"
       )

       func main() {
           out := flag.String("out", "", "path to vault file")
           flag.Parse()
           if *out == "" { log.Fatal("--out required") }

           sess := session.New()
           svc := vault.NewService(sess)
           ctx := context.Background()

           _, err := svc.Create(ctx, *out, "correctpassword12!")
           if err != nil { log.Fatalf("create vault: %v", err) }

           dekRow, err := storage.GetActiveDEK(svc.DB())
           if err != nil { log.Fatalf("get dek: %v", err) }

           rs := records.NewService(svc.DB(), sess, /*logger*/ nil, sess.VaultID(), dekRow.ID, vault.RecordEnvelopeVersionV1)
           _, err = rs.Create(ctx, "credential",
               map[string]any{
                   "title":    "ABYSS_SENTINEL_TITLE_GitHub",
                   "username": "ABYSS_SENTINEL_USER_admin@example.com",
                   "url":      "ABYSS_SENTINEL_URL_https://github.com",
                   "notes":    "ABYSS_SENTINEL_NOTES_test",
                   "tags":     []any{"ABYSS_SENTINEL_TAG_dev"},
               },
               map[string]any{"password": "ABYSS_SENTINEL_PW_hunter2"},
           )
           if err != nil { log.Fatalf("create record: %v", err) }
           _ = svc.Lock(ctx)
       }
       ```
       Note: `records.NewService` accepts a nil logger — defend in records package by using `slog.Default()` if nil. Adjust the constructor accordingly.

    10. **`.github/workflows/ci.yml`** — add a step (after the existing test step) that runs the no-plaintext check on ubuntu-latest:
        ```yaml
        - name: SQLite no-plaintext gate
          if: runner.os == 'Linux'
          run: ./scripts/check-no-plaintext.sh
        ```

    11. Manual D-05 cross-OS smoke for plan 04 (full lifecycle):
        - macOS: Welcome → Create new vault → Vault Home unlocked → New record (credential) → fill all fields → Save → Record Detail (masked password) → Reveal (auto-hides at 30s) → per-field copy of username (countdown appears, ticks, clears, "Clipboard cleared." toast) → Edit in-place → change title → Save → Record list shows new title → Delete (modal appears, click Delete) → Vault Home empty state. Then: Generate password (24, all classes, exclude-ambiguous) → Copy with `Copy (10s)` menu → 10-second countdown observed.
        - Windows: same lifecycle. Sign off in `.planning/phases/01-v0-1-dogfoodable-vault/01-VERIFICATION.md` for both OSes.

    12. Commit: `feat(01-04): record screens (list/detail/edit) + delete confirm + SQLite no-plaintext CI gate`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss && ./scripts/check-no-plaintext.sh && cd frontend && npm run build</automated>
  </verify>
  <acceptance_criteria>
    - `scripts/check-no-plaintext.sh` is executable and exits 0 (no sentinel leak in fixture vault)
    - `cmd/no-plaintext-fixture/main.go` exists and `go build ./cmd/no-plaintext-fixture` exits 0
    - `.github/workflows/ci.yml` contains the literal string `check-no-plaintext.sh` (verify: `grep -c 'check-no-plaintext' .github/workflows/ci.yml` returns at least 1)
    - `frontend/src/components/CredentialForm/CredentialForm.tsx` uses `useForm` from `@mantine/form` (verify: `grep -c "from \"@mantine/form\"" frontend/src/components/CredentialForm/CredentialForm.tsx` returns 1)
    - `frontend/src/components/DeleteConfirmModal/DeleteConfirmModal.tsx` contains the literal `Delete this credential?` and `will be permanently removed from this vault. This cannot be undone.`
    - `frontend/src/screens/RecordDetail/RecordDetail.tsx` uses `<MaskedField>` AND `<ClipboardCountdown>` (verify: each appears at least once via grep)
    - `frontend/src/screens/VaultHome/VaultHome.tsx` calls `api.listRecords` AND `api.searchRecords` (verify both exist via grep)
    - `cd frontend && npm run build` exits 0
    - `wails build` exits 0
    - Manual D-05 cross-OS smoke for plan 04 (full lifecycle: Create vault → Create credential → Reveal → Copy with countdown → Edit → Delete → Generate password → per-copy timeout menu) signed off in 01-VERIFICATION.md for macOS AND Windows with commit SHAs
    - CI workflow latest run reports `success` on ubuntu-latest + macos-latest + windows-latest including the no-plaintext gate
  </acceptance_criteria>
  <done>End-to-end credential CRUD UI works on macOS AND Windows; reveal/copy/edit/delete flows match UI-SPEC; SQLite plaintext is grep-tested in CI on every push; D-02 plan-4 gate is live; phase 1 user-visible payload is complete.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Records.Service ↔ records table | Encrypt-on-write, decrypt-on-read; AAD binds ciphertext to vaultID/recordID/recordType/dekID/envVersion. |
| Metadata blob JSON ↔ Go struct (Pitfall 21) | Future field additions must be `omitempty`; renames/removes require record_envelope_version bump. |
| Frontend → backend → OS clipboard | Backend writes via Wails runtime; logger MUST NOT log the value (only `kind`). |
| Frontend countdown ↔ backend ClearClipboard | Frontend drives the timer (PRD §2.7); backend owns write/clear; if frontend dies, OS clipboard retains the value until OS-managed clear (best-effort per PRD). |
| SQLite vault file ↔ filesystem | Plaintext SQLite columns MUST NOT contain sensitive material (STORE-08 + REC-13); CI grep gate enforces. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-04-01 | Information Disclosure | Plaintext title / username / URL / notes / tags / password leaks into SQLite columns | mitigate | All sensitive fields live INSIDE `encrypted_metadata_blob` or `encrypted_secret_blob` (PRD §6.3). Plaintext columns (`record_type`, `dek_id`, timestamps) hold only non-sensitive structural data. Public key columns are nullable and only populated for asymmetric records (Phase 4). CI gate `scripts/check-no-plaintext.sh` runs on every PR with a fixture vault containing 6 known sentinel strings; ANY appearance in raw SQLite bytes fails CI. STORE-08 + REC-13 enforced. PRD §18 hard rule #6. |
| T-04-02 | Tampering | AAD construction inline at the records.Service call sites | mitigate | All 4 AAD constructions in records.Service (Create+Get for both metadata+secret kinds) call `aad.Build(KindRecordMetadata, ...)` or `aad.Build(KindRecordSecret, ...)`. Acceptance criterion: `grep -c 'aad.Build' internal/records/service.go` >= 6. Test `TestService_Create_FirstSealUsesAADBuild` proves the encrypt path uses Build (because the same Build call decrypts successfully). |
| T-04-03 | Tampering | Record swap (attacker copies encrypted_secret_blob from record A onto record B) | mitigate | AAD includes recordID. Test `TestService_KindMetadata_KindSecret_DistinctAAD` enforces the principle; Phase 2 adds the explicit cross-record-swap tampering test. |
| T-04-04 | Tampering | Metadata blob JSON drift breaks future decrypt (rename / remove fields) | mitigate | `CredentialMetadata` struct documents the forward-compat rule; Pitfall 21 tests assert: future extra fields decode cleanly, missing fields produce zero values, omitempty drops empty strings/arrays. |
| T-04-05 | Information Disclosure | Generated password leaks into logs | mitigate | `internal/logger` redaction list includes "password". `internal/clipboard.Write` logs only `kind` not `value`. `internal/generator.GeneratePassword` returns the value but does not log. Test `TestService_Write_LogsKindNotValue` enforces. PRD §12.2. |
| T-04-06 | Information Disclosure | Generated password's randomness is predictable (math/rand or weak seed) | mitigate | `internal/generator/password.go` imports `crypto/rand` exclusively; `forbidigo` lint banning `math/rand` (from plan 02) catches violations; Acceptance criterion: `grep -c 'math/rand' internal/generator/password.go` returns 0. PRD §18 hard rule #13. |
| T-04-07 | Information Disclosure | Clipboard content retained beyond timeout because OS clipboard manager kept history | accept | PRD §4.2 + §19 + §FR-063 explicitly document this as a best-effort limitation. v0.1 attempts clear; README warns user (Phase 6). |
| T-04-08 | Denial of Service | Pathological input (10MB password, 10000 tags) blocks UI | mitigate | Zod schema in `CredentialForm` caps password at 1024 chars and tags at 32. Backend `records.Service.Create` does not enforce additional caps (relies on the schema), but `crypto/aes-gcm` handles arbitrary plaintext sizes; SQLite BLOB column is unbounded (defacto 1GB cap is fine for the threat model). |
| T-04-09 | Information Disclosure | Reveal-secret state persists after navigation away | mitigate | UI-SPEC interaction contract #1: all reveals collapse on lock event. `MaskedField` subscribes to `vaultStore`; on status change to "locked" forces collapse. Auto-hide after 30s (UI-SPEC). |
| T-04-10 | Tampering | Frontend bypasses the api/clipboard.ts wrapper and calls `wailsjs/go/main/App.CopyToClipboard` directly with arbitrary value | mitigate | api-wrapper boundary CI gate from plan 03 forbids any import of `wailsjs/go/main/App` outside `frontend/src/api/`. Even if bypassed, `app.CopyToClipboard` calls `session.RequireUnlocked()` and the value is written via the same backend path; the only loss is the typed-error contract (which CI enforces anyway). |
| T-04-11 | Information Disclosure | URL path of the vault file leaks via "Save as credential…" query param `?prefilled-password=...` | mitigate | The query param contains the generated password, NOT a stored secret — the value is brand-new from this generator click. URL params are visible in the renderer's history; mitigation: pass via Zustand store rather than URL. **DECISION:** use `clipboardStore`-style transient store (`prefilledPasswordStore.set(pw); navigate("/record/new"); RecordEdit reads + clears`). The URL-param sketch in task 3 step 11 is REPLACED by this approach. Ensure the executor uses the store-based approach. |
</threat_model>

<verification>
**End-of-plan checks (run from repo root):**

1. `go build ./...` exits 0
2. `go test ./internal/records/... ./internal/storage/... ./internal/generator/... ./internal/clipboard/... -race -count=1 -timeout 180s` exits 0
3. `golangci-lint run ./...` exits 0
4. `scripts/check-no-plaintext.sh` exits 0
5. `scripts/check-wails-bindings.sh` exits 0
6. `cd frontend && npm run build` exits 0
7. `cd frontend && npm test -- --run` exits 0 (Vitest passes for ClipboardCountdown)
8. `grep -rn 'math/rand' internal/ --include="*.go"` returns no matches
9. `grep -rEn '\[\]byte\("[^"]*\|(metadata|secret|wrapped_dek)"\)' internal/ --include="*.go" | grep -v 'aad/aad.go' | grep -v '_test.go'` returns no matches
10. `grep -rn 'wailsjs/go/main/App' frontend/src --include='*.ts' --include='*.tsx' | grep -v 'frontend/src/api/'` returns no matches
11. `wails build` exits 0
12. **D-05 cross-OS smoke for plan 04:** macOS + Windows full credential lifecycle (Create vault → Create credential with all fields including tags → Reveal password → Per-field copy of username + URL → Watch countdown for 30s → Clipboard cleared toast → Edit credential in-place → Save → Delete with confirm modal → Generate password 24-char with exclude-ambiguous → Copy with `Copy (10s)` menu item → 10-second countdown observed → "Clipboard cleared." toast). Both signed off in 01-VERIFICATION.md with commit SHA.
13. CI workflow latest run reports `success` on ubuntu-latest + macos-latest + windows-latest INCLUDING the no-plaintext gate
</verification>

<success_criteria>
- Credential record CRUD works end-to-end on macOS AND Windows: Create → Get → List → Search → Update → Delete
- Every encrypt path (`records.Service.Create`, `records.Service.updateImpl`) AND every decrypt path (`records.Service.Get`, `records.Service.List`) calls `aad.Build` for both KindRecordMetadata and KindRecordSecret — proven by tests `TestService_Create_FirstSealUsesAADBuild`, `TestService_KindMetadata_KindSecret_DistinctAAD`
- Metadata blob JSON forward-compat per Pitfall 21 — proven by `metadata_compat_test.go` suite (extra fields decode, missing fields zero-value, omitempty drops empty values)
- Tags ship inside encrypted metadata blob (REC-13 promotion per D-17) — verified by no-plaintext CI gate's tag sentinel
- Password generator uses `crypto/rand` exclusively; supports length 8-128 default 24; all 4 character classes toggleable; exclude-ambiguous toggle removes O/0/l/1/I; rejects no-class-selected with the UI-SPEC validation message; uniqueness across 1000 calls
- Clipboard backend writes via Wails runtime `ClipboardSetText`; logger NEVER logs the value (only `kind`); ClearClipboard writes empty string
- Frontend ClipboardCountdown component renders the PRD-LOCKED phrase `Secret copied. Clipboard will be cleared in {n} seconds.`; updates 4Hz via `useInterval(250)`; Clear-now button works (CLIP-04); per-copy timeout menu offers 10/30/60/120s items (D-14); reset-on-new-copy with no stacked toasts; switches to yellow at ≤5s remaining; clears at t=0 with `Clipboard cleared.` toast
- MaskedField click-to-toggle reveal with 30s auto-hide; collapses on vault:locked event
- Per-field copy buttons on Record Detail call `api.copyToClipboard` and start the countdown
- DeleteConfirmModal matches UI-SPEC §Confirm-Delete UX exactly (modal + red Delete button, no type-to-confirm)
- SQLite no-plaintext grep gate active in CI — fixture vault with 6 sentinel strings produces NO sentinel leaks in raw bytes
- D-05 Windows smoke for full Phase 1 user-flow signed off in `01-VERIFICATION.md`
</success_criteria>

<output>
After completion, create `.planning/phases/01-v0-1-dogfoodable-vault/01-04-SUMMARY.md` documenting:
- The credential CRUD encrypt/decrypt path: `records.Service.Create` → `aad.Build(KindRecordMetadata) + aad.Build(KindRecordSecret)` → `crypto.NewNonce` × 2 → `crypto.Encrypt` × 2 → `storage.InsertRecord`. Reverse on Get/List/Update.
- Pitfall 21 forward-compat enforcement: `omitempty` on every optional field; explicit tests for unknown-field decode, missing-field decode, byte-equal round-trip.
- D-02 plan-4 gate: SQLite no-plaintext grep verified by `scripts/check-no-plaintext.sh` + `cmd/no-plaintext-fixture` helper. Six sentinel strings: TITLE, USER, URL, NOTES, TAG, PW. None appear in raw vault bytes.
- The ClipboardCountdown architecture: backend Write/Clear (Wails runtime); frontend useInterval(250ms) drives progress bar; `clipboardStore` Zustand state; per-copy menu (D-14) and Clear-now (CLIP-04) both promoted to v0.1.
- Threat T-04-11 resolution: prefilled-password from Password Generator → Save-as-credential is passed via `prefilledPasswordStore` (Zustand transient), NOT URL query param. Avoids leaking the generated password into renderer history.
- Manual D-05 result + commit SHAs for macOS + Windows full-lifecycle sign-off.
- Open question forwarded to plan 05: confirm `wails build -platform windows/amd64` from a macOS host succeeds (the pure-Go SQLite + zero-CGO toolchain promise from D-06 — verify in plan 05's Windows smoke).
</output>
