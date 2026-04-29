# PRD / Build Contract: Local Password & Key Vault

**Version:** 1.2 — Architecture-Locked MVP Build Contract
**Project type:** Local-first desktop password and cryptographic key vault
**Primary implementation stack:** Go + Wails v2
**Frontend stack:** Vite + React + TypeScript + Mantine
**Storage:** SQLite with record-level authenticated encryption
**Target OS:** macOS + Windows
**Primary purpose:** Learn Go + Wails by building a real, functional, local-first password and cryptographic key vault.

---

# 1. Executive Summary

Build a local-first desktop application that allows a user to create and unlock an encrypted vault, store credentials, generate strong passwords, generate cryptographic keys, and safely copy/export selected secrets.

The app will be built with **Go + Wails v2** and a **Vite + React + TypeScript + Mantine** frontend. Data will be stored in **SQLite**, with sensitive record contents encrypted using a **DEK/KEK envelope encryption model**.

The MVP is intentionally not a commercial password-manager competitor. It is a **learning-grade but functionally useful local vault** for passwords, API keys, secure notes, AES keys, RSA key pairs, and Ed25519 key pairs.

---

# 2. Locked Architecture Decisions

## 2.1 Desktop Framework

Use:

```text
Go + Wails v2
```

Wails is the core learning target. Use the stable v2 line for MVP. Do not use Wails v3 alpha/pre-release unless explicitly approved later.

---

## 2.2 Frontend Stack

Use:

```text
Vite + React + TypeScript + Mantine
```

Do **not** use Next.js for this project.

### Rationale

Inside a Wails desktop app, the frontend is a static embedded app. Next.js server features such as SSR, API routes, server actions, server components, and runtime Node behavior are not needed and create unnecessary build complexity.

Vite + React + TypeScript + Mantine is the preferred stack because it gives the app a fast SPA frontend with less framework overhead.

---

## 2.3 Target Operating Systems

The MVP targets:

```text
macOS
Windows
```

Linux is out of scope for MVP.

### Cross-Platform Rule

The product target is macOS + Windows from the beginning, but early development should proceed in staged increments:

* Develop features on the active development machine.
* Build/test on the second OS at milestone boundaries.
* Do not let packaging issues block early feature progress.
* Do not defer all cross-platform testing until the end.

---

## 2.4 Storage

Use:

```text
SQLite
Record-level encryption
Rollback journal mode for MVP
```

Do not use SQLCipher for MVP.

Do not use WAL mode for MVP unless explicitly approved.

### WAL Decision

Use rollback journal mode because vault portability matters more than write throughput. Users may copy, zip, move, back up, or restore a vault file. WAL mode can be safe, but it requires checkpointing and careful handling of sidecar state. Rollback journal mode is the safer MVP default for a portable local vault file.

---

## 2.5 Migration System

Use:

```text
golang-migrate
```

Migration files shall be embedded into the Go binary.

Migrations must be:

* Versioned.
* Repeatable across macOS and Windows.
* Fail-closed on dirty migration state.
* Applied before opening the vault for normal operations.
* Covered by tests where practical.

Do not hand-roll a migration framework unless `golang-migrate` creates Wails packaging problems.

---

## 2.6 Encryption Model

Use:

```text
DEK/KEK envelope encryption
```

Where:

* **DEK** = Data Encryption Key, randomly generated.
* **KEK** = Key Encryption Key, derived from the master password using Argon2id.
* Records are encrypted with the DEK.
* The DEK is wrapped/encrypted by the KEK.
* The wrapped DEK acts as the password verifier.

---

## 2.7 Clipboard Behavior

Default behavior:

```text
Copy secret → show countdown → attempt clear after 30 seconds
```

Clipboard timeout must be configurable.

Supported timeout settings:

```text
10 seconds
30 seconds
60 seconds
120 seconds
```

Default:

```text
30 seconds
```

The frontend owns the countdown/progress bar. The Go backend owns the clipboard write/clear operations.

---

## 2.8 Private Key Export

Private keys may be copied/exported only when:

1. Vault is unlocked.
2. User explicitly chooses copy/export.
3. App displays a private-key exposure warning.
4. User confirms the action.

Private key export is a Must-have for MVP.

---

## 2.9 Public Key Storage and UX

Public keys are not secret material and may be stored in plaintext columns.

However, for MVP simplicity:

```text
Locked vault = no record interaction
Unlocked vault = record interaction allowed
```

Therefore, MVP may still require unlock before viewing/copying public keys. This is a UX simplification, not a security requirement. A future version may expose a locked-state public key ring.

---

# 3. Product Goals

## 3.1 Primary Goals

1. Build a working desktop app using Go + Wails.
2. Learn desktop app architecture with a Go backend and React frontend.
3. Create a local encrypted SQLite vault.
4. Store password credentials.
5. Generate strong random passwords.
6. Store API keys.
7. Store secure notes.
8. Generate and store AES keys.
9. Generate and store RSA key pairs.
10. Generate and store Ed25519 key pairs.
11. Copy secrets with a visible clipboard countdown.
12. Export public and private key material safely.
13. Support macOS and Windows.

---

## 3.2 Learning Goals

The project should provide practical experience with:

* Wails method bindings.
* Go service-layer architecture.
* SQLite persistence.
* Schema migrations.
* Authenticated encryption.
* Argon2id key derivation.
* DEK/KEK envelope encryption.
* Desktop clipboard handling.
* Cross-platform desktop packaging.
* React/Mantine UI development.
* Error boundary design between frontend and backend.

---

## 3.3 Non-Goals

Do not build these in MVP:

