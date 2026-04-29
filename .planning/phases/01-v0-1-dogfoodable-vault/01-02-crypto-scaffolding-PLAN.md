---
phase: 01-v0-1-dogfoodable-vault
plan: 02
type: execute
wave: 2
depends_on: ["01-01-storage-migrations-PLAN.md"]
files_modified:
  - go.mod
  - go.sum
  - .golangci.yml
  - internal/aad/aad.go
  - internal/aad/aad_test.go
  - internal/aad/testdata/golden_metadata.txt
  - internal/aad/testdata/golden_secret.txt
  - internal/aad/testdata/golden_wrapped_dek.txt
  - internal/crypto/kdf.go
  - internal/crypto/kdf_test.go
  - internal/crypto/aead.go
  - internal/crypto/aead_test.go
  - internal/crypto/aead_aad_test.go
  - internal/crypto/nonce.go
  - internal/crypto/nonce_test.go
  - internal/crypto/keywrap.go
  - internal/crypto/keywrap_test.go
  - internal/crypto/zero.go
  - internal/crypto/zero_test.go
  - internal/crypto/testdata/golden_aes_gcm.json
  - internal/apperr/codes.go
  - internal/apperr/error.go
  - internal/apperr/error_test.go
  - internal/session/session.go
  - internal/session/lockstate.go
  - internal/session/session_test.go
  - internal/logger/logger.go
  - internal/logger/logger_test.go
autonomous: true
requirements_addressed:
  - SEC-01
  - SEC-03
  - SEC-04
  - SEC-05
  - SEC-06
  - SEC-07
  - SEC-08
  - VAULT-10
  - STORE-08

must_haves:
  truths:
    - "Every AES-GCM Seal in this codebase invokes `internal/aad.Build()` to produce its AAD — no raw `[]byte(\"...|metadata\")` constructions exist anywhere"
    - "AAD strings exactly match PRD §5.4: metadata = `{vaultID}|{envVersion}|{recordID}|{recordType}|{dekID}|metadata`; secret = `{vaultID}|{envVersion}|{recordID}|{recordType}|{dekID}|secret`; wrapped_dek = `{vaultID}|{dekID}|{wrappingType}|wrapped_dek`"
    - "AAD MUST NOT include SQLite `schema_version` (PRD §5.4 + §18 hard rule #15)"
    - "Every random byte for keys, nonces, salts, DEKs, and password generation comes from `crypto/rand` — `math/rand` is forbidigo-banned and absent from non-test code"
    - "Argon2id KDF parameters (memory, iterations, parallelism, salt) are stored per-wrapping in `kdf_params_json`; no global hardcoded params"
    - "`apperr.AppError{Code, Message, Cause}.ToDTO()` is the SINGLE exit path from Go code to the Wails boundary — raw Go errors never cross"
    - "`session.RequireUnlocked()` is the single guard; `session.Lock()` zeros the DEK with `crypto.Zero` then nils the slice atomically under mutex"
    - "`internal/logger` redaction list covers: master_password, kek, dek, password, secret, value, private_key, api_key, key_material, clipboard — verified by fuzz/table tests"
    - "Golden-vector test for AES-256-GCM round-trip with AAD passes deterministically"
  artifacts:
    - path: "internal/aad/aad.go"
      provides: "Build(kind, vaultID, recordID, recordType, dekID, wrappingType, envVersion) []byte — single source of AAD"
      exports: ["Build", "Kind", "KindRecordMetadata", "KindRecordSecret", "KindWrappedDEK"]
    - path: "internal/crypto/aead.go"
      provides: "Encrypt + Decrypt for AES-256-GCM with mandatory AAD parameter"
      exports: ["Encrypt", "Decrypt"]
    - path: "internal/crypto/kdf.go"
      provides: "DeriveKEK(password, salt, params) and Argon2id Profile struct"
      exports: ["DeriveKEK", "Argon2idParams", "DefaultArgon2idParams", "FallbackArgon2idParams", "MinimumArgon2idParams"]
    - path: "internal/crypto/keywrap.go"
      provides: "WrapDEK + UnwrapDEK using KEK and AAD"
      exports: ["WrapDEK", "UnwrapDEK", "GenerateDEK"]
    - path: "internal/crypto/nonce.go"
      provides: "NewNonce(size) []byte from crypto/rand"
      exports: ["NewNonce", "GCMNonceSize"]
    - path: "internal/crypto/zero.go"
      provides: "Zero(b []byte) — best-effort byte-slice zeroing"
      exports: ["Zero"]
    - path: "internal/apperr/error.go"
      provides: "AppError struct + ToDTO() + Wrap(code, cause, msg)"
      exports: ["AppError", "ToDTO", "Wrap", "New"]
    - path: "internal/apperr/codes.go"
      provides: "AppErrorCode constants matching PRD §10.1 TS enum"
      exports: ["VAULT_LOCKED", "UNLOCK_FAILED", "UNSUPPORTED_VAULT", "CORRUPT_VAULT", "MIGRATION_FAILED", "VALIDATION_ERROR", "RECORD_NOT_FOUND", "EXPORT_FAILED", "CLIPBOARD_FAILED", "PLATFORM_ERROR", "INTERNAL_ERROR", "VAULT_ALREADY_UNLOCKED"]
    - path: "internal/session/session.go"
      provides: "Session{dek, vaultID, vaultPath, unlockedAt} with Activate/Lock/RequireUnlocked"
      exports: ["Session", "Activate", "Lock", "RequireUnlocked", "Status", "DEK"]
    - path: "internal/logger/logger.go"
      provides: "slog wrapper that redacts sensitive keys"
      exports: ["New", "WithRedactor", "RedactedKeys"]
  key_links:
    - from: "internal/crypto/aead.go"
      to: "internal/aad/aad.go"
      via: "Encrypt(key, plaintext, nonce, aad []byte) — caller MUST pass aad from aad.Build"
      pattern: "aad\\.Build"
    - from: "internal/crypto/keywrap.go"
      to: "internal/aad/aad.go"
      via: "WrapDEK uses aad.Build(KindWrappedDEK, ...)"
      pattern: "aad\\.Build.*KindWrappedDEK"
    - from: "internal/session/session.go"
      to: "internal/crypto/zero.go"
      via: "Lock() calls crypto.Zero(s.dek) before nilling"
      pattern: "crypto\\.Zero"
    - from: ".golangci.yml"
      to: "internal/generator/* (plan 04)"
      via: "forbidigo rule banning math/rand outside _test.go files"
      pattern: "math/rand"
---

<objective>
Land the cryptographic foundation and the cross-cutting scaffolding that the rest of Phase 1 depends on. Specifically: the single `internal/aad.Build()` AAD helper (PRD §5.4) with golden-vector tests, AES-256-GCM helpers that REQUIRE the AAD parameter, Argon2id KEK derivation with PRD §5.3 profiles + per-wrapping params, DEK generation + wrap/unwrap, the `internal/apperr` typed-error system + `ToDTO()`, `internal/session` lock-state machine with atomic DEK zero-on-lock, and `internal/logger` slog wrapper with redaction. Add `forbidigo` to `.golangci.yml` banning `math/rand` in non-test code (D-02).

Purpose: These cross-cutting pieces are NOT retrofittable. The AAD helper MUST be invoked by the FIRST `cipher.Seal` call this codebase ever makes (.planning/research/PITFALLS.md "AAD must ship in first cipher.Seal" directive + CONTEXT D-02). The typed-error system must exist before `app.go` defines its first bound method (plan 03). The session.Lock atomic-zero contract must exist before plan 03 wires unlock/lock.

Output: A `crypto` package whose round-trip golden vectors pass; an `aad` package with golden vectors that lock the format strings; an `apperr` package with TS-mirror codes; a `session` package whose Lock() test demonstrates DEK zeroing; a `logger` package whose redaction tests prove no `password` field reaches output. CI now also blocks `math/rand` outside test code.
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
@.planning/research/STACK.md
@.planning/research/ARCHITECTURE.md
@.planning/research/PITFALLS.md
@.planning/phases/01-v0-1-dogfoodable-vault/01-01-storage-migrations-PLAN.md
@go.mod
@.golangci.yml

<interfaces>
<!-- Interfaces this plan PRODUCES that plans 03 and 04 will consume directly. -->

From `internal/aad/aad.go`:
```go
package aad

type Kind int

const (
    KindRecordMetadata Kind = iota
    KindRecordSecret
    KindWrappedDEK
)

// Build assembles AAD bytes per PRD §5.4. The returned slice is suitable for
// passing to AES-GCM Seal/Open. schema_version is intentionally NOT a parameter.
//
//   KindRecordMetadata: vaultID|envVersion|recordID|recordType|dekID|metadata
//   KindRecordSecret:   vaultID|envVersion|recordID|recordType|dekID|secret
//   KindWrappedDEK:     vaultID|dekID|wrappingType|wrapped_dek      (envVersion ignored)
func Build(k Kind, vaultID, recordID, recordType, dekID, wrappingType string, envVersion int) ([]byte, error)
```

