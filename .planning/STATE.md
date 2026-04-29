---
gsd_state_version: 1.0
milestone: v0.1
milestone_name: Dogfoodable Vault
status: planning
stopped_at: Phase 1 context gathered
last_updated: "2026-04-29T14:49:36.441Z"
last_activity: 2026-04-29 — Roadmap created from PRD §15 milestones; 115/115 v1 requirements mapped
progress:
  total_phases: 6
  completed_phases: 0
  total_plans: 0
  completed_plans: 0
  percent: 0
---

# Project State

## Project Reference

See: `.planning/PROJECT.md` (updated 2026-04-29)
Authoritative build contract: `docs/PRD.md` v1.2

**Core value:** A user can create an encrypted local vault with a master password, store and retrieve secrets, and never see plaintext sensitive data persisted to disk. Vault confidentiality and tamper-evidence (DEK/KEK envelope encryption with AES-256-GCM AAD), end-to-end, on macOS and Windows.
**Current focus:** Phase 1 — v0.1 Dogfoodable Vault

## Current Position

Phase: 1 of 6 (v0.1 Dogfoodable Vault)
Plan: 0 of TBD in current phase
Status: Ready to plan
Last activity: 2026-04-29 — Roadmap created from PRD §15 milestones; 115/115 v1 requirements mapped

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity:**

- Total plans completed: 0
- Average duration: —
- Total execution time: 0.0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1. v0.1 Dogfoodable Vault | 0/TBD | — | — |
| 2. v0.2 Security Foundation | 0/TBD | — | — |
| 3. v0.3 Expanded Record Types | 0/TBD | — | — |
| 4. v0.4 Key Generation and Export | 0/TBD | — | — |
| 5. v0.5 Cross-Platform Hardening | 0/TBD | — | — |
| 6. v1.0 MVP Complete | 0/TBD | — | — |

**Recent Trend:** N/A — no plans executed yet.

*Updated after each plan completion.*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table (frozen by PRD §2 and §18).
Recent decisions affecting current work:

- Phase 1: AAD ships in the FIRST `cipher.Seal` call, not deferred to Phase 2 (PRD §15 lists "AAD enforcement" as a v0.2 EXIT CRITERION but research confirms it must be PRESENT from v0.1 — retrofitting AAD requires re-encrypting every record).
- Phase 1: REC-13 (encrypted tags) and CLIP-04 (manual clear button) promoted from PRD Should-have to v0.1 per research findings (both are cheap and high-value).
- Phase 1: All cross-cutting scaffolding (`internal/aad/`, `internal/apperr/`, `internal/session/`, `internal/logger/`, `frontend/src/api/`, migration fail-closed, CI lint rules) lands in v0.1 — not retrofittable later.
- Phase 1: SQLite plaintext-grep test runs in CI on every PR, not only at milestone boundaries.
- Phase 1: Cross-OS smoke test required at end of every milestone (PRD §2.3); not deferred to Phase 5.

### Pending Todos

[From `.planning/todos/pending/` — ideas captured during sessions]

None yet.

### Blockers/Concerns

[Issues that affect future work]

- **Phase 1 research flag (MEDIUM confidence):** Confirm exact import path for the pure-Go SQLite database driver in `golang-migrate/v4` (`database/sqlite` vs `database/sqlite3`). After wiring, run `go mod why github.com/mattn/go-sqlite3` to confirm CGO has not crept in. Resolve at Phase 1 plan-research time.
- **Phase 4 research flag (MEDIUM confidence):** Verify `golang.org/x/crypto/ssh.MarshalPrivateKey` availability at the chosen `x/crypto` module version BEFORE planning ED-06 (Ed25519 OpenSSH private-key export). If unavailable, defer ED-06 to post-v1.0 and document in README.
- **Phase 5 research flag:** System-sleep detection (VAULT-13) requires platform-specific APIs (IOKit on macOS, `WM_WTSSESSION_CHANGE` on Windows). Research exact Wails/OS hook approach at Phase 5 plan-research time; if infeasible in MVP, document and defer with Settings warning.
- **Phase 1 Mantine version decision:** v8.3.18 is the recommendation (research HIGH confidence); v9 just shipped (~6 weeks before MVP planning). Re-evaluate at v1.0 (Phase 6) or post-MVP. Migrate at a milestone boundary if at all.

## Deferred Items

Items acknowledged and carried forward (none from prior milestones since this is initialization):

| Category | Item | Status | Deferred At |
|----------|------|--------|-------------|
| *(none)* | | | |

## Session Continuity

Last session: 2026-04-29T14:49:36.406Z
Stopped at: Phase 1 context gathered
Resume file: .planning/phases/01-v0-1-dogfoodable-vault/01-CONTEXT.md
