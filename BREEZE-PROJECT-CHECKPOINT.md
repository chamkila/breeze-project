# Breeze — Complete Project Checkpoint v2
**Updated:** 2026-03-26 | **Covers:** All sessions (1–44) across all conversation threads
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

BTP was designed by studying actual SS7 source code (OpenSS7 mtp3.cpp state machines, Extended-jSS7 M3UA As.java application server pattern). Five Swift files implement it: BTPServiceIndicator, BTPState, BTPLink, BTPLinkManager, BTPTransferRequest. The BTPServiceIndicator routes messages by priority (management > signaling > interactive > bearer > bulk), mirroring SS7's SI field.

### FRAME Pattern — Thin Jacket Architecture

Intelligence lives in Core and Plugins. Each layer independently updatable. Inspired by the MDS elmgr/datamgr jacket pattern where the same binary reads different config sections to become different processing pipelines.

### Transport Backends (priority order)

1. **BrzTransport** (pri: 300) — libbrz.a static library, PRIMARY AND ACTIVE
2. **OpenSSHTransport** (pri: 10) — Process-based fallback, DISABLED (BreezeEngineOnly=true)
3. **LibSSH2Transport** (pri: 100) — STUBBED OUT without permission (J-40, needs restore from git)
4. **LibSSHTransport** — STUBBED OUT without permission (needs restore from git)
5. **FTPTransport** (pri: 80) — URLSession-based, filtered out for SFTP connections

**Current state:** BreezeEngineOnly=true. All connections go through libbrz. OpenSSH fallback is disabled.

### TransferHandler Architecture (THE BREAKTHROUGH — Session 29)

**Root cause found:** SwiftUI structs are recreated constantly. Closures captured on structs become stale. Drag-drop events fire from AppKit AFTER SwiftUI recreates the struct → closure points to dead copy.

**Solution:** `TransferHandler` is an `@Observable @MainActor final class` (reference type). All access goes to the same live object.

**Rule:** ALL transfer callbacks use TransferHandler. Never use struct closure callbacks for transfers again.

---

## libbrz — THE SSH/SFTP ENGINE

### What Is libbrz

A pure C11 SSH/SFTP client library with zero external dependencies. Built by studying public-domain crypto primitives from tinyssh, dropbear, and openssh-portable — concepts and algorithms only, no code copied.

### Evolution Timeline

| Session | Version | Milestone |
|---------|---------|-----------|
| ~30 | v0.3.0-ssh | Initial scaffolding: SSH packet framing, crypto primitives vendored, 249 tests |
| ~34 | v1.0.0 | **FIRST COMPLETE SFTP CONNECTION** — curve25519-sha256, aes256-ctr, RSA agent auth, SFTP v3. 607 tests |
| 35 | v0.4.2 | Connection pool (CAS borrow/return), folder upload fix (brz_mkdir_p), error code collision fix |
| 41 | v0.4.3 | BBR-inspired adaptive transfer engine, server profile cache, brz_probe() API |
| 41c | — | tune.c/tune.h extracted and tested. 646 tests |
| 43e | — | AES-256-GCM end-to-end working, atomic sftp_send_packet, cipher failure API. 651 tests |

### Current State (Session 43e)

- **Tests:** 651, 0 warnings, 0 external dependencies
- **Working ciphers:** aes256-ctr + hmac-sha2-256, aes-256-gcm (software)
- **Broken ciphers:** chacha20-poly1305 send path (recv works — parked after exhaustive investigation)
- **KEX:** curve25519-sha256 only (no unimplemented algorithms offered)
- **Auth:** RSA via ssh-agent (rsa-sha2-256), Ed25519 pubkey from file
- **SFTP:** v3, brz_ls, brz_put (pipelined), brz_get, brz_mkdir, brz_mkdir_p
- **Adaptive engine:** BBR-inspired 4-phase (STARTUP/DRAIN/PROBE_BW/PROBE_RTT), BDP-based chunk/depth
- **Connection pool:** CAS borrow/return, signaling/bearer separation, maintenance thread
- **Status bar shows:** `implicit · AES-256-GCM · via libbrz · BTP`

### Performance State

