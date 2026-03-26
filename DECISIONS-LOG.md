# Breeze — Decisions & Rationale Log
**Purpose:** WHY we chose X over Y. When a future session questions a decision, check here before relitigating.

---

## Architecture Decisions

### AD-001: Own SSH engine (libbrz) over using libssh/libssh2/OpenSSH
**When:** Session ~30 (Thread 3)
**Chose:** Build libbrz from scratch in pure C
**Over:** Using libssh2 (BSD, working) or libssh (LGPL) or NIOSSH (Swift)
**Why:** Zero dependencies = smaller attack surface. Full control over pipelining, adaptive chunking, BBR. No license complications (libssh is LGPL). Can optimize for exact Breeze use case. Cross-platform (C runs everywhere). libssh2 was working but limited — no async pipelining by design, hidden accidental pipelining only
**Risk accepted:** Massive engineering effort. Every crypto bug is ours to fix
**Status:** Validated. libbrz connects, transfers, 651 tests

### AD-002: BreezeEngineOnly=true (disable OpenSSH fallback)
**When:** Session 43e (Thread 5)
**Chose:** All connections through libbrz only
**Over:** Keeping OpenSSH Process-based fallback active
**Why:** Can't benchmark or optimize libbrz if traffic silently falls back to OpenSSH. Need to find and fix libbrz issues, not hide them. All performance testing must measure the actual engine
**Risk accepted:** If libbrz fails to connect to a server, the user has no fallback. Acceptable during development — will re-evaluate for release
**Revisit when:** Beta testing with diverse servers

### AD-003: NEON intrinsics over CommonCrypto for AES acceleration
**When:** Session 43 (Thread 5)
**Chose:** Direct ARM NEON intrinsics (vaeseq_u8, vaesmcq_u8, vmull_p64)
**Over:** Apple's CommonCrypto (CCCryptorCreateWithMode) which uses hardware internally
**Why:** Dennis Ritchie way — understand every instruction. CommonCrypto is a black box. NEON intrinsics give full control, visibility into every operation, cross-platform ARM64 (not macOS-only). Zero dependencies principle
**Risk accepted:** Must get AESE round key schedule exactly right (session 43 got it wrong). CommonCrypto would "just work" but violates the zero-dependency philosophy
**Status:** Pending — session 44

### AD-004: BTP signaling/bearer separation (SS7 model)
**When:** Sessions in Thread 3
**Chose:** Separate TCP connections for browsing (signaling) and transfers (bearer)
**Over:** Single-connection multiplexing like all competitors
**Why:** Browsing should NEVER stall because a transfer is saturating the pipe. SS7 solved this in the 1970s — control plane and data plane on separate links. Real-world benefit: user browses files while 1GB upload runs in background
**Risk accepted:** More SSH connections = more handshake overhead. Mitigated by connection pool with CAS borrow/return
**Status:** Working. Each BTPLink.open() creates a NEW SSH connection