* Cloud sync.
* Browser extension.
* Browser autofill.
* Mobile app.
* Team vaults.
* Shared vaults.
* Multi-user accounts.
* Biometric unlock.
* Hardware security key unlock.
* TPM/Secure Enclave integration.
* Password breach monitoring.
* TOTP authenticator.
* SSH agent integration.
* PGP key management.
* Auto-update.
* Enterprise audit logs.
* Plugin system.
* Linux packaging.
* Formal security audit.

---

# 4. Security Positioning

## 4.1 Product Security Statement

This app is a **local-first learning-grade password and cryptographic key vault**. It should use modern cryptographic practices and avoid unsafe patterns, but it must not be marketed as an audited enterprise-grade password manager.

---

## 4.2 Threat Model

The app should protect against:

* Casual inspection of the vault file.
* Offline theft of the SQLite vault without the master password.
* Accidental plaintext persistence of secrets.
* Basic shoulder-surfing via masked secret fields.
* Accidental long-lived clipboard exposure.
* Record-swapping/tampering inside the SQLite file where AEAD AAD can detect it.

The app does **not** protect against:

* Malware on the user’s machine.
* Keyloggers.
* Screen capture.
* Memory scraping by malware.
* Compromised operating system.
* Malicious builds.
* Weak master passwords.
* User exporting private keys to unsafe locations.
* User copying secrets into unsafe apps.
* Third-party clipboard managers that preserve clipboard history.

---

## 4.3 DEK Memory Exposure

In the envelope encryption model:

* The master password is used briefly to derive a KEK.
* The KEK unwraps the DEK.
* The DEK remains in memory while the vault is unlocked.
* Decrypted record data may also exist in memory while the vault is unlocked.

Therefore, the unlocked session is the primary memory-exposure window.

Idle auto-lock is a Must-have because it reduces how long the DEK and decrypted records remain available in memory. This does not protect against malware or a compromised OS, but it reduces exposure from unattended sessions and casual local access.

---

## 4.4 Go Memory Handling Limitations

The README/security documentation must include this statement:

```text
This app is written in Go. Go does not provide perfect control over sensitive memory.

Strings are immutable and cannot be reliably zeroed. The Wails binding layer may pass sensitive input as strings before the backend copies it into byte slices. The app will zero byte slices where practical, avoid unnecessary copies, clear session state on lock, and avoid logging sensitive values, but it cannot guarantee complete memory erasure.
```

---

# 5. Cryptographic Design

## 5.1 Required Algorithms

| Use Case               | Algorithm                          | MVP Priority |
| ---------------------- | ---------------------------------- | -----------: |
| Record encryption      | AES-256-GCM                        |         Must |
| Master password KDF    | Argon2id                           |         Must |
| Password generation    | CSPRNG via Go `crypto/rand`        |         Must |
| AES key generation     | AES-128, AES-192, AES-256 raw keys |         Must |
| RSA key generation     | 2048, 3072, 4096-bit               |         Must |
| Ed25519 key generation | Ed25519                            |         Must |
| ECDSA                  | Defer                              | Out of scope |
| X25519                 | Defer                              | Out of scope |
| TOTP secrets           | Defer                              | Out of scope |

---

## 5.2 Envelope Encryption

### Vault Creation

```text
DEK = crypto/rand 32 bytes
KEK = Argon2id(master_password, salt)
wrapped_DEK = AES-256-GCM-Encrypt(KEK, DEK, AAD)
```

Store:

* Vault metadata.
* DEK metadata.
* Key wrapping row with:

	* KDF parameters.
	* Salt.
	* Wrapped DEK.
	* Wrapped DEK nonce.

Do not store:

* Master password.
* KEK.
* Plaintext DEK.
* Plaintext records.

---

### Vault Unlock

```text
KEK = Argon2id(master_password, salt)
DEK = AES-256-GCM-Decrypt(KEK, wrapped_DEK, AAD)
```

If unwrap fails, return a generic unlock failure.

---

### Master Password Change

Changing the master password is a Must-have.

Process:

```text
1. User enters current master password.
2. App unlocks/verifies current wrapping.
3. User enters new master password and confirmation.
4. App derives new KEK with new salt/KDF params.
5. App re-wraps the same DEK.
6. App replaces the old master_password key_wrapping row.
7. Records do not need to be re-encrypted.
```

---

## 5.3 KDF Parameters

Use Argon2id.

Initial target profile:

```text
memory:      256 MiB
iterations:  3
parallelism: 4
key length:  32 bytes
salt:        16+ random bytes
```

Fallback profile:

```text
memory:      64 MiB
iterations:  3
parallelism: 1
key length:  32 bytes
salt:        16+ random bytes
```

Minimum acceptable profile:

```text
memory:      19 MiB
iterations:  2
parallelism: 1
key length:  32 bytes
salt:        16+ random bytes
```

The app should benchmark/tune unlock time on the slowest supported dev machine.

Store KDF parameters per key wrapping row, not globally at the vault level.

---

## 5.4 AES-GCM AAD Rules

All AES-GCM encryption/decryption operations must use Additional Authenticated Data.

### Record Metadata Blob AAD

```text
AAD = vault_id || record_envelope_version || record_id || record_type || dek_id || "metadata"
```

### Record Secret Blob AAD

```text
AAD = vault_id || record_envelope_version || record_id || record_type || dek_id || "secret"
```

### Wrapped DEK AAD

```text
AAD = vault_id || dek_id || wrapping_type || "wrapped_dek"
```

Do not use SQLite `schema_version` in AAD. Schema migrations must not force record re-encryption unless the record envelope itself changes.

---

## 5.5 Nonce Rules

