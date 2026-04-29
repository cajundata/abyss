# Feature Research

**Domain:** Local-first desktop password & cryptographic key vault (single-user, learning-grade)
**Researched:** 2026-04-29
**Confidence:** HIGH (PRD scope is fixed; research validates against industry norms)

---

## Reference Products

Brief survey of the relevant comparator set. Abyss positions closest to **KeePassXC**, with first-class crypto-key generation/export borrowed from the SSH/key-management world.

| Product | Storage Model | Scope vs Abyss |
|---------|---------------|----------------|
| **KeePassXC** | Local KDBX file, optional cloud-sync via 3rd-party file sync (Dropbox, etc.) | Closest analogue. Stores credentials, notes, attachments, SSH keys (via KeeAgent). Has password generator, clipboard timeout, auto-lock. Abyss is essentially a narrower KeePassXC scope with first-class key-pair record types and OpenSSH/PKCS#8 export built in. |
| **KeePass (original)** | Local KDBX file, Windows-first | Older Windows-native fork. Same shape as KeePassXC but more dated UX. Reference for KDBX format and security expectations. |
| **Bitwarden (self-hosted desktop)** | Cloud-first, encrypted vault sync; self-host server option | Cloud sync is core; local-only is not the primary mode. Different threat model. Abyss explicitly excludes cloud sync (PRD §3.3). |
| **1Password (legacy standalone)** | Local AGILEKEYCHAIN/OPVAULT until 1P8; now cloud-only | Older standalone mode is the closest 1P analogue. Modern 1P is cloud-only and not directly comparable. |
| **pass / gopass** | Per-secret GPG-encrypted files in a Git repo | CLI-first, files-on-disk, per-record encryption (similar in spirit to Abyss's per-record envelope, but using GPG). Inspires Abyss's per-record encryption choice. |
| **Buttercup** | Local file + multi-vault, optional cloud sync | Electron-based local-first. Demonstrates the shape Abyss is targeting on the desktop side. |
| **Enpass** | Local file + optional cloud sync via WebDAV/Dropbox | Local-first commercial product. Clipboard timeout configurable; default 30s historically. |

**Abyss's positioning:** Single-vault, single-user, no-sync, no-extension desktop product with first-class symmetric/asymmetric key generation and OpenSSH/PKCS#8 export. Closest commercial-quality analogue is **KeePassXC**, but Abyss intentionally drops attachments, browser integration, SSH-agent serving, key files, multi-vault, plugins, and anything network-touching.

---

## Feature Landscape

### Table Stakes (Users Expect These)

These are the features that, if missing, make a local-first vault feel broken or untrustworthy. Each row notes whether the PRD covers it.

| Feature | Why Expected | Complexity | PRD Coverage | Notes |
|---------|--------------|------------|--------------|-------|
| Create vault with master password | Foundational. The "first run" act. | LOW | Covered (FR-001…006, §9.3) | PRD also requires the "no recovery" warning on this screen — correct posture. |
| Unlock vault with master password | Foundational. | LOW | Covered (FR-007, §9.4) | PRD requires generic error on failure — correct (avoids unlock oracle). |
| Manual lock | Users expect a "lock now" affordance. | LOW | Covered (FR-008) | |
| Auto-lock on idle | Standard across all reference products. Primary mitigation against unattended sessions. | MEDIUM | Covered (FR-009, §13) | Default 10 min; allowed values 1/5/10/15/30/Disabled. Industry-aligned (1Password defaults vary; KeePassXC user-configurable; Bitwarden vault-timeout configurable). |
| Strong KDF (Argon2id) | Modern best practice. PBKDF2 is acceptable but dated; Argon2id is current OWASP recommendation. | MEDIUM | Covered (PRD §5.1, §5.3) | PRD's 256 MiB / 3 iter / 4 parallelism target **substantially exceeds** OWASP minimum (19 MiB / 2 iter / 1 parallelism). Strong choice. Fallback profile (64 MiB) is also above OWASP minimum. |
| Authenticated encryption (AEAD) | AES-GCM or ChaCha20-Poly1305. Plain AES-CBC is insufficient. | MEDIUM | Covered (PRD §5.1, §5.4) | AES-256-GCM with AAD on every operation. PRD AAD strings (record_metadata / record_secret / wrapped_dek) prevent record-swap attacks. Strong design. |
| CSPRNG-backed password generator | Users will not type strong passwords; the generator is the primary secret-creation surface. | LOW | Covered (FR-027…033) | Defaults length=24, all classes on, exclude ambiguous on. **Validated against industry:** KeePassXC defaults to length 25, NordPass/Bitwarden/1Password commonly default 16–20. Abyss's 24 is on the strong side of the range — appropriate for a security-positioning product. |
| Clipboard copy with best-effort clear | Without this, the only way to use stored secrets is to retype them. | MEDIUM | Covered (FR-057…064) | Default 30s timeout, configurable {10/30/60/120}. **Validated:** KeePassXC default = 10s; NordPass default = 30s; Bitwarden default = "never" (criticized as unsafe). 30s default is mid-industry and defensible. |
| Search across stored items | Users will accumulate dozens of records; without search the product is unusable past ~20 entries. | LOW | Covered (FR-022) | In-memory after unlock — correct (avoids decrypting on disk for every keystroke). |
| Tags / categorization | Standard organization mechanism; folders are the alternative. | LOW–MEDIUM | Covered (FR-026, §9.5 record-type filter) | PRD chooses tags + record-type filter, no folders. Reasonable simplification — folders add tree-management UX without security value. |
| Multiple item types | Beyond plain credentials, users have API keys, secure notes, etc. | MEDIUM | Covered (5 types: credential / api_key / secure_note / symmetric_key / asymmetric_key_pair) | Differentiator on the cryptographic types — see below. |
| Mask secrets by default + reveal | Shoulder-surfing mitigation; standard UX. | LOW | Covered (FR-023, FR-024) | |
| Edit / delete with confirmation | CRUD completeness. | LOW | Covered (FR-020, FR-021) | |
| Master password change without re-encrypting all records | DEK/KEK envelope is the standard pattern that enables this. Without it, master-password change is either impossible or a slow re-encryption job. | MEDIUM | Covered (FR-010, §5.2.3) | Re-wrap the DEK with a new KEK, replace the wrapping row. Records untouched. Industry standard. |
| "No recovery" warning at vault creation | Users have lost data without this; setting expectation up front is critical. | LOW | Covered (§9.3, §19 README) | Correct posture for a local-first product without escrow. |
| Public threat model | Users (and reviewers) need to know what the product does and doesn't protect against. | LOW | Covered (§4, §19 README) | PRD §4.2 is explicit and honest. Strong. |
| Generic unlock failure (no oracle) | An attacker should not learn whether the master password vs DB structure was wrong. | LOW | Covered (§9.4, FR-007 implicit) | Correct. |
| Per-record nonce; never reuse | Crypto correctness baseline. Reuse with same key = catastrophic. | LOW | Covered (§5.5) | |
| Per-wrapping salt | Salt-per-row enables future multi-wrapping (e.g., add a second unlock factor without rotating everything). | LOW | Covered (§5.3, key_wrappings table) | |
| Application-ID identification of vault file | Don't rely on file extension to identify vault format. | LOW | Covered (§6.1, FR-003) | `application_id = 0x4E56544E`. |
| Sanitized logging | Logs in `~/Library/Logs` or Windows Event Log are routinely uploaded with bug reports — leaking secrets there is a real failure mode. | LOW | Covered (§12.2) | Explicit list of forbidden log content. |

**Verdict:** Every table-stakes feature is in scope. Defaults (clipboard 30s, auto-lock 10min, password length 24, RSA 3072, Argon2id 256MiB) are at or above industry norms.

---

### Differentiators (Abyss-specific value)

Features that distinguish Abyss from the password-manager mainstream. These are also where most of the learning value lives.

| Feature | Value Proposition | Complexity | PRD Coverage | Notes |
|---------|-------------------|------------|--------------|-------|
| Native AES key generation (128/192/256, Base64 + hex display) | Most password managers do not generate symmetric keys at all. Users currently generate AES keys at the CLI (`openssl rand -base64 32`) and paste them into a secure note. Abyss makes this first-class. | LOW | Covered (FR-034…040, §7.4) | Symmetric key record type with algorithm/key_length metadata. |
| Native RSA key generation (2048/3072/4096; default 3072) | Same friction story — most users currently use `ssh-keygen` outside the vault, then store the result as an attachment or note. Abyss generates and stores in one step. | MEDIUM | Covered (FR-041…048) | **Validated default 3072:** [NIST SP 800-57](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt3r1.pdf) recommends 2048 acceptable through 2030 (112-bit security), 3072 preferred for use beyond 2030 (128-bit security). RSA 4096 has CPU cost without proportional benefit. **3072 is the correct modern default.** |
| Native Ed25519 key generation | Modern SSH default; smaller, faster, often-preferred over RSA for new keys. | LOW | Covered (FR-049…056) | |
| First-class asymmetric key pair record type with public/private separation | Public key stored plaintext (correctly — public keys are not secrets); private key encrypted. Public-key fingerprint visible without unlock would be a future enhancement (PRD §2.9 acknowledges this). | MEDIUM | Covered (§7.5, records.public_key_* columns) | **Trade-off PRD §2.9:** even though public keys are not secret, the MVP gates record interaction behind unlock for UX simplicity. This is honest and correct — exposing a "locked-state public key ring" is a separable feature with real UX cost (a different navigation tree) and no security benefit at MVP scale. Defer is correct. |
| OpenSSH public key export (RSA + Ed25519) | The dominant interop format for SSH workflows. Without this, users have to convert by hand (`ssh-keygen -i -m PKCS8 …`) which is friction and an exposure window. | MEDIUM | Covered (FR-044, FR-051) | Uses `golang.org/x/crypto/ssh` for marshaling — the right library. |
| PKCS#8 PEM private-key export | Standard interop format for non-SSH consumers (TLS, JWT signing libs, KMSes). | MEDIUM | Covered (FR-046, FR-053) | |
| OpenSSH private-key export (Ed25519) | Lets users drop the file straight into `~/.ssh/id_ed25519`. Should-have, not Must-have. | MEDIUM | Covered (FR-054 = Should) | Reasonable scope choice. |
| Private-key exposure warning modal | Forces users to acknowledge the risk before exposing private key material. Not mere UX — it shifts the failure mode from "I accidentally clicked Copy" to "I made an explicit choice." | LOW | Covered (§9.8, FR-047, FR-055) | Modal text in PRD is appropriate. |
| Honest "learning-grade" positioning | Sets expectations correctly. The README warning ("not formally audited, not enterprise-grade") protects both users and the project from misuse. | LOW | Covered (§19) | Distinguishes Abyss from products that overclaim. |
| AAD bound to envelope, not to schema_version | Subtle but correct: ordinary schema migrations should not invalidate every record's authentication tag. PRD §5.4 explicitly excludes `schema_version` from AAD. | LOW | Covered (§5.4) | Many homegrown vaults get this wrong. |
| DEK/KEK envelope with separable wrapping rows | The `key_wrappings` table is keyed on `wrapping_type`, allowing future addition of biometric / hardware-key wrappings without re-encrypting records. Sets up post-MVP work cleanly. | MEDIUM | Covered (§6.3 key_wrappings) | Strong forward-compatibility design even though MVP only ships `master_password` wrapping type. |

**Differentiator summary:** Abyss is "a vault that natively understands SSH/TLS/AES key material" rather than "a credentials store with attachments." This is a real differentiator vs the KeePassXC/Bitwarden/1Password landscape, which all treat keys as opaque files to attach.

---

### Anti-Features (Deliberately NOT building)

Features that look attractive but create disproportionate risk or scope. Most are explicitly excluded by PRD §3.3 or §18 — this section validates those exclusions and adds a few that the PRD has not explicitly named but should.

| Anti-Feature | Why Tempting | Why Problematic | What Abyss Does Instead |
|--------------|--------------|-----------------|-------------------------|
| **Cloud sync** | Convenience; "use from any device" is the marketing pitch users expect. | Different threat model (server compromise, MITM, account takeover). Pulls in auth, sessions, network code, conflict resolution, and a server-side product. None are MVP. | Out of scope (PRD §3.3). Local-first; users sync via filesystem (Dropbox / iCloud Drive / a USB stick) at their own risk. |
| **Browser autofill / browser extension** | The single most-used feature of commercial password managers. | Massive attack surface: extension → page DOM → frame messaging. CVE history is long (LastPass, 1Password, Bitwarden have all had extension-side vulns). Wrong scope for a learning project. | Out of scope (PRD §3.3). Users copy/paste with clipboard countdown. |
| **Plugin / extension system** | "Make it customizable." | Plugins in a security product = arbitrary code with vault access. Cannot be safely sandboxed without significant infrastructure. | Out of scope (PRD §3.3, §18 Hard Rule). |
| **Custom cryptographic algorithms** | "Roll my own KDF / cipher" sometimes attracts hobbyists. | Universal anti-pattern. Side-channel and correctness failures are catastrophic. | Out of scope (PRD §18.1). Use Go stdlib + `golang.org/x/crypto`. |
| **`math/rand` for any security-sensitive value** | Easier API than `crypto/rand`. | Predictable. Catastrophic for passwords / salts / nonces / keys. | Out of scope (PRD §18 Hard Rule). `crypto/rand` only. |
| **Unsigned auto-update** | "Users want updates." | An auto-update channel is a privileged code-execution channel. Without code signing it is a malware delivery vector. | Out of scope (PRD §3.3). Manual builds for MVP. |
| **Cloud password breach monitoring** | "Have I Been Pwned" style features feel responsible. | Requires network. Different threat model. Sends hashes (or worse) outside the user's machine. | Out of scope (PRD §3.3). |
| **Recovery questions / "security questions"** | "Don't lose your vault." Users ask for it. | [Universally considered the weakest link.](https://cheatsheetseries.owasp.org/cheatsheets/Choosing_and_Using_Security_Questions_Cheat_Sheet.html) Answers are guessable from social media. Reduces the master password's strength to the strength of the recovery answer. Recent 2026 research found 25 attacks against password-manager recovery mechanisms specifically. | **Implicit in PRD's "no recovery" stance** (§9.3, §19). The PRD takes the explicit position that there is no recovery — this is the correct posture for a local-first vault. **Recommend FEATURES.md call this out as a deliberate exclusion** so future contributors don't try to "helpfully" add it. |
| **Custom password-strength estimator** | "Tell users how strong their password is." | Naïve estimators give false confidence (e.g., counting character classes rewards `Password1!`). | PRD §8.3 says use a vetted zxcvbn-style library only, do not roll one. Correct. |
| **"Remember last unlock" / "stay unlocked across app launches"** | Convenience. | Subverts the entire lock model. The DEK survives a relaunch, sitting on disk in some form. | **Not explicitly named in PRD but implied** by the lifecycle: vault unlock requires master password every launch (FR-007). Should be called out as an anti-feature so future "UX improvements" don't add it. |
| **Recent-vaults menu that auto-unlocks** | "Convenience." | Same problem as above — short-circuits lock semantics. | PRD §9.2 allows a recent-vaults *list* (showing paths only, not unlocking) and defers it unless trivial. Correct nuance. |
| **Whole-database encryption (SQLCipher)** | "Encrypt the whole file, simpler." | Forces re-encrypting on every open, complicates DB tooling, and conflates the schema layer with the secrets layer. | Out of scope (PRD §2.4). Per-record encryption preserves SQL queryability of non-secret columns and keeps the migration story simple. |
| **WAL journal mode** | Better write throughput. | Sidecar files (`-shm`, `-wal`) complicate vault portability — users zip the `.db` and lose unflushed data. | Out of scope (PRD §2.4). Rollback journal mode for portability. |
| **Plaintext "vault hint" stored in the DB** | "Help users remember their master password." | A hint useful enough to actually remind the user is also useful enough to help an attacker. | **Not in PRD; recommend continuing to omit.** Master password should stand on its own. |
| **Auto-lock disabled by default** | "Users find auto-lock annoying." | Defeats the primary mitigation against unattended sessions (PRD §4.3). | PRD §13.1: auto-lock enabled by default at 10 min. Disabling requires "Disabled with warning" explicit choice. Correct. |
| **Telemetry / analytics** | "Understand how the product is used." | A vault should not phone home. Period. | **Not in PRD; recommend explicit exclusion** in PROJECT.md Out of Scope. |
| **TOTP authenticator built into the vault** | "Replace Authy / Google Authenticator too." | Conflates "thing you have" with "thing you know" — co-locating the password and the second factor in the same vault eliminates the second factor's security benefit. | Out of scope (PRD §3.3). Correct rationale. |
| **SSH agent integration (serving keys to ssh)** | Power-user feature; KeePassXC has it via KeeAgent. | Real attack surface (unix sockets, named pipes, IPC). Not MVP scope. | Out of scope (PRD §3.3). Users export PKCS#8 / OpenSSH and run their own agent. |
| **PGP key management** | Power-user feature. | Different format family (OpenPGP packets), different lifecycle (subkeys, revocation, web of trust). Whole separate product. | Out of scope (PRD §3.3). |

**Verdict:** The PRD's exclusion list is well-reasoned. The two gaps worth surfacing are (1) "remember last unlock" should be explicitly named as an anti-pattern in the README/threat model, and (2) telemetry should be an explicit "we will never add this" item, not a silent omission.

---

## Behavioral Defaults — Validated Against Industry

| Default | PRD Value | Industry Comparison | Verdict |
|---------|-----------|---------------------|---------|
| Clipboard timeout | 30s default; configurable {10, 30, 60, 120} | KeePassXC = 10s, NordPass = 30s, Bitwarden = "never" (criticized), 1Password ≈ 90s historically. ([TechSpot](https://www.techspot.com/news/97320-you-change-password-manager-clipboard-settings-now.html)) | **Mid-industry, defensible.** 10s is more conservative; 30s strikes a balance between security and "users actually have time to paste." Configurable down to 10s is good. |
| Auto-lock idle timeout | 10 min default; configurable {1, 5, 10, 15, 30, Disabled} | 1Password defaults to "never" with strong nudging toward setting one; Bitwarden default varies (browser extension = 15 min historically). KeePassXC user-configurable. ([Bitwarden vault timeout docs](https://bitwarden.com/help/vault-timeout/), [1Password unlock & auto-lock docs](https://support.1password.com/unlock-auto-lock/)) | **10 min is a reasonable middle ground.** Conservative options (1, 5 min) are available. "Disabled with warning" is the right pattern — don't quietly let users disable a security primitive. |
| Password generator length | 24 chars; all classes on; exclude ambiguous on | KeePassXC = 25, Bitwarden ≈ 14, 1Password ≈ 20, NordPass = 16. | **24 is on the strong side; appropriate for the product positioning.** Classes-on by default is industry-standard. Exclude-ambiguous default is conservative (slightly reduces entropy by ~2 bits at length 24, in exchange for fewer transcription errors when user reads-then-types — defensible for a vault that emphasizes copy-paste anyway). |
| RSA default key size | 3072 | NIST SP 800-57: 2048 acceptable through 2030, 3072 preferred beyond 2030. RSA 4096 has CPU cost without proportional benefit. ([NIST SP 800-57 Part 3 Rev 1](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt3r1.pdf), [Fastly: Key size for TLS](https://www.fastly.com/blog/key-size-for-tls)) | **Correct modern default.** 2048 still offered (interop with older systems); 4096 offered (paranoid users); 3072 default is the current best balance. |
| Argon2id parameters | Target: 256 MiB / 3 iter / 4 parallelism. Fallback: 64 MiB / 3 / 1. Minimum: 19 MiB / 2 / 1. | OWASP minimum is 19 MiB / 2 / 1 ([OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)). | **Substantially stronger than OWASP minimum.** 256 MiB is at the high end of usable (will be slow on a 4 GB laptop). Fallback profile (64 MiB) is well above OWASP minimum. Tuning per-machine is the right approach. |
| Master password minimum length | 12 chars; passphrases allowed; no composition rules | NIST 800-63B: 8 char minimum, no composition rules, allow long passphrases. KeePass (original) has no minimum. 1Password ≈ 10 char minimum historically. | **12 is reasonable** for a vault master password (a stronger bar than NIST 800-63B's 8-char general-purpose minimum). The "no composition rules; passphrases allowed" stance follows current NIST guidance. |
| "No recovery" stance | No recovery, no escrow, no security questions | Bitwarden / LastPass support emergency-access / escrow with documented vulnerabilities. KeePassXC has no recovery. pass has no recovery. | **Correct for a local-first single-user product.** Recovery mechanisms have been a [recurring source of CVEs in commercial password managers](https://thehackernews.com/2026/02/study-uncovers-25-password-recovery.html). Better to be honest about it. |

---

## Feature Dependencies

```
Vault create / DB schema
    └── DEK generation
        └── KEK derivation (Argon2id)
            └── DEK wrapping (AES-GCM + AAD)
                └── Vault unlock
                    ├── Record CRUD
                    │   ├── Search (in-memory)
                    │   ├── Tags / filtering
                    │   └── Mask / reveal / copy
                    │       └── Clipboard countdown
                    │           └── Clipboard auto-clear (depends on lock)
                    ├── Auto-lock (idle tracking)
                    │   └── Lock behavior (clear DEK, clear clipboard, clear UI state)
                    ├── Change master password
                    │   └── (re-uses KEK derivation + DEK wrapping; does NOT touch records)
                    └── Generators
                        ├── Password generator (CSPRNG only)
                        ├── AES key generator
                        ├── RSA key pair generator
                        │   └── OpenSSH public-key export
                        │   └── PEM public-key export
                        │   └── PKCS#8 private-key export (gated by warning modal)
                        └── Ed25519 key pair generator
                            └── (same export options as RSA; OpenSSH private-key export Should-have)

AAD enforcement
    └── enables: Tamper detection (record-swap, blob-swap)
    └── enables: ordinary schema migration without forced re-encryption

Typed AppErrorCode DTO
    └── enables: frontend error handling without string-matching
    └── enables: sanitized public errors (no oracle, no leaks)
```

### Critical Dependency Notes

- **Auto-lock depends on the entire lock chain working:** clear backend DEK, clear backend record cache, clear frontend record state, attempt clipboard clear, return UI to locked state. PRD §13.3 spells this out — all five steps must run on every lock event (manual, auto, app-close-equivalent). Implementation should funnel all lock paths into one routine.
- **Change master password depends on DEK/KEK separation, not on records:** by design, records are not touched. This is what makes master-password change fast. If anyone proposes "let's also rotate the DEK on master-password change," that is a separate feature (DEK rotation) and pulls in re-encryption — it should not be conflated.
- **Search depends on records being decrypted in memory after unlock.** This is correct and standard for local-first vaults. The cost is memory residency of decrypted records during the unlocked session — explicitly acknowledged in PRD §4.3.
- **Public-key fingerprint visibility currently depends on unlock** (PRD §2.9). Even though public keys are not secret, the records table gates all interaction behind unlock for UX simplicity. Future v0.6+ feature could expose a locked-state public-key ring; not MVP.
- **Encrypted tags (FR-026, Should) depend on the metadata blob format.** Tags live inside `encrypted_metadata_blob` per PRD §7.x. Implementing this is "free" once metadata blobs are encrypted — there is no separate tag table.
- **Clipboard countdown frontend ↔ backend split:** Frontend owns the visible countdown UI; backend owns clipboard write/clear. This means the countdown timer is *in the frontend*, but the clear action calls back into Go. If the Wails IPC is slow or the frontend hangs, the backend clear may not fire on time — this is a known limitation that the README clipboard warning covers.

---

## MVP Definition (Aligned With PRD §15)

The PRD already defines six milestones (v0.1 → v1.0). Re-stating in the FEATURES.md vocabulary:

### Launch With (v1.0 MVP)

All Must-priority FRs from PRD §8. Categorically:

- [x] Vault create / unlock / lock / auto-lock / change-master-password (FR-001…013)
- [x] All 5 record types: credential, api_key, secure_note, symmetric_key, asymmetric_key_pair (FR-014…025)
- [x] In-memory search after unlock (FR-022); encrypted tags Should-have (FR-026)
- [x] Password generator with configurable length / classes / exclude-ambiguous (FR-027…033)
- [x] AES key generation (128/192/256, base64+hex display) (FR-034…040)
- [x] RSA key pair generation (2048/3072/4096, default 3072) with OpenSSH/PEM/PKCS#8 export (FR-041…048)
- [x] Ed25519 key pair generation with OpenSSH/PKCS#8 export (FR-049…056)
- [x] Private-key exposure warning modal (FR-047, FR-055; §9.8)
- [x] Clipboard copy with countdown, configurable timeout, attempt-clear on timeout/lock/manual (FR-057…063)
- [x] Cross-platform builds for macOS + Windows; manual test matrix passes (§14.3)
- [x] AAD on every AES-GCM operation; AAD-tampering tests pass (§5.4, §14.1)
- [x] Sanitized logging; typed AppErrorCode DTO (§10.1, §12)
- [x] README with threat model, no-recovery warning, clipboard limitations, Go-memory limitations (§19)

### Add After Validation (v1.x potential)

These were considered and deferred. Each is a sensible v1.x candidate if the MVP ships and gets used.

- [ ] **Lock-on-system-sleep / lid-close** — PRD §13.4 calls this Should-have but not v0.1-required. Worth promoting to a v1.x release because it closes a real exposure window (laptop snapped shut → user walks away → vault still unlocked until idle timer fires).
- [ ] **OpenSSH private-key export for RSA** — PRD currently has it for Ed25519 (FR-054, Should). Adding RSA OpenSSH export improves drop-in compatibility with `~/.ssh/`.
- [ ] **Locked-state public-key ring** — PRD §2.9 explicitly defers this. Genuinely useful for SSH workflows (read out a fingerprint without unlocking).
- [ ] **Optional harmless-overwrite-then-clear of clipboard** (FR-064, Could) — A second wipe pass with junk data before clearing. Mitigates some clipboard-history scenarios. Cheap to add.
- [ ] **Vetted password-strength meter** (FR-033, Should) — zxcvbn or similar. Not Must in PRD; ship without if scope-tight.
- [ ] **CSV import** — Standard onboarding for users migrating from another manager. **Currently not in PRD.** Worth flagging: if anyone is going to actually use Abyss as their vault, they probably already have credentials elsewhere. This is a reasonable v1.x addition. Important constraint: import path must never leave the imported plaintext on disk after the import completes.
- [ ] **CSV/JSON export** (encrypted-zip wrapped, with strong warning) — backup affordance. The "no recovery" stance makes this more important, not less: users should be able to take their own offline backup.
- [ ] **Duplicate-master-password warning at creation** — if the user reuses a master password across vaults (which is a footgun), a "this password is in your clipboard / has been used recently" warning would help. Subtle UX feature; not MVP.

### Future Consideration (v2+)

- [ ] **DEK rotation** — `deks` table is already keyed for this (status = active/retired), but the rotation flow is not in MVP. v2 because it requires re-encrypting all records.
- [ ] **Multi-wrapping (e.g., add hardware-key unlock alongside master-password unlock)** — `key_wrappings` table is keyed for this. v2 because it pulls in hardware-key infrastructure (FIDO2/WebAuthn from a desktop app via Wails is non-trivial).
- [ ] **TOTP authenticator** — explicitly excluded by PRD §3.3 with sound rationale (co-locating second factor). Likely a permanent exclusion, not a v2 candidate.
- [ ] **SSH agent integration** — explicitly excluded by PRD §3.3. Real complexity (sockets, named pipes), real attack surface. Likely permanent exclusion.
- [ ] **Browser extension** — explicitly excluded. Permanent exclusion for this product.
- [ ] **Cloud sync** — explicitly excluded. Permanent exclusion for this product (fundamentally different product).
- [ ] **Linux packaging** — excluded for MVP only; reasonable v2 candidate once macOS + Windows are stable.

---

## Feature Prioritization Matrix

Subset of features where prioritization is informative (i.e., not strict-MVP):

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Encrypted tags (FR-026, Should) | MEDIUM | LOW (it's already in the metadata blob) | **P1** — basically free; ship it |
| Password strength meter via zxcvbn (FR-033, Should) | LOW–MEDIUM (informational) | MEDIUM (library integration + UX) | P2 |
| RSA public-key PEM export (FR-045, Should) | MEDIUM (TLS interop) | LOW | **P1** — cheap and high-interop |
| Ed25519 OpenSSH private-key export (FR-054, Should) | HIGH (drop into `~/.ssh/`) | MEDIUM | **P1** — primary SSH interop path |
| Manual clipboard clear button (FR-060, Should) | MEDIUM | LOW | **P1** — cheap and aligns with security posture |
| Lock-on-system-sleep (PRD §13.4, Should) | HIGH | MEDIUM (platform-specific event hooks) | P2 — promote to v1.x if scope-tight in v1.0 |
| Optional harmless-overwrite-then-clear clipboard (FR-064, Could) | LOW–MEDIUM | LOW | P3 |
| Recent-vaults list (PRD §9.2, optional) | LOW | LOW (paths only) | P3 |
| CSV import | MEDIUM (migration path) | MEDIUM | P3 — v1.x candidate |
| CSV/JSON encrypted-zip export | MEDIUM (user-controlled backup) | MEDIUM | P3 — v1.x candidate |

**Priority key:**
- **P1:** Worth shipping in MVP if effort is small (most "Should"s qualify here)
- **P2:** v1.x; valuable but not MVP-blocking
- **P3:** v2+; defer

---

## Competitor Feature Analysis

| Feature | KeePassXC | Bitwarden (self-host) | 1Password | Abyss (planned) |
|---------|-----------|-----------------------|-----------|-----------------|
| Storage model | Local KDBX file | Cloud-first, sync | Cloud-only (1P8) | **Local SQLite, no sync** |
| KDF | Argon2id (configurable) | PBKDF2-SHA256 (default 600k iter) | PBKDF2 + 2SKD | **Argon2id (256 MiB / 3 / 4 target)** |
| Per-record AEAD | Whole-DB encrypted | Per-record + sync | Per-record | **Per-record AES-256-GCM with AAD** |
| Master password change without re-encrypting all records | Re-encrypts whole DB | Yes (re-wraps account key) | Yes (re-wraps account key) | **Yes (re-wraps DEK)** |
| Auto-lock default | User-configurable | 15 min (varies) | "Never" with prompt | **10 min** |
| Clipboard timeout default | 10s | "Never" (criticized) | ~90s | **30s** |
| Password generator length default | 25 | ~14 | ~20 | **24** |
| AES key generation | No (attach as file) | No | No | **Yes, first-class record type** |
| RSA key generation in-app | No (KeeAgent serves but doesn't generate) | No | No | **Yes, first-class** |
| Ed25519 key generation in-app | No | No | No | **Yes, first-class** |
| OpenSSH key export | Via attachment | No | No | **Yes, native** |
| PKCS#8 private-key export | Via attachment | No | No | **Yes, native** |
| Browser autofill / extension | Yes | Yes | Yes | **No (out of scope)** |
| TOTP authenticator | Yes (built-in) | Yes (premium) | Yes | **No (out of scope)** |
| Cloud sync | No (3rd-party file sync) | Yes (server) | Yes (proprietary) | **No (out of scope)** |
| Recovery questions / escrow | No | Emergency access (premium) | Recovery key (printed) | **No (no recovery, by design)** |
| File attachments | Yes | Yes | Yes | **No (out of scope)** |
| Plugin system | Yes | No | No | **No (out of scope)** |

**Reading of the matrix:** Abyss strips a *lot* of feature surface area (sync, autofill, attachments, TOTP, plugins, recovery). What it adds is the cryptographic-key generation/export pipeline that none of the three commercial-quality reference products do natively. This is a coherent, defensible scope for a learning-grade product.

---

## Open Questions for Requirements Review

Things worth raising with the user during REQUIREMENTS.md generation. **None of these are scope-creep recommendations** — they are gaps in the explicit decision record.

1. **Should "remember last unlock" / "stay unlocked across launches" be explicitly listed as an anti-pattern in PROJECT.md Out of Scope?** Currently implicit only. A future contributor might propose it as a "UX improvement." Worth naming.
2. **Should telemetry / analytics be explicitly excluded?** Not currently in PRD §3.3. A vault should not phone home; making this explicit prevents "we'll just collect crash reports" creeping in.
3. **Should backup/export-of-vault be a planned v1.x feature, or deliberately omitted?** Given the "no recovery" stance, users will want a way to take an offline backup. Currently the only path is "copy the SQLite file." If that is the intended answer, it should be documented in the README.
4. **Should CSV import be on the post-MVP roadmap?** Most users adopting a new password manager have credentials elsewhere. Without an import path, the migration cost is "retype every credential by hand." Mark v1.x candidate or explicitly defer.
5. **Should password-generator strength meter (FR-033, Should) ship in MVP or defer?** zxcvbn integration is non-trivial. The PRD says use a vetted library *if any* — i.e., it's optional. Worth deciding now whether this is in MVP or v1.x.
6. **Should lock-on-system-sleep / lid-close (PRD §13.4, Should) be part of v1.0 or deferred to v1.x?** Currently flagged in PRD as Should-have but explicitly NOT required for v0.1. The decision point is whether v1.0 ("MVP Complete" milestone) requires it. Recommend: Yes — closes a real exposure window users will notice.
7. **Should the README clipboard warning be surfaced inside the app (Settings page, About screen) and not just in README?** Many users will never read the README. PRD §19 puts the warning in README only. Worth duplicating into Settings or About.
8. **Should "duplicate master password warning" (the user pasted their master password from clipboard) be a feature?** Subtle UX. Detects when the user is reusing a master password across vaults. Possibly v1.x.
9. **Public-key fingerprint visibility while locked** — PRD §2.9 acknowledges the trade-off but defers. Worth noting in PROJECT.md as a known v0.6+ candidate so it's not lost.
10. **Lockout / rate-limiting on repeated wrong-password attempts** — PRD does not specify. For a local-first product the threat model is offline brute-force *anyway* (attacker has the file), so in-app rate-limiting only helps against shoulder-surfers. Reasonable to omit, but worth an explicit decision.

---

## Sources

**Reference products and feature comparisons:**
- [KeePassXC User Guide](https://keepassxc.org/docs/KeePassXC_UserGuide) — clipboard timeout, auto-lock, password generator behavior
- [KeePassXC Documentation hub](https://keepassxc.org/docs/) — feature reference
- [KeePass main site](https://keepass.info/) — original product reference
- [KeePass Master Key documentation](https://keepass.info/help/base/keys.html) — multi-component master key model
- [Bitwarden vault timeout documentation](https://bitwarden.com/help/vault-timeout/) — auto-lock behavior
- [1Password unlock & auto-lock support article](https://support.1password.com/unlock-auto-lock/) — idle timeout reference
- [Bitwarden vs KeePassXC vs 1Password 2026 comparison (dasroot.net)](https://dasroot.net/posts/2026/03/bitwarden-keepassxc-1password-password-manager-comparison/) — high-level positioning
- [Cybernews 1Password vs Bitwarden 2026](https://cybernews.com/best-password-managers/bitwarden-vs-1password/) — feature comparison
- [Cybernews KeePass vs 1Password 2026](https://cybernews.com/best-password-managers/keepass-vs-1password/)
- [TechSpot: change your password manager's clipboard settings](https://www.techspot.com/news/97320-you-change-password-manager-clipboard-settings-now.html) — clipboard-timeout defaults across products

**Cryptographic standards and best practices:**
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — Argon2id minimum parameters (19 MiB / 2 / 1)
- [NIST SP 800-57 Part 3 Rev 1](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt3r1.pdf) — RSA key size recommendations
- [NIST SP 800-78-5 (PIV cryptographic algorithms and key sizes)](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-78-5.pdf) — RSA 2048/3072/4096 acceptability windows
- [Fastly: Key size for TLS](https://www.fastly.com/blog/key-size-for-tls) — practical RSA size discussion (3072 vs 4096 cost/benefit)
- [OWASP Choosing and Using Security Questions Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Choosing_and_Using_Security_Questions_Cheat_Sheet.html) — why security questions are anti-features
- [The Hacker News: 25 password recovery attacks against major password managers (Feb 2026)](https://thehackernews.com/2026/02/study-uncovers-25-password-recovery.html) — recent research on recovery-mechanism vulnerabilities
- [arxiv: Evaluating Argon2 Adoption and Effectiveness](https://arxiv.org/html/2504.17121v1) — empirical Argon2 deployment study

**SSH key storage in password managers (validates Abyss's differentiator):**
- [Mendhak: KeePass and KeeAgent for SSH](https://code.mendhak.com/keepass-and-keeagent-setup/) — KeePass treats keys as attachments
- [How to use KeePassXC with ssh-agent](https://blog.valouille.fr/post/2018-03-27-how-to-use-keepass-xc-with-ssh-agent/) — KeePassXC SSH integration approach

**PRD source:**
- `/Users/weldon/projects/abyss/docs/PRD.md` v1.2 — Architecture-Locked MVP Build Contract
- `/Users/weldon/projects/abyss/.planning/PROJECT.md` — Active requirements

---

*Feature research for: Local-first desktop password & cryptographic key vault*
*Researched: 2026-04-29*
*Confidence: HIGH (PRD scope is explicitly fixed; this research validates against industry norms and surfaces gaps)*