### AD-005: SwiftData over CoreData/SQLite/UserDefaults for persistence
**When:** Session 1-2 (Thread 1)
**Chose:** SwiftData (Apple's modern persistence)
**Over:** CoreData (mature but verbose), raw SQLite, UserDefaults (flat)
**Why:** Native Swift integration, no boilerplate, automatic migration, SwiftUI binding. Breeze is macOS 14+ so SwiftData is available
**Status:** Working. SavedConnection is a SwiftData @Model

### AD-006: $14.99 one-time (Sublime Text model) over subscription
**When:** Session 1 (Thread 1)
**Chose:** One-time purchase, fully functional forever, gentle nag after 30 days
**Over:** Monthly/yearly subscription, freemium with feature gates
**Why:** Below every competitor (Forklift $30, Transmit $45). Students free (honor system). Registration key intentionally crackable — "the hacking IS the advertisement." Developer tools should be bought once
**Status:** Decision locked

### AD-007: Task.detached for all Process() calls
**When:** Session 2-3 (Thread 1), reinforced Session 35
**Chose:** ALL Process() and blocking I/O calls in Task.detached
**Over:** Running on MainActor or default executor
**Why:** Session 35 beachball: sequential mkdir calls blocked MainActor for 26 seconds. Process() spawns shell commands that can block indefinitely. Task.detached ensures MainActor is never held. Reinforced as CLAUDE.md rule after beachball incident
**Status:** Enforced. Every violation is a bug

### AD-008: TransferHandler as @Observable class (not struct callbacks)
**When:** Session 29 (Thread 2/3)
**Chose:** @Observable @MainActor final class for all transfer callbacks
**Over:** Struct closures, Combine publishers, delegation
**Why:** SwiftUI struct recreation makes closures stale. AppKit drag-drop fires after struct recreation. Reference type = same object always. This was THE breakthrough that made drag-drop work after weeks of debugging
**Risk:** None — this is objectively correct for SwiftUI+AppKit interop
**Status:** Rule. Case study in Docs/. Will never change

### AD-009: Apple Keychain for ALL credential storage
**When:** Session Thread 3
**Chose:** KeychainCredentialStore via Security.framework
**Over:** UserDefaults, plists, custom encrypted storage, any third-party vault
**Why:** Hardware-backed on Apple Silicon (Secure Enclave). iCloud Keychain sync. Apple maintains it. Best-in-class security we don't have to build. Passwords app integration on macOS
**Status:** Rule. Non-negotiable

### AD-010: Versioning — no premature tags
**When:** Session Thread 5
**Chose:** Dev builds tracked by date/commit. Tags only at real milestones
**Over:** Tagging every session (v1.0.0, v0.4.3, etc.)
**Why:** Early tags (v1.0.0 when it first connected) created confusion — is this release-quality? No. Dev builds are commits. 0.1.0-beta.N for milestones. Existing tags stay as history but new ones follow the policy
**Status:** Active

### AD-011: 5 premium themes (not user-customizable)
**When:** Sessions 13+ (Thread 1)
**Chose:** 5 curated themes: Breeze (default), Midnight (dark), Dusk (warm), Forest (green), Mono (B&W)
**Over:** User-customizable colors, unlimited themes, system-only appearance
**Why:** Curated quality. Every theme is designer-picked to work perfectly. User-customizable means ugly combinations. Apple's own apps (Xcode, Terminal) offer curated themes. BreezeColors delegation means one-click transform of entire app
**Status:** Shipped. All 5 working

### AD-012: Joy features as PRIMARY, not extras
**When:** Thread 1, reinforced Thread 3
**Chose:** Music, games, celebrations, wellness nudges are CORE product identity
**Over:** Treating them as post-launch polish or Easter eggs
**Why:** Inspired by Swift Student Challenge winners (FocusFish, Hanafuda). The emotional connection IS the product. "Every line of code must serve the mesmerizing, blissful, joy to work with standard." These features are what make people stop scrolling on the website
**Status:** Designed, not built. Must ship with M1/M2, not deferred to M4

---

## Technical Decisions (libbrz)

### TD-001: Pure C11 (not C++, not Rust, not Swift)
**Why:** Maximum portability. Builds with `make`. No runtime. Links into anything. The SSH RFCs are byte-level specs that map naturally to C. Rust would add build complexity. Swift would tie to Apple ecosystem. C++ would add complexity without benefit for this domain

### TD-002: brz_ prefix for all public symbols
**Why:** C has no namespaces. brz_ prefix prevents collision with any other library. Every public function, type, enum, macro starts with brz_ or BRZ_

### TD-003: BBR-inspired adaptive engine (not static tuning)
**Why:** No hardcoded chunk sizes or pipeline depths. Every network is different. LAN vs WAN vs satellite — the engine measures BtlBw and RTprop and adapts. Google's BBR research proved this approach. 4-phase state machine (STARTUP/DRAIN/PROBE_BW/PROBE_RTT) maps cleanly to SSH/SFTP transfer lifecycle

### TD-004: Server profile cache (~/.brz/profiles)
**Why:** File-based, no IPC needed. When connecting to a known server, load last-known BtlBw/RTprop/chunk/depth. Start optimized instead of re-probing. 1-hour TTL. Survives app restarts

### TD-005: Cipher preference order (runtime, not compile-time)
**Why:** _rank_ciphers scores each cipher by security×performance considering hardware availability. On Apple Silicon with NEON: GCM > CTR. On software-only: CTR > GCM. Cipher failure API (brz_exclude_cipher) removes known-bad ciphers per-server. No static preference list that works for all cases

---

## Decisions NOT YET Made (open questions for future sessions)

- **chacha20-poly1305 send bug:** Root cause unknown despite exhaustive investigation. May need packet capture comparison with OpenSSH
- **One-handle architecture:** BTP should share SFTPDataSource's brz_t* handle instead of creating separate connections per bearer. Design not finalized
- **OpenSSH fallback for release:** Re-enable for beta testing? Or ship libbrz-only and fix failures as they come?
- **QUIC transport (J-59):** Parked. No IETF draft for SSH-over-QUIC. Research when one appears
- **Test count recovery:** Peak was 1530, current ~503 after transport refactoring. Decide: restore old tests that still apply, or rewrite for current architecture?

---

*Updated when decisions are made. Check here before relitigating settled questions.*