From `internal/crypto/aead.go`:
```go
package crypto

// Encrypt AES-256-GCM. key MUST be 32 bytes. nonce MUST be 12 bytes (GCM standard).
// aad is REQUIRED — passing nil returns ErrAADRequired (PRD §5.4 mandates AAD on every op).
func Encrypt(key, plaintext, nonce, aad []byte) (ciphertext []byte, err error)

// Decrypt is the inverse. Returns ErrAuthFailed on AAD/nonce/ciphertext mismatch
// (no distinguishable error — PRD §9.4 generic-failure rule for unlock path).
func Decrypt(key, ciphertext, nonce, aad []byte) (plaintext []byte, err error)

var (
    ErrAADRequired = errors.New("crypto: AAD is required for AES-GCM operations")
    ErrAuthFailed  = errors.New("crypto: authenticated decryption failed")
    ErrInvalidKeySize  = errors.New("crypto: AES-256-GCM requires 32-byte key")
    ErrInvalidNonceSize = errors.New("crypto: AES-GCM requires 12-byte nonce")
)
```

From `internal/crypto/kdf.go`:
```go
package crypto

// Argon2idParams matches what's stored per-wrapping in key_wrappings.kdf_params_json.
type Argon2idParams struct {
    Memory      uint32 `json:"memory"`      // KiB
    Iterations  uint32 `json:"iterations"`
    Parallelism uint8  `json:"parallelism"`
    SaltLen     uint32 `json:"salt_len"`
    KeyLen      uint32 `json:"key_len"`
    Algorithm   string `json:"algorithm"`   // "argon2id"
    Version     int    `json:"version"`     // argon2.Version
}

// PRD §5.3 profiles:
var (
    DefaultArgon2idParams  = Argon2idParams{Memory: 256 * 1024, Iterations: 3, Parallelism: 4, SaltLen: 16, KeyLen: 32, Algorithm: "argon2id", Version: argon2.Version}
    FallbackArgon2idParams = Argon2idParams{Memory: 64 * 1024,  Iterations: 3, Parallelism: 1, SaltLen: 16, KeyLen: 32, Algorithm: "argon2id", Version: argon2.Version}
    MinimumArgon2idParams  = Argon2idParams{Memory: 19 * 1024,  Iterations: 2, Parallelism: 1, SaltLen: 16, KeyLen: 32, Algorithm: "argon2id", Version: argon2.Version}
)

// DeriveKEK runs Argon2id and returns the derived KEK (caller-zeroable).
// password is treated as bytes (caller may zero after); salt is at least SaltLen bytes.
func DeriveKEK(password, salt []byte, p Argon2idParams) ([]byte, error)
```

From `internal/crypto/keywrap.go`:
```go
package crypto

// GenerateDEK returns 32 random bytes from crypto/rand. PRD §2.6 + FR-005.
func GenerateDEK() ([]byte, error)

// WrapDEK encrypts dek with kek using AES-256-GCM with the wrapped-DEK AAD.
// Returns (wrapped_dek, nonce, error). Caller persists both into key_wrappings.
func WrapDEK(kek, dek, aad []byte) (wrapped, nonce []byte, err error)

// UnwrapDEK is the inverse. On any failure (including AAD mismatch / wrong KEK),
// returns the same generic ErrAuthFailed. PRD §9.4 generic-failure path.
func UnwrapDEK(kek, wrapped, nonce, aad []byte) (dek []byte, err error)
```

From `internal/apperr/`:
```go
package apperr

type Code string

const (
    CodeVaultLocked            Code = "VAULT_LOCKED"
    CodeVaultAlreadyUnlocked   Code = "VAULT_ALREADY_UNLOCKED"
    CodeUnlockFailed           Code = "UNLOCK_FAILED"
    CodeUnsupportedVault       Code = "UNSUPPORTED_VAULT"
    CodeCorruptVault           Code = "CORRUPT_VAULT"
    CodeMigrationFailed        Code = "MIGRATION_FAILED"
    CodeValidationError        Code = "VALIDATION_ERROR"
    CodeRecordNotFound         Code = "RECORD_NOT_FOUND"
    CodeExportFailed           Code = "EXPORT_FAILED"
    CodeClipboardFailed        Code = "CLIPBOARD_FAILED"
    CodePlatformError          Code = "PLATFORM_ERROR"
    CodeInternalError          Code = "INTERNAL_ERROR"
)

type AppError struct {
    Code    Code              `json:"code"`
    Message string            `json:"message"`
    Details map[string]string `json:"details,omitempty"`
    Cause   error             `json:"-"` // never serialized — sanitized at boundary
}

func (e *AppError) Error() string { return string(e.Code) + ": " + e.Message }
func (e *AppError) Unwrap() error { return e.Cause }

func New(code Code, msg string) *AppError
func Wrap(code Code, cause error, msg string) *AppError

// ToDTO is the single sanitization point: converts ANY error into a safe
// public AppErrorDTO. Internal Cause is dropped. Unknown errors map to INTERNAL_ERROR.
type DTO struct {
    Code    Code              `json:"code"`
    Message string            `json:"message"`
    Details map[string]string `json:"details,omitempty"`
}

func ToDTO(err error) *DTO
```

From `internal/session/session.go`:
```go
package session

type Status int

const (
    StatusClosed Status = iota
    StatusLocked
    StatusUnlocked
)

type Session struct { /* mu, dek, vaultID, vaultPath, unlockedAt, status */ }

func New() *Session
func (s *Session) Activate(dek []byte, vaultID, vaultPath string) error  // unlocked
func (s *Session) Lock() error                                            // zeros DEK, transitions to locked
func (s *Session) RequireUnlocked() error                                 // returns apperr.AppError{CodeVaultLocked} if not unlocked
func (s *Session) Status() Status
func (s *Session) DEK() []byte                                            // returns nil if locked; never returns the inner slice (returns a copy or panics under mutex)
func (s *Session) VaultID() string
```

