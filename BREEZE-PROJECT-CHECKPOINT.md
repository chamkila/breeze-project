# Breeze — Complete Project Checkpoint v2
**Updated:** 2026-03-26 | **Covers:** All sessions (1–44, updated after S44 results) across all conversation threads
**Purpose:** Single document to resume any thread with full context. Paste into project knowledge or hand to a new session.

---

## IDENTITY

**Breeze** — Native macOS SFTP client. Not another FTP tool. A remote filesystem platform for power users. Tagline: "Files flow effortlessly."

- **Developer:** Jagjit Singh (j) — Singh Digital, Inc.
- **Repo:** github.com/chamkila/breeze (private)
- **libbrz Repo:** github.com/chamkila/libbrz (private)
- **Local paths:** ~/Projects/breeze/ (Breeze.app), ~/Projects/libbrz/ (C library)
- **Branch:** `feature/single-pane-redesign`
- **Tags:** `v0.1.0-beta1` (safe fallback on main), `v0.2.0-beta1` (first bidirectional transfer)
- **Linear:** https://linear.app/chamkila/project/breeze-ftp-a885104b1f5c
- **Domains:** coolbreeze.cloud (product + relay), flybrz.com (short URL), feelbreeze.dev (developer docs)
- **Copyright:** Singh Digital, Inc. across all files

---

## PROJECT ORIGIN (Session 1)

Breeze was born from prior FTP client discussions consolidated into a single vision. In Session 1, the project was named, a 28KB BREEZE-SPEC.md was written, the ACE FRAME-inspired plugin architecture was designed in Swift (BreezeDataSource protocol, TransferEngine actor, PluginManager), the repo was created on GitHub, and the first commit landed (04fddc9). The architecture draws from j's professional experience building a telecom mediation server (MDS) that ran 24/7/365 — the jacket pattern (elmgr/datamgr with numbered plugin configs), IPC bus (cwmon), and zero-downtime operations are core inspirations.

---

## CORE ARCHITECTURE

### BreezeTransfer Protocol (BTP) — SS7-Inspired 5-Layer Stack

```
Layer 5: APPLICATION     SwiftUI, Cmd+K, TransferQueue
Layer 4: INTELLIGENCE    ConflictResolver, Predictor, FTDR, NetworkProbe
Layer 3: MULTIPLEXER     BTPLinkManager (SS7 MTP Level 3) — signaling/bearer separation
Layer 2: TRANSPORT       BTPLink wrapping transport backends, async AIO
Layer 1: PHYSICAL        Transport backends + TCP sockets
```

**Key rule:** Signaling link (browsing) is a SEPARATE TCP connection from bearer links (transfers). Signaling NEVER waits for bearer. Each BTPLink.open() creates a NEW SSH connection. No layer above Layer 1 touches crypto. No layer above Layer 2 touches sockets.

### FRAME Pattern — Thin Jacket Architecture

Intelligence lives in Core and Plugins. Each layer independently updatable. Inspired by the MDS elmgr/datamgr jacket pattern.

### Transport Backends

1. **BrzTransport** (pri: 300) — libbrz.a static library, PRIMARY AND ACTIVE
2. **OpenSSHTransport** (pri: 10) — DISABLED (BreezeEngineOnly=true)
3. **LibSSH2Transport** (pri: 100) — STUBBED OUT without permission (J-40)
4. **FTPTransport** (pri: 80) — URLSession-based, filtered out for SFTP

### TransferHandler Architecture (THE BREAKTHROUGH — Session 29)

SwiftUI struct recreation makes closures stale. Fix: `TransferHandler` as `@Observable @MainActor final class`. ALL transfer callbacks use this. Never struct closures again.

---

## libbrz — THE SSH/SFTP ENGINE

Pure C11 SSH/SFTP library. Zero external dependencies. Crypto vendored from public-domain sources.

### Evolution Timeline

| Session | Version | Milestone |
|---------|---------|-----------|
| ~30 | v0.3.0-ssh | Initial scaffolding, 249 tests |
| ~34 | v1.0.0 | **FIRST SFTP CONNECTION** — curve25519-sha256, aes256-ctr, RSA agent auth. 607 tests |
| 35 | v0.4.2 | Connection pool, folder upload fix, error code collision fix |
| 41 | v0.4.3 | BBR adaptive engine, server profile cache, brz_probe() |
| 43e | — | AES-256-GCM working, atomic sftp_send_packet, cipher failure API. 651 tests |
| 44 | — | NEON AES-256 hw accel re-enabled (600 MB/s CTR). Breeze UI fixes. 660 tests |

### Current State (Session 44)

- **Tests:** 660, 0 warnings, 0 external dependencies
- **Working ciphers:** aes256-ctr + hmac-sha2-256, aes-256-gcm (software GHASH)
- **Broken ciphers:** chacha20-poly1305 send path (parked)
- **KEX:** curve25519-sha256 only
- **Auth:** RSA via ssh-agent (rsa-sha2-256), Ed25519 pubkey from file
- **SFTP:** v3, brz_ls, brz_put (pipelined), brz_get, brz_mkdir, brz_mkdir_p
- **Adaptive engine:** BBR 4-phase (STARTUP/DRAIN/PROBE_BW/PROBE_RTT), BDP-based
- **Connection pool:** CAS borrow/return, signaling/bearer separation, maintenance thread