- **Current:** ~1.4 MB/s for large folders (dmp/, 283 files, 1.05 GB)
- **Software GCM:** 41.8s for dmp/ | **Software CTR:** 26s for dmp/
- **Target:** 20+ MB/s LAN (beat FileZilla's 13 MB/s)
- **NEON hw acceleration:** Attempted S43, disabled — AESE round key mismatch. NEXT PRIORITY
- **Known bottleneck:** Per-file SSH connection overhead. One-handle architecture needed

### Architecture Rules (NON-NEGOTIABLE)

- ALL intelligence in libbrz. Breeze/CLI/REST are thin jackets
- No hardcoded values — BBR model measures and adapts
- Pure C11. brz_ prefix. Zero external dependencies
- Software fallback for everything — never platform-lock

### Implementation Roadmap (Docs/Implementation-Roadmap.md, J-62)

Phase 0: Transfers work (DONE) → Phase 1: SFTP extensions → Phase 2: Resume, parallel → Phase 3: Sync, tree ops → Phase 4: BreezeCLI/REST → Phase 5: Encryption, jump hosts. Every feature ships in 3 channels: libbrz, Breeze.app, BreezeCLI/REST.

---

## BREEZE.APP — CURRENT STATE

### What Works (Verified by Human QA)

SFTP connection/browsing via libbrz, directory navigation with breadcrumbs, file browser (sizes/permissions/dates), SSH config auto-import (24 hosts), SSH Key viewer, connection form, Cmd+K palette, single-pane + local drawer (Cmd+L), density modes (Cmd+1/2/3), transcript drawer, 5 themes (Breeze/Midnight/Dusk/Forest/Mono), context menus, about window, DMG packaging, upload (toolbar + drag-drop), download (drag-drop), delete, new folder (recursive), Quick Look, transfer celebration sound, shimmer animation, BTP signaling/bearer, folder upload (283 files verified).

### Open UI Bugs (prioritized)

1. Sidebar click unresponsive — gesture conflict or invisible overlay
2. Transfer progress "Zero KB/s" — speed hardcoded to 0
3. File list doesn't auto-scroll on directory change
4. File icons generic — all same icon regardless of type
5. Sidebar text contrast too low (partially fixed — vibrancy override)
6. "Drifting to..." loading full-screen takeover
7. Settings button not wired on welcome screen

### Build Sessions Summary

| Sessions | Tests After | Highlights |
|----------|-------------|------------|
| 1 | ~40 | Xcode project, SwiftUI shell, libssh2 plugin |
| 2-3 | 114 | Refactor, SessionPool, TransferQueue, FTP plugin, robustness |
| 4-7 | ~427 | Multiplexer, NetworkProbe, transfers, search, key gen, profiles |
| 8-9 | ~1052 | BreezeVault, SSHFS, prefetch, archive VFS, smart queries |
| 10-12 | 1530 | BTP wired, keyboard ops, pipelining, error handling |
| 13+ | 503+ | Bug bash, transport refactoring (test count decreased during major changes) |

---

## CRITICAL RULES

### Engineering (NON-NEGOTIABLE — all projects, all chats)

- **Dennis Ritchie way:** Fix root causes. No shortcuts, workarounds, or stubs. Buy once, cry once
- **"In love code"** — meticulous, passionate, crafted. Not "good enough"
- **Never normalize bugs** in new code

### Code (enforced via CLAUDE.md)

- NEVER remove/stub working code without discussion
- ALL transfer callbacks use TransferHandler @Observable class
- ALL Process() calls in Task.detached — NEVER on main thread
- BreezeColors for ALL colors. os.Logger not print(). No force unwraps except tests
- Build and test after EVERY task. Test count NEVER decreases

### GUI Quality Gates (Docs/Design-Principles.md — MANDATORY)

1. Depth (glass/blur) 2. Restraint (minimal color) 3. Typography hierarchy 4. Intentional negative space 5. Meaningful motion

### Product

- UI is #1 priority. "Touch water, it ripples" — minimum gesture, maximum result
- Don't reinvent Apple's tools — orchestrate them
- Apple frameworks as foundation. Never depend on competitor's library
- Keychain for ALL credential storage

---

## PRODUCT VISION

### Business Model (Sublime Text approach)

$14.99 one-time. Students free. Fully functional forever, gentle reminder after 30 days. Registration key intentionally crackable. Direct DMG + Homebrew Cask.

### Joy Features (PRIMARY features, not extras)

Ambient music during transfers, SSH key generation mini-games, wellness nudges (20-20-20), transfer celebrations, Mission Control dashboard, elastic window detachment, Breeze Landscape, Knowledge Cards, text shimmer/sparkle like Claude Code terminal.

### AI Strategy: Rules Engine (always on) → CoreML on FTDR → Optional LLM (Ollama/Claude API)

### FTDR: Every transfer = immutable record (telecom CDR model). NDJSON, 90-day retention, ~500 bytes/record.

---

## MILESTONES

M1 (GUI, Apr 7) → M2 (Function, May 1) → M3 (Performance, Jun 1) → M4 (BreezeFS+CLI, Aug 1) → M5 (Launch, Sep 1)

**Rule:** Never work on M(N+1) while M(N) bugs exist.

M3 depends on: NEON AES acceleration + one-handle architecture fix.
M4 includes: BreezeFS (zero-dep mount), BreezeRelay (coolbreeze.cloud), BreezeFlow (canvas pipelines), BreezeCLI.
M5: Product Hunt → HN → Reddit → Twitter benchmark video.

---

## VERSIONING

No premature tags. Dev builds by date/commit. 0.1.0-beta.N for milestones. 0.1.0-rc1 = "ਵਖਤ ਦੇ ਪਲਾਂ ਦੀ ਕੈਦ" (locked in). Existing tags stay as history.

---

## COMPETITIVE MOAT

Own SSH engine (libbrz, pure C, zero deps) · BTP dual-channel (SS7 heritage) · BBR adaptive engine · BreezeFS planned (no macFUSE) · FTDR telecom-grade audit · 10 speed techniques (no server agent) · $14.99 one-time.

Competitors: ExpanDrive/Transmit → macFUSE. Cyberduck → Java. All single-channel. None own their SSH engine.

---

## SESSION WORKFLOW

Personal Mac + Claude Code = execution. Any Mac + Claude.ai = strategy. Never mix.
Start: `git pull`. End: `git push`. Claude Code: `AUTONOMOUS MODE. Read CLAUDE.md + Design-Principles.md first.`

---

## KEY DOCS (in ~/Projects/breeze/Docs/)

Protocol-Layered-Architecture.md · BTP-Protocol-Spec.md · Implementation-Roadmap.md · Design-Principles.md · Interaction-Philosophy.md · BREEZE-SPEED-RESEARCH.md · Code-Philosophy.md · Case-Study-AppKit-SwiftUI-Drag-Drop.md · Feature-Registry.md · COMPETITIVE-ANALYSIS.md

---

## CONVERSATION THREADS

1. **"Building a next-generation SFTP client"** (ended 2026-03-15) — Sessions 1-13+. Project creation, overnight mega builds (40→1530 tests), GUI polish, themes, QA
2. **"Breeze SFTP single-pane redesign progress"** (ended 2026-03-17) — P-013 planning, sidebar fixes, Termius competitive analysis
3. **"Sending remaining mixed up images"** (ended 2026-03-19) — BTP/SS7 deep dive, libbrz creation, crypto vendoring, BrzTransport, shimmer effects, icon design
4. **"Breeze GUI development approach"** (ended 2026-03-25) — libbrz breakthrough (v1.0.0), KEX debugging, AES-GCM, folder upload, performance analysis, pool architecture
5. **"Breeze upload performance"** (current) — BBR engine, tune extraction, NEON AES attempt, GCM working, cipher failure API, session 44 prompt

---

## DESIGN INSPIRATION

GUI: Linear, Raycast, Things 3, Arc, Fantastical, Apogee. Colors: Xcode dark theme pastels. Animation: Claude Code shimmer, Swift Student Challenge (FocusFish, Hanafuda). Loading UX: Transmit pattern.

---

## COMPANION DOCUMENTS (same directory)

| Document | Size | What it answers |
|----------|------|-----------------|
| **DEBUGGING-JOURNAL.md** | ~8KB | "What was the root cause of X?" — one line per bug, 40+ entries |
| **SESSION-LOG.md** | ~5KB | "When did we build X?" — one paragraph per session |
| **DECISIONS-LOG.md** | ~7KB | "Why did we choose X over Y?" — rationale for every architectural decision |

*All four documents live in ~/Projects/breeze-project/. Update after each session.*