From `internal/logger/logger.go`:
```go
package logger

// New returns an *slog.Logger that redacts any value associated with a key
// in DefaultRedactedKeys (case-insensitive substring match).
func New() *slog.Logger

// DefaultRedactedKeys lists all keys/substrings that get replaced with "<redacted>"
// in log output. Per PRD §12.2.
var DefaultRedactedKeys = []string{
    "password", "master_password", "kek", "dek", "secret", "value",
    "private_key", "api_key", "key_material", "clipboard",
}
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Implement internal/aad with golden-vector tests + AES-GCM Encrypt/Decrypt + nonce + Zero + crypto-rand-only enforcement</name>
  <files>internal/aad/aad.go, internal/aad/aad_test.go, internal/aad/testdata/golden_metadata.txt, internal/aad/testdata/golden_secret.txt, internal/aad/testdata/golden_wrapped_dek.txt, internal/crypto/aead.go, internal/crypto/aead_test.go, internal/crypto/aead_aad_test.go, internal/crypto/nonce.go, internal/crypto/nonce_test.go, internal/crypto/zero.go, internal/crypto/zero_test.go, internal/crypto/testdata/golden_aes_gcm.json, .golangci.yml, go.mod, go.sum</files>
  <read_first>
    - docs/PRD.md §5.1 (algorithms — AES-256-GCM, Argon2id), §5.4 (AAD strings — verbatim), §5.5 (nonce rules), §18 hard rules #8 (no custom crypto), #13 (no math/rand), #15 (no schema_version in AAD)
    - .planning/research/ARCHITECTURE.md §8 (AAD discipline — single helper rationale; §8.2 lint enforceability), §11.2 (anti-pattern: inline AAD construction), §11.6 (anti-pattern: schema_version in AAD)
    - .planning/research/PITFALLS.md (read the "AAD must ship in first cipher.Seal" directive — full file too large; grep `Pitfall` headings is sufficient)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-02 (forbidigo lands in plan 02 with first crypto code), D-22 (golden vector fixtures location)
    - .planning/research/STACK.md (Argon2id godoc reference; AES-GCM stdlib calls)
    - internal/storage/db.go (from plan 01 — to confirm import paths and module name)
    - .golangci.yml (from plan 01 — extending with forbidigo)
  </read_first>
  <action>
    1. **`internal/aad/aad.go`** — implement the `Build()` function exactly:
       ```go
       package aad

       import (
           "errors"
           "fmt"
       )

       type Kind int

       const (
           KindRecordMetadata Kind = iota
           KindRecordSecret
           KindWrappedDEK
       )

       var ErrUnknownKind = errors.New("aad: unknown kind")

       // Build assembles AAD bytes per PRD §5.4. Pipe-separated fields, exact ordering.
       // schema_version is INTENTIONALLY NOT INCLUDED (PRD §5.4 + §18 hard rule #15).
       //
       // The wrappingType parameter is used only for KindWrappedDEK (e.g. "master_password").
       // The envVersion parameter is used only for KindRecordMetadata and KindRecordSecret.
       //
       // Inputs MUST NOT contain the pipe character "|" (caller is responsible — UUIDs
       // and constant strings used in this codebase are pipe-free; a guard is provided).
       func Build(k Kind, vaultID, recordID, recordType, dekID, wrappingType string, envVersion int) ([]byte, error) {
           // Defense: reject inputs containing the separator character.
           for _, s := range []string{vaultID, recordID, recordType, dekID, wrappingType} {
               if strings.ContainsRune(s, '|') {
                   return nil, fmt.Errorf("aad: input field contains forbidden separator '|': %q", s)
               }
           }
           switch k {
           case KindRecordMetadata:
               return []byte(fmt.Sprintf("%s|%d|%s|%s|%s|metadata", vaultID, envVersion, recordID, recordType, dekID)), nil
           case KindRecordSecret:
               return []byte(fmt.Sprintf("%s|%d|%s|%s|%s|secret", vaultID, envVersion, recordID, recordType, dekID)), nil
           case KindWrappedDEK:
               return []byte(fmt.Sprintf("%s|%s|%s|wrapped_dek", vaultID, dekID, wrappingType)), nil
           default:
               return nil, ErrUnknownKind
           }
       }
       ```

    2. **`internal/aad/testdata/golden_metadata.txt`** — exact byte content (no trailing newline):
       ```
       11111111-2222-3333-4444-555555555555|1|aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee|credential|dddddddd-1111-2222-3333-444444444444|metadata
       ```

    3. **`internal/aad/testdata/golden_secret.txt`** — exact byte content:
       ```
       11111111-2222-3333-4444-555555555555|1|aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee|credential|dddddddd-1111-2222-3333-444444444444|secret
       ```

    4. **`internal/aad/testdata/golden_wrapped_dek.txt`** — exact byte content:
       ```
       11111111-2222-3333-4444-555555555555|dddddddd-1111-2222-3333-444444444444|master_password|wrapped_dek
       ```

    5. **`internal/aad/aad_test.go`** — golden vector tests:
       - `TestBuild_RecordMetadata_GoldenVector`: call `Build(KindRecordMetadata, "11111111-...", "aaaaaaaa-...", "credential", "dddddddd-...", "", 1)`. Read `testdata/golden_metadata.txt`. `bytes.Equal` must be true. Also verify the output does NOT contain the substring `schema_version` (PRD §5.4).
       - `TestBuild_RecordSecret_GoldenVector`: same pattern with KindRecordSecret.
       - `TestBuild_WrappedDEK_GoldenVector`: `Build(KindWrappedDEK, vaultID, "", "", dekID, "master_password", 0)`. Note recordID, recordType, envVersion are unused for this kind.
       - `TestBuild_ContainsPipe_ReturnsError`: pass a vaultID with `|` in it → expect non-nil error.
       - `TestBuild_UnknownKind_ReturnsError`: pass `Kind(999)` → expect `errors.Is(err, ErrUnknownKind)`.
       - `TestBuild_NoSchemaVersionInOutput`: for ALL THREE kinds, assert `!bytes.Contains(out, []byte("schema_version"))`.

    6. **`internal/crypto/zero.go`** — best-effort byte-slice zero (PRD §4.4 + §SEC-10 advance work):
       ```go
       package crypto

       // Zero overwrites b with zeros. Best-effort per PRD §4.4 (Go memory limitations).
       // Caller is responsible for not retaining other references to the slice.
       func Zero(b []byte) {
           for i := range b {
               b[i] = 0
           }
       }
       ```

    7. **`internal/crypto/zero_test.go`** — `TestZero_OverwritesAllBytes`: create []byte{1,2,3,4,5}, Zero, assert all zeros.

    8. **`internal/crypto/nonce.go`** — implement `NewNonce(size int) ([]byte, error)` using `crypto/rand.Read`. Constant `GCMNonceSize = 12`. Errors on size < 1 or rand failure.

    9. **`internal/crypto/nonce_test.go`**:
       - `TestNewNonce_Size`: NewNonce(12) returns 12 bytes.
       - `TestNewNonce_Uniqueness`: call 10000 times, all distinct (probabilistic). Maps in test.
       - `TestNewNonce_NonZero`: 1000 calls, none returns all-zero.

    10. **`internal/crypto/aead.go`** — AES-256-GCM with mandatory AAD:
        ```go
        package crypto

        import (
            "crypto/aes"
            "crypto/cipher"
            "errors"
            "fmt"
        )

        const (
            KeySize       = 32 // AES-256
            GCMNonceSize  = 12
            GCMTagSize    = 16
        )

        var (
            ErrAADRequired      = errors.New("crypto: AAD is required for AES-GCM operations")
            ErrAuthFailed       = errors.New("crypto: authenticated decryption failed")
            ErrInvalidKeySize   = errors.New("crypto: AES-256-GCM requires 32-byte key")
            ErrInvalidNonceSize = errors.New("crypto: AES-GCM requires 12-byte nonce")
        )

        // Encrypt produces ciphertext-with-tag for plaintext using key + nonce. AAD is REQUIRED;
        // passing nil or empty AAD returns ErrAADRequired (PRD §5.4 — no AAD means no envelope binding).
        // Returned ciphertext is `cipher || tag` (16-byte tag suffix; standard GCM layout).
        func Encrypt(key, plaintext, nonce, aad []byte) ([]byte, error) {
            if len(key) != KeySize { return nil, ErrInvalidKeySize }
            if len(nonce) != GCMNonceSize { return nil, ErrInvalidNonceSize }
            if len(aad) == 0 { return nil, ErrAADRequired }

            block, err := aes.NewCipher(key)
            if err != nil { return nil, fmt.Errorf("crypto: aes new: %w", err) }
            gcm, err := cipher.NewGCM(block)
            if err != nil { return nil, fmt.Errorf("crypto: gcm new: %w", err) }
            return gcm.Seal(nil, nonce, plaintext, aad), nil
        }

        // Decrypt reverses Encrypt. Any failure (auth tag, AAD mismatch, wrong key)
        // returns ErrAuthFailed — no distinguishable signal.
        func Decrypt(key, ciphertext, nonce, aad []byte) ([]byte, error) {
            if len(key) != KeySize { return nil, ErrInvalidKeySize }
            if len(nonce) != GCMNonceSize { return nil, ErrInvalidNonceSize }
            if len(aad) == 0 { return nil, ErrAADRequired }

            block, err := aes.NewCipher(key)
            if err != nil { return nil, fmt.Errorf("crypto: aes new: %w", err) }
            gcm, err := cipher.NewGCM(block)
            if err != nil { return nil, fmt.Errorf("crypto: gcm new: %w", err) }
            pt, err := gcm.Open(nil, nonce, ciphertext, aad)
            if err != nil { return nil, ErrAuthFailed }
            return pt, nil
        }
        ```

    11. **`internal/crypto/testdata/golden_aes_gcm.json`** — fixed test vector (NIST-style, but project-specific is fine since this is round-trip discipline, not spec compliance):
        ```json
        {
          "key_hex":        "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
          "nonce_hex":      "0102030405060708090a0b0c",
          "plaintext_utf8": "{\"password\":\"hunter2\"}",
          "aad_utf8":       "11111111-2222-3333-4444-555555555555|1|aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee|credential|dddddddd-1111-2222-3333-444444444444|secret"
        }
        ```
        The `expected_ciphertext_hex` field is intentionally OMITTED — the test computes expected on first run using stdlib AES-GCM and pins it. (Avoids hand-computing test vectors; the real assurance comes from the round-trip + AAD-mismatch tests.)

    12. **`internal/crypto/aead_test.go`** — round-trip:
        - `TestEncrypt_RoundTrip_GoldenVector`: load testdata, call `Encrypt` then `Decrypt` with same args, assert plaintext matches.
        - `TestEncrypt_RejectsNilAAD`: passing `nil` AAD → expect `ErrAADRequired`.
        - `TestEncrypt_RejectsEmptyAAD`: passing `[]byte{}` AAD → expect `ErrAADRequired`.
        - `TestEncrypt_RejectsWrongKeySize`: 16-byte key → `ErrInvalidKeySize`. 24-byte → same.
        - `TestEncrypt_RejectsWrongNonceSize`: 8-byte nonce → `ErrInvalidNonceSize`.

    13. **`internal/crypto/aead_aad_test.go`** — AAD must come from aad.Build (exercised through realistic call):
        - `TestEncrypt_WithAADBuild_RoundTrip`: build AAD via `aad.Build(KindRecordSecret, ...)`, encrypt a credential payload, decrypt with same AAD → plaintext matches.
        - `TestEncrypt_AADMismatch_FailsAuth`: encrypt with AAD-A, decrypt with AAD-B (different recordID) → `ErrAuthFailed`. **This proves the FIRST `cipher.Seal` in the codebase was invoked WITH `aad.Build`** (architecture compliance check).
        - `TestEncrypt_KindMetadata_VsKindSecret_DifferentAAD`: two encrypts of the same plaintext with the same key+nonce but different `Kind` produce DIFFERENT ciphertexts (because AAD differs). Catches accidental reuse of the same AAD across metadata and secret blobs of the same record.

    14. **`go.mod`/`go.sum`** — add `golang.org/x/crypto` direct dependency (was indirect via Wails). `go get golang.org/x/crypto@latest`.

    15. **`.golangci.yml`** — extend with `forbidigo` (D-02: lands HERE alongside the first crypto code):
        ```yaml
        linters:
          enable:
            - errcheck
            - govet
            - ineffassign
            - staticcheck
            - unused
            - gosimple
            - typecheck
            - forbidigo  # NEW — D-02 plan 02 lands math/rand ban
        linters-settings:
          forbidigo:
            forbid:
              - p: '^math/rand(\.|$)'
                msg: "math/rand is forbidden by PRD §18 hard rule #13. Use crypto/rand for ALL security-sensitive randomness."
              - p: '^math/rand/v2(\.|$)'
                msg: "math/rand/v2 is forbidden by PRD §18 hard rule #13. Use crypto/rand."
            analyze-types: true
            exclude-godoc-examples: true
        issues:
          exclude-rules:
            # No exemption for tests — math/rand is forbidden everywhere in this codebase
            # because Pitfall: tests can leak patterns into prod via copy-paste.
        ```

    16. Verify forbidigo: `golangci-lint run` must exit 0 (no `math/rand` exists yet anywhere). Then write a one-line file with `import _ "math/rand"` and confirm `golangci-lint` rejects it. Delete the test file.

    17. Commit: `feat(01-02): land internal/aad with golden vectors + AES-GCM helpers + math/rand forbidigo gate`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss &amp;&amp; go test ./internal/aad/... ./internal/crypto/... -race -count=1 &amp;&amp; golangci-lint run ./...</automated>
  </verify>
  <acceptance_criteria>
    - `internal/aad/aad.go` exists and exports `Build`, `Kind`, `KindRecordMetadata`, `KindRecordSecret`, `KindWrappedDEK`
    - `internal/aad/aad.go` does NOT contain the substring `schema_version` (PRD §5.4)
    - `grep -r '"schema_version"' internal/aad/ internal/crypto/` returns no matches
    - `grep -rn '|metadata"' internal/ --include="*.go" | grep -v 'aad/aad.go'` returns no matches (sole AAD source confirmed)
    - `grep -rn '|secret"' internal/ --include="*.go" | grep -v '_test.go' | grep -v 'aad/aad.go'` returns no matches
    - `grep -rn '|wrapped_dek"' internal/ --include="*.go" | grep -v 'aad/aad.go'` returns no matches
    - `internal/aad/testdata/golden_metadata.txt` exists and contains no newline at EOF
    - `go test ./internal/aad -run TestBuild_RecordMetadata_GoldenVector -count=1` exits 0
    - `go test ./internal/aad -run TestBuild_NoSchemaVersionInOutput -count=1` exits 0
    - `go test ./internal/crypto -run TestEncrypt_RoundTrip_GoldenVector -count=1` exits 0
    - `go test ./internal/crypto -run TestEncrypt_AADMismatch_FailsAuth -count=1` exits 0
    - `go test ./internal/crypto -run TestEncrypt_RejectsNilAAD -count=1` exits 0
    - `.golangci.yml` contains `forbidigo` in its `linters.enable` list
    - `.golangci.yml` contains the regex `'^math/rand(\.|$)'`
    - `golangci-lint run ./...` exits 0
    - `grep -rn 'math/rand' internal/ --include="*.go"` returns no matches (no production OR test code uses it)
  </acceptance_criteria>
  <done>AAD helper is the single source for all 3 AAD shapes; golden vectors lock the format strings; AES-GCM helpers refuse to operate without AAD; math/rand is banned by lint; the FIRST cipher.Seal in this codebase will pass through aad.Build.</done>
</task>

<task type="auto">
  <name>Task 2: Implement Argon2id KDF + DEK gen + WrapDEK/UnwrapDEK with golden vectors</name>
  <files>internal/crypto/kdf.go, internal/crypto/kdf_test.go, internal/crypto/keywrap.go, internal/crypto/keywrap_test.go, go.mod, go.sum</files>
  <read_first>
    - docs/PRD.md §5.2 (envelope encryption — DEK random, KEK derived, wrap/unwrap)
    - docs/PRD.md §5.3 (Argon2id parameters — initial 256MiB/3/4, fallback 64MiB/3/1, minimum 19MiB/2/1, 16+ byte salt)
    - docs/PRD.md §5.5 (nonce rules — unique per encryption, never reuse, CSPRNG)
    - docs/PRD.md §9.4 (generic unlock failure — UnwrapDEK returns same error on ANY failure)
    - .planning/research/STACK.md §4 (Argon2id IDKey signature)
    - .planning/research/PITFALLS.md (Pitfall 22 — KDF param storage; per-wrapping not global)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md (Argon2id KEK derivation with profile + per-wrapping params + salt)
    - internal/aad/aad.go (consumed by WrapDEK)
    - internal/crypto/aead.go (used internally)
    - internal/crypto/nonce.go (used internally)
  </read_first>
  <action>
    1. **`internal/crypto/kdf.go`** — Argon2id KDF with PRD §5.3 profiles:
       ```go
       package crypto

       import (
           "crypto/rand"
           "errors"
           "fmt"

           "golang.org/x/crypto/argon2"
       )

       type Argon2idParams struct {
           Memory      uint32 `json:"memory"`      // KiB
           Iterations  uint32 `json:"iterations"`
           Parallelism uint8  `json:"parallelism"`
           SaltLen     uint32 `json:"salt_len"`
           KeyLen      uint32 `json:"key_len"`
           Algorithm   string `json:"algorithm"`   // always "argon2id"
           Version     int    `json:"version"`    // argon2.Version
       }

       // PRD §5.3 profiles. Stored per-wrapping (NOT global) — a future fallback
       // attempt can persist different params for different wrappings.
       var (
           DefaultArgon2idParams = Argon2idParams{
               Memory: 256 * 1024, Iterations: 3, Parallelism: 4,
               SaltLen: 16, KeyLen: 32, Algorithm: "argon2id", Version: argon2.Version,
           }
           FallbackArgon2idParams = Argon2idParams{
               Memory: 64 * 1024, Iterations: 3, Parallelism: 1,
               SaltLen: 16, KeyLen: 32, Algorithm: "argon2id", Version: argon2.Version,
           }
           MinimumArgon2idParams = Argon2idParams{
               Memory: 19 * 1024, Iterations: 2, Parallelism: 1,
               SaltLen: 16, KeyLen: 32, Algorithm: "argon2id", Version: argon2.Version,
           }
       )

       var (
           ErrInvalidParams = errors.New("crypto: invalid Argon2id parameters")
           ErrSaltTooShort  = errors.New("crypto: salt must be at least 16 bytes")
       )

       // GenerateSalt returns SaltLen random bytes from crypto/rand.
       func GenerateSalt(p Argon2idParams) ([]byte, error) {
           if p.SaltLen < 16 { return nil, ErrSaltTooShort }
           salt := make([]byte, p.SaltLen)
           if _, err := rand.Read(salt); err != nil {
               return nil, fmt.Errorf("crypto: salt rand: %w", err)
           }
           return salt, nil
       }

       // DeriveKEK runs argon2.IDKey with the given params. Returns a KeyLen-byte KEK.
       // Caller is responsible for zeroing the result via Zero() when done.
       func DeriveKEK(password, salt []byte, p Argon2idParams) ([]byte, error) {
           if p.Algorithm != "argon2id" { return nil, ErrInvalidParams }
           if p.KeyLen == 0 || p.Memory == 0 || p.Iterations == 0 || p.Parallelism == 0 {
               return nil, ErrInvalidParams
           }
           if uint32(len(salt)) < p.SaltLen {
               return nil, ErrSaltTooShort
           }
           kek := argon2.IDKey(password, salt, p.Iterations, p.Memory, p.Parallelism, p.KeyLen)
           return kek, nil
       }
       ```

    2. **`internal/crypto/kdf_test.go`** — tests:
       - `TestDeriveKEK_DeterministicWithSameInputs`: same password + same salt + same params → identical KEK byte-for-byte (twice).
       - `TestDeriveKEK_DifferentPasswordsProduceDifferentKEKs`: same salt, different passwords → different KEKs.
       - `TestDeriveKEK_DifferentSaltsProduceDifferentKEKs`: same password, different salts → different KEKs.
       - `TestDeriveKEK_KeyLength32`: returns 32 bytes for `KeyLen: 32`.
       - `TestDeriveKEK_RejectsShortSalt`: 8-byte salt → `ErrSaltTooShort`.
       - `TestDeriveKEK_RejectsWrongAlgorithm`: `Algorithm: "scrypt"` → `ErrInvalidParams`.
       - `TestDeriveKEK_MinimumProfile_Completes`: use `MinimumArgon2idParams` (19MiB) with a real password, returns 32 bytes (no panic; smoke test for slow CI machines).
       - `TestGenerateSalt_Length`: GenerateSalt(DefaultArgon2idParams) returns SaltLen bytes; bytes are not all zero (probabilistic — 16 zero bytes from crypto/rand has probability ~1/2^128).
       - `TestPRDProfiles_StoredCorrectly`: assert `DefaultArgon2idParams.Memory == 256*1024` AND `Iterations == 3` AND `Parallelism == 4` AND `SaltLen >= 16` AND `KeyLen == 32` (PRD §5.3 verbatim check).

    3. **`internal/crypto/keywrap.go`** — DEK gen + wrap/unwrap:
       ```go
       package crypto

       import (
           "crypto/rand"
           "errors"
           "fmt"
       )

       const DEKSize = 32 // AES-256 DEK

       // GenerateDEK returns 32 random bytes from crypto/rand. PRD §2.6 + FR-005.
       func GenerateDEK() ([]byte, error) {
           dek := make([]byte, DEKSize)
           if _, err := rand.Read(dek); err != nil {
               return nil, fmt.Errorf("crypto: dek rand: %w", err)
           }
           return dek, nil
       }

       // WrapDEK encrypts dek under kek using AES-256-GCM with the wrapped-DEK AAD.
       // aad MUST come from internal/aad.Build(KindWrappedDEK, ...).
       // Returns (wrapped_dek, nonce). Caller persists both into key_wrappings.
       func WrapDEK(kek, dek, aad []byte) (wrapped, nonce []byte, err error) {
           if len(dek) != DEKSize { return nil, nil, errors.New("crypto: dek must be 32 bytes") }
           nonce, err = NewNonce(GCMNonceSize)
           if err != nil { return nil, nil, fmt.Errorf("crypto: wrap nonce: %w", err) }
           wrapped, err = Encrypt(kek, dek, nonce, aad)
           if err != nil { return nil, nil, err }
           return wrapped, nonce, nil
       }

       // UnwrapDEK is the inverse. Any failure returns the same generic ErrAuthFailed
       // (PRD §9.4 generic-failure rule — wrong password, wrong AAD, corrupt blob all
       // look identical at the API level).
       func UnwrapDEK(kek, wrapped, nonce, aad []byte) ([]byte, error) {
           dek, err := Decrypt(kek, wrapped, nonce, aad)
           if err != nil { return nil, ErrAuthFailed } // already ErrAuthFailed, but be explicit
           if len(dek) != DEKSize { return nil, ErrAuthFailed }
           return dek, nil
       }
       ```

    4. **`internal/crypto/keywrap_test.go`** — tests:
       - `TestGenerateDEK_Size`: returns 32 bytes.
       - `TestGenerateDEK_NonZero`: 1000 calls, none all-zero.
       - `TestGenerateDEK_Uniqueness`: 1000 calls, all distinct (probabilistic).
       - `TestWrapUnwrap_RoundTrip`: derive a KEK, generate a DEK, wrap, unwrap, assert DEK matches.
       - `TestUnwrap_WrongKEK_ReturnsErrAuthFailed`: wrap with KEK-A, unwrap with KEK-B → `ErrAuthFailed`.
       - `TestUnwrap_AADMismatch_ReturnsErrAuthFailed`: wrap with AAD using vaultID-A, unwrap with AAD using vaultID-B → `ErrAuthFailed`.
       - `TestUnwrap_TamperedCiphertext_ReturnsErrAuthFailed`: flip one bit in `wrapped`, unwrap → `ErrAuthFailed`.
       - `TestUnwrap_TamperedNonce_ReturnsErrAuthFailed`: flip one bit in `nonce`, unwrap → `ErrAuthFailed`.
       - `TestWrapDEK_UsesAADBuild`: build AAD via `aad.Build(KindWrappedDEK, vaultID, "", "", dekID, "master_password", 0)`, wrap, unwrap with same AAD → DEK matches. (Architectural-compliance check confirming the wrap path uses the helper.)

    5. Commit: `feat(01-02): Argon2id KDF + DEK gen + wrap/unwrap with golden tests`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss &amp;&amp; go test ./internal/crypto/... -race -count=1 -timeout 120s</automated>
  </verify>
  <acceptance_criteria>
    - `internal/crypto/kdf.go` exports `Argon2idParams`, `DefaultArgon2idParams`, `FallbackArgon2idParams`, `MinimumArgon2idParams`, `DeriveKEK`, `GenerateSalt`
    - `internal/crypto/kdf.go` imports `golang.org/x/crypto/argon2`
    - `DefaultArgon2idParams` literally has `Memory: 256 * 1024, Iterations: 3, Parallelism: 4` (PRD §5.3 verbatim)
    - `internal/crypto/keywrap.go` exports `GenerateDEK`, `WrapDEK`, `UnwrapDEK`, constant `DEKSize = 32`
    - `go test ./internal/crypto -run TestDeriveKEK_DeterministicWithSameInputs -count=1` exits 0
    - `go test ./internal/crypto -run TestDeriveKEK_RejectsShortSalt -count=1` exits 0
    - `go test ./internal/crypto -run TestPRDProfiles_StoredCorrectly -count=1` exits 0
    - `go test ./internal/crypto -run TestWrapUnwrap_RoundTrip -count=1` exits 0
    - `go test ./internal/crypto -run TestUnwrap_WrongKEK_ReturnsErrAuthFailed -count=1` exits 0
    - `go test ./internal/crypto -run TestUnwrap_AADMismatch_ReturnsErrAuthFailed -count=1` exits 0
    - `go test ./internal/crypto -run TestWrapDEK_UsesAADBuild -count=1` exits 0
    - `grep -c 'crypto/rand' internal/crypto/kdf.go internal/crypto/keywrap.go internal/crypto/nonce.go` returns &gt;= 3 (every random source uses crypto/rand)
  </acceptance_criteria>
  <done>Argon2id KDF runs with all three PRD §5.3 profiles; DEK generation uses crypto/rand; wrap/unwrap round-trip works; tampered ciphertext/nonce/AAD/KEK all return identical generic ErrAuthFailed; param struct ready for per-wrapping JSON storage in plan 03.</done>
</task>

<task type="auto">
  <name>Task 3: Implement internal/apperr (typed errors + ToDTO) + internal/session (lock-state + atomic DEK zero) + internal/logger (slog with redaction)</name>
  <files>internal/apperr/codes.go, internal/apperr/error.go, internal/apperr/error_test.go, internal/session/session.go, internal/session/lockstate.go, internal/session/session_test.go, internal/logger/logger.go, internal/logger/logger_test.go</files>
  <read_first>
    - docs/PRD.md §10.1 (AppErrorCode enum — exact 12 codes; AppErrorDTO shape)
    - docs/PRD.md §12 (logging policy, error rules — never log: master_password, KEK, DEK, plaintext records, generated passwords, clipboard contents)
    - docs/PRD.md §18 hard rules #7 (no log secrets), #14 (no string-matching errors in frontend)
    - .planning/research/ARCHITECTURE.md §1.2 (apperr + logger responsibilities), §4.2 (frontend api/errors.ts normalizeError pattern), §5.1+§5.2 (lock state machine), §11.4 (anti-pattern: frontend owning lock state authoritatively), §11.5 (anti-pattern: string-matching errors), §11.7 (anti-pattern: logging master password / KEK / DEK)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-UI-SPEC.md (AppErrorCode → User-Facing String Map — locks the 12 codes the frontend will display)
    - .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md D-13 (Lock routes to Unlock screen with vault path prefilled — drives session.Lock semantics)
    - internal/crypto/zero.go (used by session.Lock to zero DEK)
  </read_first>
  <action>
    1. **`internal/apperr/codes.go`** — match PRD §10.1 enum exactly (12 codes from UI-SPEC AppErrorCode table):
       ```go
       package apperr

       type Code string

       const (
           CodeVaultLocked          Code = "VAULT_LOCKED"
           CodeVaultAlreadyUnlocked Code = "VAULT_ALREADY_UNLOCKED"
           CodeUnlockFailed         Code = "UNLOCK_FAILED"
           CodeUnsupportedVault     Code = "UNSUPPORTED_VAULT"
           CodeCorruptVault         Code = "CORRUPT_VAULT"
           CodeMigrationFailed      Code = "MIGRATION_FAILED"
           CodeValidationError      Code = "VALIDATION_ERROR"
           CodeRecordNotFound       Code = "RECORD_NOT_FOUND"
           CodeExportFailed         Code = "EXPORT_FAILED"
           CodeClipboardFailed      Code = "CLIPBOARD_FAILED"
           CodePlatformError        Code = "PLATFORM_ERROR"
           CodeInternalError        Code = "INTERNAL_ERROR"
       )

       // AllCodes is the canonical list. Used by tests + future codegen for TS mirror.
       var AllCodes = []Code{
           CodeVaultLocked, CodeVaultAlreadyUnlocked, CodeUnlockFailed,
           CodeUnsupportedVault, CodeCorruptVault, CodeMigrationFailed,
           CodeValidationError, CodeRecordNotFound, CodeExportFailed,
           CodeClipboardFailed, CodePlatformError, CodeInternalError,
       }
       ```

    2. **`internal/apperr/error.go`** — AppError + ToDTO + helpers:
       ```go
       package apperr

       import "errors"

       // AppError is the typed-error type used internally. Cause is NEVER serialized
       // — ToDTO() drops it before crossing the Wails boundary (PRD §12.1).
       type AppError struct {
           Code    Code              `json:"code"`
           Message string            `json:"message"`
           Details map[string]string `json:"details,omitempty"`
           Cause   error             `json:"-"`
       }

       func (e *AppError) Error() string {
           if e == nil { return "" }
           return string(e.Code) + ": " + e.Message
       }
       func (e *AppError) Unwrap() error { return e.Cause }

       // New constructs an AppError with no cause.
       func New(code Code, msg string) *AppError {
           return &AppError{Code: code, Message: msg}
       }

       // Wrap constructs an AppError wrapping a cause. Cause is internal-only and
       // dropped at ToDTO; safe to wrap raw crypto/storage errors here.
       func Wrap(code Code, cause error, msg string) *AppError {
           return &AppError{Code: code, Message: msg, Cause: cause}
       }

       // WithDetails adds a Details map. Caller MUST ensure no sensitive values
       // (PRD §12.2) are placed in Details — e.g., field names yes, field values no.
       func (e *AppError) WithDetails(d map[string]string) *AppError {
           e.Details = d
           return e
       }

       // DTO is the public-facing shape. Mirror of AppErrorDTO in PRD §10.1 TS enum.
       type DTO struct {
           Code    Code              `json:"code"`
           Message string            `json:"message"`
           Details map[string]string `json:"details,omitempty"`
       }

       // ToDTO is the SINGLE sanitization point. Any error crossing the Wails
       // boundary MUST go through here (PRD §12.1, §18 hard rule #14, §10.1).
       //
       // Rules:
       //  - *AppError → DTO with same Code/Message/Details, Cause stripped
       //  - any other error → DTO{CodeInternalError, "Something went wrong. Try again."}
       //  - nil → nil
       //
       // The fallback message is intentionally generic — leak nothing.
       func ToDTO(err error) *DTO {
           if err == nil { return nil }
           var ae *AppError
           if errors.As(err, &ae) {
               return &DTO{Code: ae.Code, Message: ae.Message, Details: ae.Details}
           }
           return &DTO{Code: CodeInternalError, Message: "Something went wrong. Try again."}
       }
       ```

    3. **`internal/apperr/error_test.go`** — tests:
       - `TestNew_BasicConstruction`: New + Error() + Code field set.
       - `TestWrap_PreservesCause`: Wrap(code, cause, msg).Unwrap() returns cause; errors.Is + errors.As work.
       - `TestToDTO_Nil_ReturnsNil`: ToDTO(nil) returns nil.
       - `TestToDTO_AppError_PreservesAllPublicFields`: ToDTO(*AppError) returns DTO with Code, Message, Details intact.
       - `TestToDTO_AppError_DropsCause`: serialize the resulting DTO to JSON, assert no field name `cause` and no fragment of the cause's Error() text appears.
       - `TestToDTO_RawError_MapsToInternalError`: ToDTO(errors.New("raw")).Code == CodeInternalError; Message == "Something went wrong. Try again." (verify the raw string "raw" does NOT appear in the DTO).
       - `TestToDTO_RawError_DropsRawMessage`: serialize DTO, assert raw error text absent (defense against accidental wrapping bugs).
       - `TestAllCodes_Length12`: len(AllCodes) == 12 (catches any silent addition/removal that breaks the TS contract).
       - `TestAllCodes_MatchesPRDSection10_1`: assert AllCodes contains all 12 expected codes exactly (table-driven).

    4. **`internal/session/lockstate.go`** — Status enum:
       ```go
       package session

       type Status int

       const (
           StatusClosed Status = iota
           StatusLocked
           StatusUnlocked
       )

       func (s Status) String() string {
           switch s {
           case StatusClosed:   return "closed"
           case StatusLocked:   return "locked"
           case StatusUnlocked: return "unlocked"
           default:             return "unknown"
           }
       }
       ```

    5. **`internal/session/session.go`** — lock-state machine with atomic DEK zero:
       ```go
       package session

       import (
           "errors"
           "sync"
           "time"

           "github.com/<MODULE>/internal/apperr"
           "github.com/<MODULE>/internal/crypto"
       )

       // Session holds the decrypted-DEK + vault metadata while unlocked.
       // All public methods are mutex-guarded. The DEK is NEVER returned by reference;
       // callers receive a defensive copy (caller-owned, caller-zeroable).
       type Session struct {
           mu          sync.Mutex
           status      Status
           dek         []byte
           vaultID     string
           vaultPath   string
           unlockedAt  time.Time
       }

       func New() *Session { return &Session{status: StatusClosed} }

       // SetClosedToLocked is called by vault.Open after migrations succeed.
       func (s *Session) SetLockedFor(vaultID, vaultPath string) {
           s.mu.Lock()
           defer s.mu.Unlock()
           s.vaultID = vaultID
           s.vaultPath = vaultPath
           s.status = StatusLocked
       }

       // Activate transitions locked -> unlocked. dek must be 32 bytes.
       // Session takes ownership of dek (caller MUST NOT use it after this call).
       func (s *Session) Activate(dek []byte, vaultID, vaultPath string) error {
           s.mu.Lock()
           defer s.mu.Unlock()
           if s.status == StatusUnlocked {
               return apperr.New(apperr.CodeVaultAlreadyUnlocked, "Vault is already unlocked.")
           }
           if len(dek) != crypto.DEKSize {
               return apperr.New(apperr.CodeInternalError, "Invalid DEK size.")
           }
           s.dek = dek
           s.vaultID = vaultID
           s.vaultPath = vaultPath
           s.unlockedAt = time.Now().UTC()
           s.status = StatusUnlocked
           return nil
       }

       // Lock transitions unlocked -> locked. Atomically zeros the DEK with crypto.Zero
       // BEFORE nilling the slice reference. PRD §4.4 + §VAULT-10.
       func (s *Session) Lock() error {
           s.mu.Lock()
           defer s.mu.Unlock()
           if s.status != StatusUnlocked {
               return nil // idempotent — locking a locked vault is a no-op
           }
           if s.dek != nil {
               crypto.Zero(s.dek)
               s.dek = nil
           }
           s.unlockedAt = time.Time{}
           s.status = StatusLocked
           return nil
       }

       // RequireUnlocked is the single guard called by every secret-touching service method.
       // Returns *apperr.AppError{CodeVaultLocked} if not unlocked.
       func (s *Session) RequireUnlocked() error {
           s.mu.Lock()
           defer s.mu.Unlock()
           if s.status != StatusUnlocked {
               return apperr.New(apperr.CodeVaultLocked, "Vault is locked. Unlock to continue.")
           }
           return nil
       }

       // DEK returns a DEFENSIVE COPY of the current DEK. Returns nil if not unlocked.
       // The returned slice is caller-owned; caller MUST zero it after use.
       func (s *Session) DEK() []byte {
           s.mu.Lock()
           defer s.mu.Unlock()
           if s.status != StatusUnlocked || s.dek == nil { return nil }
           cp := make([]byte, len(s.dek))
           copy(cp, s.dek)
           return cp
       }

       func (s *Session) Status() Status {
           s.mu.Lock()
           defer s.mu.Unlock()
           return s.status
       }

       func (s *Session) VaultID() string {
           s.mu.Lock()
           defer s.mu.Unlock()
           return s.vaultID
       }

       func (s *Session) VaultPath() string {
           s.mu.Lock()
           defer s.mu.Unlock()
           return s.vaultPath
       }
       ```
       (Replace `<MODULE>` with the actual module name from go.mod, e.g. `weldon0405/abyss` or whatever was set in plan 01.)

    6. **`internal/session/session_test.go`** — tests:
       - `TestNew_StatusClosed`: New().Status() == StatusClosed.
       - `TestActivate_TransitionsToUnlocked`: SetLockedFor + Activate(32-byte-dek, ...) → Status() == StatusUnlocked, RequireUnlocked() == nil.
       - `TestActivate_RejectsShortDEK`: Activate([]byte{1,2,3}, ...) → returns AppError{CodeInternalError}.
       - `TestActivate_AlreadyUnlocked_ReturnsVaultAlreadyUnlocked`: Activate twice → second returns AppError{CodeVaultAlreadyUnlocked}.
       - `TestRequireUnlocked_WhenLocked_ReturnsVaultLocked`: New(), call RequireUnlocked() → AppError{CodeVaultLocked}.
       - `TestLock_ZerosDEK`: Activate with a known DEK ([]byte{1,2,3,...,32}). Save a reference to the internal slice via DEK() copy + a runtime trick OR via a test-only helper exported under build tag. **Practical approach:** call `Activate(dek)` where dek is a buffer the test owns; after Lock, assert that the test-owned buffer is now all zeros (because session took ownership and Lock called Zero on it).
         ```go
         dek := []byte{...32 bytes of 0xAB}
         s := New(); s.SetLockedFor("v","p")
         _ = s.Activate(dek, "v","p")
         _ = s.Lock()
         for i, b := range dek {
             if b != 0 { t.Fatalf("byte %d not zeroed: 0x%X", i, b) }
         }
         ```
       - `TestLock_TransitionsToLocked`: after Lock, Status() == StatusLocked, RequireUnlocked() returns VAULT_LOCKED.
       - `TestLock_Idempotent`: Lock twice → second returns nil with no panic.
       - `TestDEK_ReturnsNilWhenLocked`: New().DEK() == nil.
       - `TestDEK_ReturnsCopyNotInternalSlice`: Activate, mutate the returned DEK, call DEK() again, assert second copy is unchanged (i.e., we got a copy, not the internal slice).
       - `TestConcurrent_Activate_Lock_RequireUnlocked`: 100 goroutines each calling Activate+Lock+RequireUnlocked in a loop; race detector must pass (`-race` enforces).

    7. **`internal/logger/logger.go`** — slog wrapper with redaction:
       ```go
       package logger

       import (
           "context"
           "log/slog"
           "os"
           "strings"
       )

       // DefaultRedactedKeys is the canonical sensitive-key list per PRD §12.2.
       var DefaultRedactedKeys = []string{
           "password", "master_password", "kek", "dek", "secret", "value",
           "private_key", "api_key", "key_material", "clipboard",
       }

       const RedactedPlaceholder = "<redacted>"

       type redactingHandler struct {
           inner   slog.Handler
           keys    []string // lowercased
       }

       func (h *redactingHandler) Enabled(ctx context.Context, lvl slog.Level) bool {
           return h.inner.Enabled(ctx, lvl)
       }

       func (h *redactingHandler) Handle(ctx context.Context, r slog.Record) error {
           // Walk attrs and redact any whose key matches (case-insensitive substring).
           newRec := slog.NewRecord(r.Time, r.Level, r.Message, r.PC)
           r.Attrs(func(a slog.Attr) bool {
               if h.shouldRedact(a.Key) {
                   newRec.AddAttrs(slog.String(a.Key, RedactedPlaceholder))
               } else if a.Value.Kind() == slog.KindGroup {
                   newRec.AddAttrs(h.redactGroup(a))
               } else {
                   newRec.AddAttrs(a)
               }
               return true
           })
           return h.inner.Handle(ctx, newRec)
       }

       func (h *redactingHandler) shouldRedact(key string) bool {
           low := strings.ToLower(key)
           for _, k := range h.keys {
               if strings.Contains(low, k) { return true }
           }
           return false
       }

       func (h *redactingHandler) redactGroup(a slog.Attr) slog.Attr {
           inner := a.Value.Group()
           out := make([]slog.Attr, 0, len(inner))
           for _, sub := range inner {
               if h.shouldRedact(sub.Key) {
                   out = append(out, slog.String(sub.Key, RedactedPlaceholder))
               } else {
                   out = append(out, sub)
               }
           }
           return slog.Group(a.Key, anySliceFromAttrs(out)...)
       }

       func (h *redactingHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
           // Redact at attach time too.
           red := make([]slog.Attr, 0, len(attrs))
           for _, a := range attrs {
               if h.shouldRedact(a.Key) {
                   red = append(red, slog.String(a.Key, RedactedPlaceholder))
               } else {
                   red = append(red, a)
               }
           }
           return &redactingHandler{inner: h.inner.WithAttrs(red), keys: h.keys}
       }

       func (h *redactingHandler) WithGroup(name string) slog.Handler {
           return &redactingHandler{inner: h.inner.WithGroup(name), keys: h.keys}
       }

       func anySliceFromAttrs(attrs []slog.Attr) []any {
           out := make([]any, 0, len(attrs))
           for _, a := range attrs { out = append(out, a) }
           return out
       }

       // New returns a logger writing JSON to stderr with redaction enabled.
       func New() *slog.Logger {
           inner := slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelInfo})
           h := &redactingHandler{inner: inner, keys: lowercaseAll(DefaultRedactedKeys)}
           return slog.New(h)
       }

       func lowercaseAll(in []string) []string {
           out := make([]string, len(in))
           for i, s := range in { out[i] = strings.ToLower(s) }
           return out
       }
       ```

    8. **`internal/logger/logger_test.go`** — tests:
       - `TestRedactingHandler_RedactsExactKey_password`: log with `slog.String("password", "hunter2")`, capture output → contains `"password":"<redacted>"` and DOES NOT contain `hunter2`.
       - Table test for every key in DefaultRedactedKeys: log a value with that key, assert redacted.
       - `TestRedactingHandler_CaseInsensitive`: keys `Password`, `PASSWORD`, `master_Password` all redact.
       - `TestRedactingHandler_SubstringMatch`: key `current_password` redacts (matches "password" substring).
       - `TestRedactingHandler_NonSensitiveKeysPass`: key `vault_id` is NOT redacted (no substring match).
       - `TestRedactingHandler_GroupAttrs`: log `slog.Group("session", slog.String("dek", "AAAAAA"), slog.String("vault_id", "abc"))` → dek redacted, vault_id passes through.
       - `TestRedactingHandler_NeverLeaksMasterPassword_FuzzString`: 1000 random masterPassword strings, log each at various keys (masterPassword, master_password, MASTER_PASSWORD), assert none of the random strings appear in output.

    9. Commit: `feat(01-02): apperr typed errors + session lock-state with DEK zero + logger redaction`.
  </action>
  <verify>
    <automated>cd /c/Users/weldo/Projects/abyss &amp;&amp; go test ./internal/apperr/... ./internal/session/... ./internal/logger/... -race -count=1</automated>
  </verify>
  <acceptance_criteria>
    - `internal/apperr/codes.go` declares all 12 Code constants matching PRD §10.1 exactly
    - `internal/apperr/codes.go` declares `var AllCodes` whose `len` is 12
    - `internal/apperr/error.go` has `ToDTO(err error) *DTO` that drops Cause
    - `go test ./internal/apperr -run TestToDTO_RawError_DropsRawMessage -count=1` exits 0
    - `go test ./internal/apperr -run TestAllCodes_Length12 -count=1` exits 0
    - `internal/session/session.go` exports `Session`, `New`, `Activate`, `Lock`, `RequireUnlocked`, `Status`, `DEK`, `VaultID`, `VaultPath`, `SetLockedFor`
    - `internal/session/session.go` imports `internal/crypto` and calls `crypto.Zero`
    - `grep -c 'crypto.Zero' internal/session/session.go` returns &gt;= 1
    - `go test ./internal/session -run TestLock_ZerosDEK -count=1` exits 0
    - `go test ./internal/session -run TestRequireUnlocked_WhenLocked_ReturnsVaultLocked -count=1` exits 0
    - `go test ./internal/session -run TestConcurrent_Activate_Lock_RequireUnlocked -race -count=1` exits 0
    - `internal/logger/logger.go` exports `New`, `DefaultRedactedKeys`, `RedactedPlaceholder`
    - `DefaultRedactedKeys` contains AT MINIMUM: password, master_password, kek, dek, secret, value, private_key, api_key, key_material, clipboard (verify via test)
    - `go test ./internal/logger -run TestRedactingHandler_NeverLeaksMasterPassword_FuzzString -count=1` exits 0
    - `golangci-lint run ./...` exits 0 (forbidigo still happy; no math/rand)
  </acceptance_criteria>
  <done>Typed-error system ready for the Wails boundary in plan 03; session lock-state machine zeros DEK atomically on Lock; logger redaction proven against PRD §12.2 sensitive-key list; all D-02 CI gates from this plan are green; the FIRST cipher.Seal in this codebase will pass through aad.Build (proven by aead_aad_test.go).</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| internal/apperr → Wails boundary | `ToDTO` is the SOLE sanitization point. Any leak of Cause across this boundary discloses internal crypto/storage error context. |
| Master password (string) → KEK derivation | Go strings are immutable. Mitigation per PRD §4.4: convert to []byte ASAP, KDF returns derivable []byte that caller zeros. |
| AAD construction call sites → cipher.Seal | If any caller constructs AAD inline rather than via aad.Build, future records become permanently unreadable. CI grep gate enforces. |
| math/rand presence in dep tree | A single `import "math/rand"` in production code (or a transitive dep that exposes it via reflection) defeats SEC-05. forbidigo lint catches direct imports; CI grep on go.sum + go.mod catches transitive. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-02-01 | Tampering | AAD construction across the codebase | mitigate | Single `internal/aad.Build()` helper; CI grep gates (`grep -rn '|metadata"' internal/ --include="*.go" | grep -v 'aad/aad.go'` returns no matches); golden-vector tests lock the byte format. PRD §5.4 + §18 hard rule #15. |
| T-02-02 | Tampering | AAD includes schema_version (would force re-encryption on every migration) | mitigate | aad.Build signature does NOT accept schema_version; test `TestBuild_NoSchemaVersionInOutput` asserts the substring is absent from all 3 output kinds. PRD §5.4 + §18 hard rule #15. |
| T-02-03 | Information Disclosure | DEK in memory beyond Lock | mitigate | `session.Lock()` calls `crypto.Zero(s.dek)` BEFORE `s.dek = nil` under mutex; test `TestLock_ZerosDEK` asserts test-owned buffer is all zeros after Lock. PRD §VAULT-10 + §4.4. |
| T-02-04 | Information Disclosure | DEK leaked via session.DEK() returning internal slice | mitigate | `session.DEK()` returns a defensive copy; mutating the returned slice does not affect the internal state; test `TestDEK_ReturnsCopyNotInternalSlice` enforces. |
| T-02-05 | Spoofing | math/rand crept in via direct import or transitive dep | mitigate | `forbidigo` lint banning `math/rand` and `math/rand/v2`; CI runs `golangci-lint`; manual `grep -rn 'math/rand' internal/ --include="*.go"` returns no matches. PRD §18 hard rule #13. |
| T-02-06 | Information Disclosure | Master password / KEK / DEK / plaintext records leaked into logs | mitigate | `internal/logger` redactingHandler matches sensitive keys substring-case-insensitively; fuzz test `TestRedactingHandler_NeverLeaksMasterPassword_FuzzString` proves random masterPassword strings never appear in output. PRD §12.2 + §18 hard rule #7. |
| T-02-07 | Information Disclosure | Internal Cause leaks across Wails boundary | mitigate | `apperr.ToDTO` strips Cause; serialization test `TestToDTO_AppError_DropsCause` verifies no fragment of cause.Error() appears in marshaled DTO. Raw errors map to generic `INTERNAL_ERROR` with hardcoded "Something went wrong. Try again." PRD §12.1. |
| T-02-08 | Tampering | Argon2id parameters hardcoded globally would prevent fallback profile selection per-wrapping | mitigate | `Argon2idParams` struct stored per-wrapping in `kdf_params_json`; PRD §5.3 mandates per-wrapping not global; test `TestPRDProfiles_StoredCorrectly` asserts the three named profiles match PRD verbatim. |
| T-02-09 | Spoofing | UnwrapDEK returns distinguishable errors (wrong-password vs corrupt-blob vs AAD-mismatch) — could be used as oracle | mitigate | Both `Decrypt` and `UnwrapDEK` collapse all failure modes to single `ErrAuthFailed`; tests `TestUnwrap_WrongKEK_ReturnsErrAuthFailed`, `TestUnwrap_AADMismatch_ReturnsErrAuthFailed`, `TestUnwrap_TamperedCiphertext_ReturnsErrAuthFailed` enforce. PRD §9.4 generic-failure rule. |
| T-02-10 | Information Disclosure | AAD field separator `|` injection via crafted input | mitigate | `aad.Build` rejects any input containing `|` with a non-nil error; test `TestBuild_ContainsPipe_ReturnsError` enforces. UUIDs (used everywhere in our schema) are pipe-free by design. |
</threat_model>

<verification>
**End-of-plan checks:**

1. `go test ./internal/aad/... ./internal/crypto/... ./internal/apperr/... ./internal/session/... ./internal/logger/... -race -count=1` exits 0
2. `golangci-lint run ./...` exits 0 with `forbidigo` enabled
3. `grep -rn 'math/rand' . --include="*.go"` returns no matches outside `vendor/` (none in test code either — D-02 strict)
4. `grep -rn 'schema_version' internal/aad/ internal/crypto/` returns no matches in source code
5. AAD construction inline-grep: `grep -rEn '\[\]byte\("[^"]*\|(metadata|secret|wrapped_dek)"\)' internal/ --include="*.go" | grep -v 'aad/aad.go' | grep -v '_test.go'` returns no matches
6. `gh run list --workflow=ci.yml --limit 1 --json conclusion` returns success on all 3 OSes
7. **D-05 cross-OS smoke (manual):** on Windows, `git pull`, `go test ./internal/...` exits 0. The crypto+session+apperr+logger packages have no platform-specific code, so this should be trivially green; sign off in 01-VERIFICATION.md.
</verification>

<success_criteria>
- `internal/aad.Build()` is the SINGLE source of AAD bytes for the entire codebase; golden vectors lock the format; the function does not accept a `schema_version` parameter
- AES-256-GCM `Encrypt`/`Decrypt` reject nil/empty AAD; round-trip test with AAD via `aad.Build` passes; AAD-mismatch test fails authentication
- Argon2id `DeriveKEK` runs with all three PRD §5.3 profiles; `DefaultArgon2idParams` matches 256MiB/3/4/16/32 verbatim; per-wrapping `Argon2idParams` struct is JSON-serializable
- `GenerateDEK` returns 32 random bytes from `crypto/rand`; `WrapDEK`/`UnwrapDEK` round-trip via the wrapped-DEK AAD
- `apperr` declares all 12 PRD §10.1 codes; `ToDTO` strips Cause; raw errors map to `INTERNAL_ERROR` with generic message
- `session.Lock()` zeros the DEK with `crypto.Zero` before nilling under mutex; `RequireUnlocked()` returns `CodeVaultLocked` AppError when not unlocked
- `logger.New()` returns an slog.Logger that redacts all keys in `DefaultRedactedKeys` substring-case-insensitively; fuzz test proves master password strings never appear in output
- `forbidigo` rule banning `math/rand` is in `.golangci.yml` and `golangci-lint run` exits 0; no direct or test-code use of `math/rand` exists
- D-05 Windows smoke for plan 02 signed off in `01-VERIFICATION.md`
</success_criteria>

<output>
After completion, create `.planning/phases/01-v0-1-dogfoodable-vault/01-02-SUMMARY.md` documenting:
- Confirmed: AAD format strings are byte-locked by golden vectors at `internal/aad/testdata/golden_*.txt`. Any future change to the AAD format will break these tests, by design.
- Confirmed: the FIRST `cipher.Seal` invocation in this codebase (in `internal/crypto/aead.go::Encrypt`) requires a non-empty `aad []byte` parameter. The aad_aad_test.go suite proves the call path uses `aad.Build`.
- Confirmed: `forbidigo` rule active and tested (manually injected a `math/rand` import into a temp file, observed lint failure, removed). D-02 plan-2 gate landed.
- Cross-OS smoke for plan 02: result + commit SHA.
- Open question for plan 03: confirm Wails ctx is wired into the App struct so `runtime.EventsEmit` (for `vault:locked` event per UI-SPEC interaction contract #5) is available to vault.Service via injection.
</output>