### Performance State

- **AES-256-CTR:** ~600 MB/s on Apple Silicon (NEON hw) — WORKING
- **AES-256-GCM:** ~17 MB/s (software GHASH bottleneck) — NEON PMULL is NEXT PRIORITY
- **End-to-end transfer:** ~1.4 MB/s for large folders (per-file SSH overhead)
- **Target:** 20+ MB/s LAN (beat FileZilla's 13 MB/s)
- **Known bottleneck:** Per-file SSH connection overhead. One-handle architecture needed

### Architecture Rules (NON-NEGOTIABLE)

ALL intelligence in libbrz. Pure C11. brz_ prefix. Zero deps. Software fallback always exists.

### Implementation Roadmap (J-62)

Phase 0 (DONE) → Phase 1: SFTP extensions → Phase 2: Resume, parallel → Phase 3: Sync → Phase 4: CLI/REST → Phase 5: Encryption, jump hosts

---

## BREEZE.APP — CURRENT STATE

### What Works

SFTP via libbrz, navigation, file browser, SSH config import (24 hosts), SSH Key viewer, connection form, Cmd+K palette, single-pane + local drawer, density modes, transcript, 5 themes, context menus, DMG packaging, upload (toolbar + drag-drop), download (drag-drop), delete, new folder (recursive), Quick Look, transfer celebration, shimmer, BTP, folder upload (283 files verified).

### Open UI Bugs

1. ~~Sidebar click unresponsive~~ — FIXED S44 (.simultaneousGesture)
2. ~~Transfer progress "Zero KB/s"~~ — FIXED S44 (fallback calc)
3. ~~File list doesn't auto-scroll~~ — FIXED S44 (.id(currentPath))
4. ~~File icons generic~~ — FIXED S44 (30+ extensions → SF Symbols)
5. Sidebar text contrast too low (partially fixed — vibrancy override)
6. "Drifting to..." loading full-screen takeover
7. Settings button not wired on welcome screen

---

## CRITICAL RULES

- **Dennis Ritchie way:** Fix root causes. No shortcuts, stubs, or workarounds
- **"In love code"** — meticulous, passionate, crafted
- NEVER remove/stub working code without discussion
- ALL Process() calls in Task.detached. BreezeColors for ALL colors. os.Logger not print()
- Build and test after EVERY task. Test count NEVER decreases
- GUI: Depth, Restraint, Typography hierarchy, Negative space, Meaningful motion
- UI is #1 priority. Apple frameworks as foundation. Keychain for ALL credentials

---

## PRODUCT VISION

$14.99 one-time (Sublime Text model). Students free. Joy features are PRIMARY (music, games, celebrations, wellness). AI: Rules Engine → CoreML → optional LLM. FTDR: telecom CDR model, NDJSON, 90-day retention.

## MILESTONES

M1 (GUI, Apr 7) → M2 (Function, May 1) → M3 (Performance, Jun 1) → M4 (BreezeFS+CLI, Aug 1) → M5 (Launch, Sep 1). Never work on M(N+1) while M(N) bugs exist.

## VERSIONING

No premature tags. 0.1.0-beta.N for milestones. 0.1.0-rc1 = "ਵਖਤ ਦੇ ਪਲਾਂ ਦੀ ਕੈਦ" (locked in).

## COMPETITIVE MOAT

Own SSH engine (libbrz) · BTP dual-channel (SS7) · BBR adaptive · BreezeFS (no macFUSE) · FTDR audit · 10 speed techniques · $14.99 one-time. Competitors all use macFUSE/Java/single-channel.

## SESSION WORKFLOW

Personal Mac + Claude Code = execution. Any Mac + Claude.ai = strategy. `AUTONOMOUS MODE. Read CLAUDE.md + Design-Principles.md first.`

## KEY DOCS (~/Projects/breeze/Docs/)

Protocol-Layered-Architecture.md · BTP-Protocol-Spec.md · Implementation-Roadmap.md · Design-Principles.md · Interaction-Philosophy.md · BREEZE-SPEED-RESEARCH.md · Code-Philosophy.md · Case-Study-AppKit-SwiftUI-Drag-Drop.md · Feature-Registry.md · COMPETITIVE-ANALYSIS.md

## DESIGN INSPIRATION

GUI: Linear, Raycast, Things 3, Arc, Fantastical, Apogee. Colors: Xcode dark theme pastels. Animation: Claude Code shimmer, Swift Student Challenge. Loading UX: Transmit pattern.

---

## COMPANION DOCUMENTS (same directory)

| Document | Size | What it answers |
|----------|------|-----------------|
| **DEBUGGING-JOURNAL.md** | ~9KB | "What was the root cause of X?" — one line per bug, 45+ entries |
| **SESSION-LOG.md** | ~6KB | "When did we build X?" — one paragraph per session |
| **DECISIONS-LOG.md** | ~7KB | "Why did we choose X over Y?" — rationale for every architectural decision |

*All four documents live in ~/Projects/breeze-project/. Update after each session.*