* Generate a unique nonce for every AES-GCM encryption.
* Never reuse a nonce with the same key.
* Store nonces next to ciphertext.
* Use CSPRNG for nonce generation.
* Tests must verify nonce helper behavior.

---

# 6. SQLite Storage Design

## 6.1 SQLite File Identification

At vault creation:

```sql
PRAGMA application_id = 0x4E56544E;
```

`0x4E56544E` represents a project-specific application ID.

The app must check SQLite `application_id` before attempting unlock. Do not rely only on file extension.

---

## 6.2 Journal Mode

MVP shall use rollback journal mode.

Suggested initialization:

```sql
PRAGMA journal_mode = DELETE;
PRAGMA foreign_keys = ON;
```

Do not use WAL mode in MVP.

---

## 6.3 Core Tables

### `vault_metadata`

```sql
CREATE TABLE vault_metadata (
  id TEXT PRIMARY KEY CHECK (id = 'vault'),
  vault_id TEXT NOT NULL UNIQUE,
  schema_version INTEGER NOT NULL,
  record_envelope_version INTEGER NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

Notes:

* Single-row table.
* `vault_id` is generated once at vault creation.
* `schema_version` tracks database migration version.
* `record_envelope_version` tracks encrypted record/AAD envelope version.
* Do not include undefined `vault_version`.

---

### `deks`

```sql
CREATE TABLE deks (
  id TEXT PRIMARY KEY,
  vault_id TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('active', 'retired')),
  created_at TEXT NOT NULL,
  retired_at TEXT
);
```

Notes:

* Stores DEK identity and lifecycle metadata.
* Does not store plaintext DEK.
* MVP uses one active DEK.
* Future DEK rotation can use this table.

---

### `key_wrappings`

```sql
CREATE TABLE key_wrappings (
  id TEXT PRIMARY KEY,
  vault_id TEXT NOT NULL,
  dek_id TEXT NOT NULL,
  wrapping_type TEXT NOT NULL CHECK (wrapping_type IN ('master_password')),
  kdf_algorithm TEXT NOT NULL,
  kdf_params_json TEXT NOT NULL,
  salt BLOB NOT NULL,
  wrapped_dek_nonce BLOB NOT NULL,
  wrapped_dek BLOB NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (dek_id) REFERENCES deks(id)
);
```

Notes:

* MVP has one row with `wrapping_type = 'master_password'`.
* Salt is stored per wrapping.
* Future unlock methods can add additional wrapping rows.

---

### `records`

```sql
CREATE TABLE records (
  id TEXT PRIMARY KEY,
  vault_id TEXT NOT NULL,
  dek_id TEXT NOT NULL,
  record_type TEXT NOT NULL,
  public_key_openssh TEXT,
  public_key_pem TEXT,
  public_key_fingerprint TEXT,
  encrypted_metadata_blob BLOB NOT NULL,
  metadata_nonce BLOB NOT NULL,
  encrypted_secret_blob BLOB NOT NULL,
  secret_nonce BLOB NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (dek_id) REFERENCES deks(id)
);
```

Notes:

* Public key fields are nullable.
* Public key fields are only used for asymmetric key records.
* Sensitive fields must be encrypted.
* Titles, notes, tags, usernames, URLs, API keys, passwords, AES keys, and private keys must not be stored in plaintext.

---

### `settings`

```sql
CREATE TABLE settings (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

Allowed settings:

* Clipboard timeout.
* Idle auto-lock timeout.
* Theme preference.
* Last selected record type filter.
* Other non-sensitive UI preferences.

Do not store sensitive data in `settings`.

---

# 7. Record Types

The app must support these record types by MVP completion:

```text
credential
api_key
secure_note
symmetric_key
asymmetric_key_pair
```

---

## 7.1 Credential Record

Encrypted metadata blob:

```json
{
  "title": "GitHub",
  "username": "user@example.com",
  "url": "https://github.com",
  "notes": "",
  "tags": ["dev"]
}
```

Encrypted secret blob:

```json
{
  "password": "plaintext-password-before-encryption"
}
```

---

## 7.2 API Key Record

Encrypted metadata blob:

```json
{
  "title": "OpenAI API Key",
  "provider": "OpenAI",
  "environment": "development",
  "expiration_date": null,
  "notes": "",
  "tags": ["dev", "api"]
}
```

Encrypted secret blob:

```json
{
  "api_key": "secret-token"
}
```

---

## 7.3 Secure Note Record

Encrypted metadata blob:

```json
{
  "title": "Router Recovery Info",
  "tags": ["home"]
}
```

Encrypted secret blob:

```json
{
  "body": "Sensitive note contents"
}
```

---

## 7.4 Symmetric Key Record

Encrypted metadata blob:

```json
{
  "title": "Local AES Key",
  "algorithm": "AES",
  "key_length": 256,
  "encoding_display_preference": "base64",
  "notes": "",
  "tags": ["crypto"]
}
```

Encrypted secret blob:

```json
{
  "key_material_base64": "...",
  "key_material_hex": "..."
}
```

---

## 7.5 Asymmetric Key Pair Record

Plaintext columns:

```text
public_key_openssh
public_key_pem
public_key_fingerprint
```

Encrypted metadata blob:

```json
{
  "title": "GitHub Deploy Key",
  "algorithm": "Ed25519",
  "key_length": null,
  "format": "pkcs8_pem",
  "notes": "",
  "tags": ["ssh", "github"]
}
```

Encrypted secret blob:

```json
{
  "private_key_pem": "...",
  "private_key_openssh": null
}
```

---

# 8. Functional Requirements

## 8.1 Vault Lifecycle

| ID     | Requirement                                     | Priority |
| ------ | ----------------------------------------------- | -------: |
| FR-001 | Create a new SQLite vault.                      |     Must |
| FR-002 | Initialize schema with migrations.              |     Must |
| FR-003 | Set and validate SQLite `application_id`.       |     Must |
| FR-004 | Generate `vault_id` at vault creation.          |     Must |
| FR-005 | Generate active DEK at vault creation.          |     Must |
| FR-006 | Wrap DEK using master-password-derived KEK.     |     Must |
| FR-007 | Unlock vault using master password.             |     Must |
| FR-008 | Lock vault manually.                            |     Must |
| FR-009 | Auto-lock vault after idle timeout.             |     Must |
| FR-010 | Change master password by re-wrapping DEK.      |     Must |
| FR-011 | Clear decrypted frontend/backend state on lock. |     Must |
| FR-012 | Attempt clipboard clear on lock.                |     Must |
| FR-013 | Reject secret actions while locked.             |     Must |

---

## 8.2 Record Management

| ID     | Requirement                            | Priority |
| ------ | -------------------------------------- | -------: |
| FR-014 | Create credential records.             |     Must |
| FR-015 | Create API key records.                |     Must |
| FR-016 | Create secure note records.            |     Must |
| FR-017 | Create symmetric key records.          |     Must |
| FR-018 | Create asymmetric key pair records.    |     Must |
| FR-019 | View records after unlock.             |     Must |
| FR-020 | Edit records.                          |     Must |
| FR-021 | Delete records after confirmation.     |     Must |
| FR-022 | Search records in memory after unlock. |     Must |
| FR-023 | Mask secret values by default.         |     Must |
| FR-024 | Temporarily reveal secrets.            |     Must |
| FR-025 | Copy selected fields.                  |     Must |
| FR-026 | Store tags encrypted.                  |   Should |

---

## 8.3 Password Generator

| ID     | Requirement                                       | Priority |
| ------ | ------------------------------------------------- | -------: |
| FR-027 | Generate passwords using `crypto/rand`.           |     Must |
| FR-028 | Allow length configuration.                       |     Must |
| FR-029 | Allow uppercase/lowercase/number/symbol toggles.  |     Must |
| FR-030 | Allow excluding ambiguous characters.             |   Should |
| FR-031 | Copy generated password.                          |     Must |
| FR-032 | Save generated password as credential.            |     Must |
| FR-033 | Show strength estimate using vetted library only. |   Should |

Password generator defaults:

```text
length: 24
uppercase: enabled
lowercase: enabled
numbers: enabled
symbols: enabled
exclude ambiguous characters: enabled
```

Do not build a custom password strength estimator. If included, use a vetted zxcvbn-style library.

---

## 8.4 AES Key Generation

| ID     | Requirement                            | Priority |
| ------ | -------------------------------------- | -------: |
| FR-034 | Generate AES-128 keys.                 |     Must |
| FR-035 | Generate AES-192 keys.                 |     Must |
| FR-036 | Generate AES-256 keys.                 |     Must |
| FR-037 | Display generated key as Base64.       |     Must |
| FR-038 | Display generated key as hex.          |     Must |
| FR-039 | Save generated AES key to vault.       |     Must |
| FR-040 | Copy generated AES key while unlocked. |     Must |

---

## 8.5 RSA Key Generation

| ID     | Requirement                              | Priority |
| ------ | ---------------------------------------- | -------: |
| FR-041 | Generate 2048-bit RSA key pairs.         |     Must |
| FR-042 | Generate 3072-bit RSA key pairs.         |     Must |
| FR-043 | Generate 4096-bit RSA key pairs.         |     Must |
| FR-044 | Export public key in OpenSSH format.     |     Must |
| FR-045 | Export public key in PEM format.         |   Should |
| FR-046 | Export private key in PKCS#8 PEM format. |     Must |
| FR-047 | Warn before private key copy/export.     |     Must |
| FR-048 | Save generated RSA key pair to vault.    |     Must |

Default RSA size:

```text
3072-bit
```

---

## 8.6 Ed25519 Key Generation

| ID     | Requirement                               | Priority |
| ------ | ----------------------------------------- | -------: |
| FR-049 | Generate Ed25519 key pairs.               |     Must |
| FR-050 | Label Ed25519 as signing key algorithm.   |     Must |
| FR-051 | Export public key in OpenSSH format.      |     Must |
| FR-052 | Export public key in PEM format.          |   Should |
| FR-053 | Export private key in PKCS#8 PEM format.  |     Must |
| FR-054 | Export private key in OpenSSH format.     |   Should |
| FR-055 | Warn before private key copy/export.      |     Must |
| FR-056 | Save generated Ed25519 key pair to vault. |     Must |

---

## 8.7 Clipboard

| ID     | Requirement                                       | Priority |
| ------ | ------------------------------------------------- | -------: |
| FR-057 | Copy selected secrets to clipboard.               |     Must |
| FR-058 | Display countdown progress bar.                   |     Must |
| FR-059 | Attempt clipboard clear after configured timeout. |     Must |
| FR-060 | Allow manual clipboard clear.                     |   Should |
| FR-061 | Reset countdown when a new secret is copied.      |     Must |
| FR-062 | Attempt clipboard clear when vault locks.         |     Must |
| FR-063 | Document clipboard manager limitations.           |     Must |
| FR-064 | Optional harmless overwrite before clearing.      |    Could |

---

# 9. UI Requirements

## 9.1 Primary Screens

MVP screens:

1. Welcome / Vault Selection
2. Create Vault
3. Unlock Vault
4. Vault Home
5. Record Detail
6. New/Edit Record
7. Password Generator
8. Key Generator
9. Settings
10. About / Security Model

---

## 9.2 Welcome Screen

Required actions:

* Create new vault.
* Open existing vault.
* View security limitations.

Optional:

* Recent vaults.

Recent vaults are deferred unless implementation is trivial and does not store sensitive data.

---

## 9.3 Create Vault Screen

Required fields:

* Vault file path.
* Master password.
* Confirm master password.

Required warning:

```text
There is no password recovery. If you lose your master password, the vault cannot be recovered.
```

Validation:

* Minimum master password length: 12 characters.
* Password and confirmation must match.
* Passphrases must be allowed.
* Do not require arbitrary composition rules.

---

## 9.4 Unlock Screen

Required fields:

* Master password.

Required behavior:

* Unlock with correct password.
* Generic error on failure.
* Do not reveal whether failure came from password, database, wrapping, KDF, or decryption.

Error message:

```text
Unable to unlock vault. Check your master password and try again.
```

---

## 9.5 Vault Home

Required regions:

* Header with vault status.
* Lock button.
* Search input.
* Record type filter.
* Record list.
* New Record button.
* Generate Password button.
* Generate Key button.
* Settings button.

The UI must clearly show locked vs unlocked state.

---

## 9.6 Record Detail

Required behavior:

* Display metadata fields.
* Mask secret fields by default.
* Reveal button for secret fields.
* Copy button for allowed fields.
* Edit button.
* Delete button.
* Private key copy/export actions require warning modal.

---

## 9.7 Clipboard Countdown Component

After copying a secret, display:

```text
Secret copied. Clipboard will be cleared in 30 seconds.

[████████████████░░░░░░] 21s remaining

[Clear now]
```

Required behavior:

* Progress starts full.
* Progress decreases over configured timeout.
* Countdown updates at least once per second.
* Manual clear button attempts immediate clipboard clear.
* Locking vault attempts immediate clipboard clear.
* New copy action resets countdown.

---

## 9.8 Private Key Warning Modal

Title:

```text
Private Key Exposure Warning
```

Body:

```text
You are about to expose a private key.

Anyone with access to this private key may be able to impersonate you, decrypt data intended for you, or access systems that trust this key.

Only continue if you understand the risk and know where this private key will be stored.
```

Buttons:

* Cancel.
* I Understand, Continue.

For export:

```text
I Understand, Export Private Key
```

For copy:

```text
I Understand, Copy Private Key
```

---

# 10. Backend API / Wails Binding Contract

All security-sensitive operations must be implemented in Go.

The frontend must not directly access SQLite, cryptographic primitives, file system writes, or clipboard internals.

---

## 10.1 Common Error DTO

```ts
export type AppErrorCode =
  | "VAULT_LOCKED"
  | "VAULT_ALREADY_UNLOCKED"
  | "UNLOCK_FAILED"
  | "UNSUPPORTED_VAULT"
  | "CORRUPT_VAULT"
  | "MIGRATION_FAILED"
  | "VALIDATION_ERROR"
  | "RECORD_NOT_FOUND"
  | "EXPORT_FAILED"
  | "CLIPBOARD_FAILED"
  | "PLATFORM_ERROR"
  | "INTERNAL_ERROR";

export type AppErrorDTO = {
  code: AppErrorCode;
  message: string;
  details?: Record<string, string>;
};
```

Frontend must use `code`, not string matching, for error handling.

Backend must map internal errors to safe public errors.

---

## 10.2 Vault DTOs

```ts
export type CreateVaultRequest = {
  path: string;
  masterPassword: string;
  confirmMasterPassword: string;
};

export type OpenVaultRequest = {
  path: string;
};

export type UnlockVaultRequest = {
  path: string;
  masterPassword: string;
};

export type ChangeMasterPasswordRequest = {
  currentMasterPassword: string;
  newMasterPassword: string;
  confirmNewMasterPassword: string;
};

export type VaultStatusDTO = {
  isOpen: boolean;
  isUnlocked: boolean;
  vaultPath?: string;
  vaultId?: string;
  lockedAt?: string;
  unlockedAt?: string;
};

export type VaultSessionDTO = {
  vaultId: string;
  vaultPath: string;
  unlockedAt: string;
  autoLockTimeoutSeconds: number;
};
```

Required Wails methods:

```go
CreateVault(request CreateVaultRequest) (VaultSessionDTO, error)
OpenVault(request OpenVaultRequest) (VaultStatusDTO, error)
UnlockVault(request UnlockVaultRequest) (VaultSessionDTO, error)
LockVault() error
ChangeMasterPassword(request ChangeMasterPasswordRequest) error
GetVaultStatus() (VaultStatusDTO, error)
```

---

## 10.3 Record DTOs

```ts
export type RecordType =
  | "credential"
  | "api_key"
  | "secure_note"
  | "symmetric_key"
  | "asymmetric_key_pair";

export type RecordSummaryDTO = {
  id: string;
  type: RecordType;
  title: string;
  subtitle?: string;
  tags?: string[];
  publicKeyFingerprint?: string;
  createdAt: string;
  updatedAt: string;
};

export type RecordDTO = {
  id: string;
  type: RecordType;
  metadata: Record<string, unknown>;
  secret: Record<string, unknown>;
  publicKeyOpenSSH?: string;
  publicKeyPEM?: string;
  publicKeyFingerprint?: string;
  createdAt: string;
  updatedAt: string;
};

export type CreateRecordRequest = {
  type: RecordType;
  metadata: Record<string, unknown>;
  secret: Record<string, unknown>;
  publicKeyOpenSSH?: string;
  publicKeyPEM?: string;
  publicKeyFingerprint?: string;
};

export type UpdateRecordRequest = {
  id: string;
  metadata: Record<string, unknown>;
  secret: Record<string, unknown>;
  publicKeyOpenSSH?: string;
  publicKeyPEM?: string;
  publicKeyFingerprint?: string;
};

export type SearchRecordsRequest = {
  query?: string;
  recordType?: RecordType;
  tags?: string[];
};
```

Required Wails methods:

```go
ListRecords() ([]RecordSummaryDTO, error)
GetRecord(id string) (RecordDTO, error)
CreateRecord(request CreateRecordRequest) (RecordDTO, error)
UpdateRecord(request UpdateRecordRequest) (RecordDTO, error)
DeleteRecord(id string) error
SearchRecords(request SearchRecordsRequest) ([]RecordSummaryDTO, error)
```

---

## 10.4 Password Generator DTOs

```ts
export type GeneratePasswordRequest = {
  length: number;
  includeUppercase: boolean;
  includeLowercase: boolean;
  includeNumbers: boolean;
  includeSymbols: boolean;
  excludeAmbiguous: boolean;
};

export type GeneratedPasswordDTO = {
  password: string;
  length: number;
};
```

Required Wails method:

```go
GeneratePassword(request GeneratePasswordRequest) (GeneratedPasswordDTO, error)
```

---

## 10.5 Key Generator DTOs

```ts
export type GenerateAESKeyRequest = {
  keyLength: 128 | 192 | 256;
};

export type GeneratedSymmetricKeyDTO = {
  algorithm: "AES";
  keyLength: 128 | 192 | 256;
  keyBase64: string;
  keyHex: string;
};

export type GenerateRSAKeyPairRequest = {
  keySize: 2048 | 3072 | 4096;
};

export type GenerateEd25519KeyPairRequest = {};

export type GeneratedKeyPairDTO = {
  algorithm: "RSA" | "Ed25519";
  keySize?: number;
  publicKeyOpenSSH: string;
  publicKeyPEM?: string;
  publicKeyFingerprint: string;
  privateKeyPEM: string;
};
```

Required Wails methods:

```go
GenerateAESKey(request GenerateAESKeyRequest) (GeneratedSymmetricKeyDTO, error)
GenerateRSAKeyPair(request GenerateRSAKeyPairRequest) (GeneratedKeyPairDTO, error)
GenerateEd25519KeyPair(request GenerateEd25519KeyPairRequest) (GeneratedKeyPairDTO, error)
```

---

## 10.6 Clipboard DTOs

```ts
export type CopyToClipboardRequest = {
  value: string;
  secretKind:
    | "password"
    | "api_key"
    | "aes_key"
    | "public_key"
    | "private_key"
    | "secure_note";
};
```

Required Wails methods:

```go
CopyToClipboard(request CopyToClipboardRequest) error
ClearClipboard() error
```

---

## 10.7 Export DTOs

```ts
export type ExportKeyRequest = {
  recordId: string;
  keyMaterial:
    | "public_openssh"
    | "public_pem"
    | "private_pkcs8_pem"
    | "private_openssh";
  destinationPath: string;
};
```

Required Wails methods:

```go
ExportPublicKey(request ExportKeyRequest) error
ExportPrivateKey(request ExportKeyRequest) error
```

Backend must verify vault unlock state before export.

---

# 11. Validation Rules

## 11.1 Master Password

* Minimum length: 12 characters.
* Confirmation must match.
* Empty password rejected.
* Arbitrary composition rules are not required.
* Passphrases must be accepted.

---

## 11.2 Password Generator

* Minimum length: 8.
* Default length: 24.
* Maximum length: 128.
* At least one character class must be selected.

---

## 11.3 AES Key Generator

Allowed key lengths:

```text
128
192
256
```

---

## 11.4 RSA Key Generator

Allowed key sizes:

```text
2048
3072
4096
```

Default:

```text
3072
```

---

## 11.5 Record Validation

Every record must have:

* ID.
* Vault ID.
* DEK ID.
* Record type.
* Created timestamp.
* Updated timestamp.
* Encrypted metadata blob.
* Encrypted secret blob.

Every decrypted record must have:

* Title, except where record type explicitly allows otherwise.
* Valid type-specific secret material.

---

# 12. Error Handling and Logging

## 12.1 User-Facing Error Rules

Errors must be:

* Clear.
* Safe.
* Non-leaky.
* Mapped to typed error codes.

Do not expose:

* Raw crypto errors.
* SQL internals.
* Stack traces.
* Wrapped sensitive errors.
* Master password details.
* KDF internals.
* Decrypted payloads.

---

## 12.2 Logging Policy

Logs must never contain:

* Master passwords.
* KEKs.
* DEKs.
* Plaintext record data.
* Passwords.
* API keys.
* AES keys.
* Private keys.
* Generated passwords.
* Decrypted vault payloads.
* Clipboard contents.

Use sanitized logging at service boundaries.

Any internal error that may contain sensitive context must be converted into a safe public error before crossing the Wails boundary.

---

# 13. Auto-Lock Requirements

## 13.1 Default

```text
Enabled by default
Default timeout: 10 minutes
```

Allowed settings:

```text
1 minute
5 minutes
10 minutes
15 minutes
30 minutes
Disabled with warning
```

## 13.2 Activity Events

The app should reset the idle timer on:

* Mouse interaction.
* Keyboard interaction.
* Navigation.
* Record view.
* Record edit.
* Copy/export action.
* Generator interaction.

## 13.3 Lock Behavior

When auto-lock triggers:

1. Clear decrypted frontend record state.
2. Clear backend decrypted record cache.
3. Clear in-memory DEK where practical.
4. Attempt clipboard clear.
5. Return UI to locked state.
6. Require master password to unlock again.

## 13.4 System Sleep

Lock-on-system-sleep/lid-close is Should-have for MVP hardening, not required for v0.1 dogfoodable build.

---

# 14. Testing Requirements

## 14.1 Required Go Unit Tests

* Argon2id KDF parameter handling.
* KEK derivation.
* DEK generation.
* DEK wrapping/unwrapping.
* Wrong password unwrap failure.
* AES-GCM encrypt/decrypt.
* AAD mismatch failure.
* Nonce generation.
* Record encryption/decryption.
* Record tamper detection.
* Password generator.
* AES key generator.
* RSA key generator.
* Ed25519 key generator.
* OpenSSH public key export.
* PKCS#8 private key export.
* SQLite application ID validation.
* SQLite migration application.
* Master password change/re-wrap.

---

## 14.2 Required Frontend Tests

* Create vault form validation.
* Unlock form validation.
* Change master password form validation.
* Record form validation.
* Password generator options.
* Key generator options.
* Clipboard countdown component.
* Private key warning modal.
* Locked-state action blocking.
* Auto-lock state transition.

---

## 14.3 Manual Cross-Platform Test Matrix

Required on macOS and Windows before MVP completion:

| Test                                      |    macOS |  Windows |
| ----------------------------------------- | -------: | -------: |
| App launches                              | Required | Required |
| Create vault                              | Required | Required |
| Open existing vault                       | Required | Required |
| Unlock with correct password              | Required | Required |
| Wrong password failure                    | Required | Required |
| Change master password                    | Required | Required |
| Create credential                         | Required | Required |
| Edit credential                           | Required | Required |
| Delete credential                         | Required | Required |
| Generate password                         | Required | Required |
| Copy password                             | Required | Required |
| Clipboard countdown                       | Required | Required |
| Clipboard clear attempt                   | Required | Required |
| Auto-lock                                 | Required | Required |
| Generate AES key                          | Required | Required |
| Generate RSA key                          | Required | Required |
| Generate Ed25519 key                      | Required | Required |
| Export OpenSSH public key                 | Required | Required |
| Export private key with warning           | Required | Required |
| Lock vault                                | Required | Required |
| Reopen vault after app restart            | Required | Required |
| SQLite file contains no plaintext secrets | Required | Required |

---

# 15. Milestones

## v0.1 — Dogfoodable Vault

Goal: First usable local vault.

Scope:

* Wails v2 project skeleton.
* Vite + React + TypeScript + Mantine.
* SQLite setup.
* `golang-migrate`.
* Vault creation.
* Vault unlock.
* Vault lock.
* DEK/KEK envelope encryption.
* Credential records only.
* Password generator.
* Copy password with countdown.
* Basic manual testing on one active OS.

Exit criteria:

* User can create a vault.
* User can unlock/lock vault.
* User can create/read/update/delete credentials.
* Credentials are encrypted in SQLite.
* User can generate/copy passwords.
* Clipboard countdown works.

---

## v0.2 — Security Foundation

Scope:

* Auto-lock.
* Change master password.
* AAD enforcement.
* Typed error system.
* Logging policy.
* SQLite `application_id`.
* Rollback journal policy.
* Backend crypto tests.

Exit criteria:

* Wrong password fails safely.
* AAD tampering tests fail as expected.
* Master password can be changed without re-encrypting records.
* Auto-lock clears session state.
* Frontend uses typed error codes.

---

## v0.3 — Expanded Record Types

Scope:

* API key records.
* Secure note records.
* Symmetric key records.
* In-memory search.
* Encrypted tags.
* Record type filtering.

Exit criteria:

* User can manage all non-asymmetric record types.
* Tags/search work after unlock.
* SQLite contains no plaintext sensitive metadata.

---

## v0.4 — Key Generation and Export

Scope:

* AES key generation.
* RSA key generation.
* Ed25519 key generation.
* OpenSSH public key export.
* PKCS#8 private key export.
* Private key warning modal.
* Key pair record storage.

Exit criteria:

* User can generate and save AES keys.
* User can generate and save RSA key pairs.
* User can generate and save Ed25519 key pairs.
* User can export public keys in OpenSSH format.
* User can export private keys only after explicit warning.

---

## v0.5 — Cross-Platform Hardening

Scope:

* macOS testing.
* Windows testing.
* File dialog testing.
* Clipboard testing.
* Export path testing.
* Packaging notes.
* README setup instructions.

Exit criteria:

* App builds/runs on macOS and Windows.
* Manual test matrix passes.
* Known platform differences are documented.

---

## v1.0 — MVP Complete

Scope:

* Polish.
* Documentation.
* Threat model.
* Security limitations.
* Final test pass.
* Build/release checklist.

Exit criteria:

* MVP definition of done is satisfied.

---

# 16. Acceptance Scenarios

## Create Vault

```gherkin
Given I launch the app
When I create a new vault with a valid master password
Then a SQLite vault file is created
And the vault has a vault_id
And an active DEK is created
And the DEK is wrapped by a master-password-derived KEK
And the vault opens unlocked
```

---

## Unlock Vault

```gherkin
Given I have an existing vault
When I enter the correct master password
Then the app unwraps the DEK
And the vault unlocks
And I can view my records
```

---

## Wrong Password

```gherkin
Given I have an existing vault
When I enter an incorrect master password
Then the vault does not unlock
And no records are displayed
And the app shows a generic unlock failure message
```

---

## Change Master Password

```gherkin
Given the vault is unlocked
When I change the master password
Then the app re-wraps the existing DEK
And records are not re-encrypted
And the old master password no longer unlocks the vault
And the new master password unlocks the vault
```

---

## Create Credential

```gherkin
Given the vault is unlocked
When I create a credential record
Then the record appears in the record list
And the password is masked by default
And the SQLite database does not contain the plaintext title, username, URL, notes, tags, or password
```

---

## AAD Tampering Detection

```gherkin
Given an encrypted record exists
When an attacker swaps encrypted_secret_blob between two records
Then decryption fails due to AAD mismatch
And the app does not display the swapped secret
```

---

## Generate Password

```gherkin
Given the vault is unlocked
When I generate a 24-character password
Then the password is generated using cryptographically secure randomness
And I can copy it
And I can save it as a credential
```

---

## Clipboard Countdown

```gherkin
Given the vault is unlocked
When I copy a secret
Then the app displays a countdown progress bar
And the app attempts to clear the clipboard when the timer expires
```

---

## Auto-Lock

```gherkin
Given the vault is unlocked
When the user is idle for the configured timeout
Then the vault locks automatically
And decrypted state is cleared where practical
And clipboard clear is attempted
```

---

## Export Private Key

```gherkin
Given the vault is unlocked
And I have an asymmetric key pair record
When I choose to export the private key
Then the app displays a private key exposure warning
And export continues only after I explicitly confirm
```

---

# 17. Definition of Done

The MVP is complete when:

1. App builds and runs on macOS.
2. App builds and runs on Windows.
3. App uses Wails v2.
4. Frontend uses Vite + React + TypeScript + Mantine.
5. SQLite vault creation works.
6. SQLite `application_id` is set and validated.
7. Migrations run through `golang-migrate`.
8. Vault uses DEK/KEK envelope encryption.
9. Master password unlocks the wrapped DEK.
10. Change master password re-wraps the DEK.
11. Records use AES-256-GCM with AAD.
12. `vault_id`, `dek_id`, and `record_envelope_version` are implemented.
13. Secrets are encrypted before SQLite persistence.
14. No plaintext secrets appear in SQLite.
15. User can create, view, edit, delete, and search records after unlock.
16. User can generate secure passwords.
17. User can generate AES keys.
18. User can generate RSA key pairs.
19. User can generate Ed25519 key pairs.
20. OpenSSH public key export works.
21. PKCS#8 private key export works.
22. Private key copy/export requires explicit warning.
23. Clipboard countdown progress bar works.
24. Clipboard clear is attempted after configured timeout.
25. Auto-lock works.
26. Locked vault blocks secret actions.
27. Typed error codes are used.
28. Logs do not contain secrets.
29. README documents setup, usage, threat model, and limitations.
30. Manual macOS + Windows test matrix passes.

---

# 18. AI Scrum Team Build Rules

## Hard Rules

1. Do not add cloud sync.
2. Do not add browser autofill.
3. Do not add user accounts.
4. Do not add team sharing.
5. Do not add Linux support unless explicitly approved.
6. Do not store plaintext secrets in SQLite.
7. Do not log secrets.
8. Do not implement custom cryptographic algorithms.
9. Do not expose private keys without explicit warning.
10. Do not use Next.js.
11. Do not use SSR/server-side frontend behavior.
12. Do not use WAL mode in MVP.
13. Do not use `math/rand` for passwords, keys, salts, or nonces.
14. Do not string-match errors in the frontend.
15. Do not couple ordinary schema migrations to record decryption.

---

## Preferred Implementation Choices

* Wails v2 stable.
* Vite + React + TypeScript.
* Mantine UI.
* SQLite.
* `golang-migrate`.
* Rollback journal mode.
* Argon2id.
* AES-256-GCM.
* DEK/KEK envelope encryption.
* AAD for all AES-GCM operations.
* `crypto/rand`.
* `golang.org/x/crypto/ssh` for OpenSSH public key marshaling.
* In-memory search after unlock.
* Typed error DTOs.
* Sanitized logging.

---

# 19. README Requirements

The README must include:

1. Project purpose.
2. Development setup.
3. macOS setup.
4. Windows setup.
5. Build instructions.
6. Test instructions.
7. Vault security model.
8. Threat model.
9. No-recovery warning.
10. Clipboard limitations.
11. Go memory handling limitations.
12. Private key export warning.
13. Known limitations.
14. Non-goals.

Required README warning:

```text
This project is a learning-grade local password and key vault. It uses modern cryptographic patterns, but it has not been formally audited and should not be treated as an enterprise password manager.
```

Required clipboard warning:

```text
Clipboard clearing is best-effort. Operating system features and third-party clipboard managers may retain clipboard history outside this app’s control.
```

Required master password warning:

```text
There is no password recovery. If you lose your master password, the vault cannot be recovered.
```

---

# 20. Final Implementation Sequence

Build in this order:

1. Wails + Vite + Mantine skeleton.
2. SQLite connection and migrations.
3. Vault metadata, `deks`, and `key_wrappings`.
4. Argon2id KDF.
5. DEK generation and wrapping.
6. Vault create/unlock/lock.
7. Credential CRUD.
8. Record encryption with AAD.
9. Password generator.
10. Clipboard copy/countdown.
11. Auto-lock.
12. Change master password.
13. API key and secure note records.
14. AES key generation.
15. RSA key generation.
16. Ed25519 key generation.
17. Public/private key export.
18. Cross-platform hardening.
19. README and threat model.
20. Final macOS + Windows test pass.

