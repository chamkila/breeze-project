# Breeze — Session Log
**Purpose:** One-paragraph summary per session. Quick scan to find when something happened.

---

## Thread 1: "Building a next-generation SFTP client with Claude and Xcode"

### Session 1 (pre-2026-03-13)
Named the project "Breeze." Created 28KB BREEZE-SPEC.md, plugin architecture (BreezeDataSource protocol, TransferEngine, PluginManager), set up GitHub repo (chamkila/breeze), first commit 04fddc9. Discussed MDS/telecom heritage — jacket pattern, cwmon, zero-downtime ops. Generated SSH key for GitHub. Signed up for Claude Max 5x plan.

### Session 2 (~2026-03-13)
First Claude Code overnight build. Created Xcode project, SwiftUI shell, dual-pane layout. Extracted ContentView into SidebarView, WelcomeView, ConnectionFormView, FileBrowserView, SSHKeyViewer, SSHConfigParser. Created SessionPool actor. Unit tests for parsers, paths, transport manager. ~79 tests. First real refactor from monolith to FRAME pattern.

### Session 3 (~2026-03-13)
TransferQueue actor singleton (enqueue/pause/resume/cancel, 4 concurrent, exponential backoff retry). TransferPanel UI. FTP plugin (URLSession-based, priority 80). Auto-reconnect, connection timeout (10s), graceful quit warning. 114 tests, 23 minutes compute.

### Session 4 (~2026-03-14)
ChannelMultiplexer (priority-based pooling: control/interactive/data/bulk). NetworkProbe (enviper-style fingerprinting — LAN/fast/WAN/slow/satellite, auto-calculates BDP). BreezeMark benchmark (rating system, JSON export). Smart transfer engine with conflict resolution. Drag-drop upload. ~200 tests.

### Sessions 5-7 (~2026-03-14)
BandwidthThrottle, synchronized browsing, SSH key generation + remote install, QuickConnect bar, BreezeCLI foundation, connection profiles (.breezeprofile), remote file search (recursive glob), port forwarding (local/remote/SOCKS), resume transfers (byte-level with ResumeManager), queue editing (drag reorder, priority), external diff (FileMerge), mkdir -p UI, recent connections + import from FileZilla/Cyberduck, connection groups (hierarchical sidebar tree). ~427 tests.

### Sessions 8-9 (~2026-03-15)
BreezeVault encryption (Apple CryptoKit AES-256-GCM), protocol visualization, SSHFS placeholder, predictive prefetch (CoreML), ASCII celebration, AppleScript integration, archive VFS (browse .tar.gz/.zip as virtual dirs), smart queries (panelized results), directory comparison, embedded terminal placeholder, QuickLook spacebar fix, clickable breadcrumb, file inspector panel. 1052 tests.

### Session 10 (~2026-03-15)
Hold music placeholder, SFTP pipelining infrastructure, transfer scheduler. 1328 tests.

### Session 11 (~2026-03-15)
BTP Phase 1 wired (SS7 signaling/bearer multiplexing live in the app). Navigation history. Buffer tuning. Loading states. CLAUDE.md rewritten as comprehensive guide. 1430 tests.

### Session 12 (~2026-03-15)
Keyboard-driven operations (Cmd+arrows, Enter, Delete, Space). Connection profiles persistence. Progress indicators throughout. Error handling improvements. 1530 tests.

### Session 13+ (~2026-03-15)
GUI polish, bug bash. Theme system (5 themes: Breeze/Midnight/Dusk/Forest/Mono) with one-click switching via BreezeColors delegation. Welcome screen redesign (compact 650×450, three-stage connection animation). Design-Principles.md created and added to CLAUDE.md as mandatory reading. QA testing: sidebar contrast bug found (highest priority), settings button unwired, toolbar tooltips missing, "Reveal Folder" → "Reveal in Finder" needed, tag filter needs text input. SESSION-STATE.md committed for continuity. Critical SFTP navigation bug fixed (libssh2 thread safety → NSLock on LibSSH2Session).

---

## Thread 2: "Breeze SFTP single-pane redesign progress" (~2026-03-17)

### Session P-013
Planned bug bash + settings wiring task. Sidebar light theme fix (replace .ultraThinMaterial with explicit theme-aware backgrounds). Settings tabs for Connections and Transfers. DiffService actor using /usr/bin/opendiff (FileMerge). Terminal toolbar button via NSAppleScript. Tag filter text input. Cmd+K file operations for connected state. Termius Beta competitive analysis saved to Docs/Competitive-Termius-Beta.md — identified Termius's mandatory account creation and subscription model as Breeze differentiation opportunity.

---

## Thread 3: "Sending remaining mixed up images" (~2026-03-17 to 2026-03-19)

### BTP Deep Dive
Studied actual SS7 source code: OpenSS7 mtp3.cpp (state matrix, signal-driven I/O, message priority, link changeover) and Extended-jSS7 M3UA As.java (application server, Mtp3TransferPrimitive, traffic modes). Designed BTP from these patterns — 5 Swift files written directly to project: BTPServiceIndicator, BTPState, BTPLink, BTPLinkManager, BTPTransferRequest.

### libbrz Creation
BreezeSSH started as integrated SSH transport inside libbrz (src/ssh/ and src/crypto/). Crypto primitives vendored from public-domain sources (tinyssh, dropbear, openssh-portable) — concepts/algorithms only, zero code copied. BrzTransport created at priority 300, replacing libssh2 as primary. Shimmer text effects added (.softLight on light, .plusLighter on dark). FTP-intercepting-SFTP bug fixed. libbrz v0.3.0-ssh: 249 tests.

### Architecture Docs
Protocol-Layered-Architecture.md created (TCP/IP + SS7 + enviper layering). BreezeVault designed (Apple CryptoKit AES-256-GCM). IP policy codified: Apple frameworks as foundation, build on top, never copy, never depend on competitor library. Security rule: Keychain for ALL credentials. 128-bit CPU future discussion — conclusion: not the next wave, real future is post-quantum crypto, neuromorphic computing, photonic interconnects. Icon design exploration across all 5 themes.

---

## Thread 4: "Breeze GUI development approach" (~2026-03-21 to 2026-03-25)

### libbrz KEX Breakthrough (Sessions ~33-34)
KEX_ECDH_INIT rejection debugged: sntrup761 offered without implementation, strict KEX seq reset at wrong point, one-sided detection, EXT_INFO not handled, compute_padding missing 4-byte packet_length. Five bugs found and fixed. aes256-ctr encrypted channel working. Ed25519 pubkey auth implemented. Agent auth RSA key type extraction fixed.

### First SFTP Connection (Session ~34)
libbrz v1.0.0: curve25519-sha256 + aes256-ctr + RSA agent auth + SFTP v3. brz_ls failure diagnosed (channel_write return convention mismatch). Fixed. 607 tests. chacha20-poly1305 send investigated exhaustively, parked.

### Breeze Integration (Session ~35)
Status bar fixed to show `via libbrz`. Folder upload failure diagnosed (brz_mkdir vs brz_mkdir_p). Error code collision fixed. Beachball diagnosed (sequential mkdir blocking MainActor). Maintenance thread wired up. Connection pool verified.

### Performance Analysis (Session 35)
Large folder upload tested (dmp/, 283 files, 1.05 GB). 32KB chunk size discovered as sweet spot for server window. Per-file SSH overhead identified as primary bottleneck. One-handle architecture needed. Performance analysis doc created.

### Prompt Preparation
Post-quantum KEX prompt written (sntrup761, mlkem768). Performance prompt written (writev, kqueue/epoll, AES-NI/NEON, mmap, delta sync). QUIC research parked (J-59).

---

## Thread 5: "Breeze upload performance and testing tasks" (2026-03-26, current)

### Sessions 36-40
libbrz rebuilt and tested. Beachball fix verified. Folder upload verified (283/283 files, zero failures). 32KB→64KB chunk optimization. Transfer speed analysis (1.4 MB/s peak on 319MB file).

### Session 41 (BBR Adaptive Engine)
brz_tune_t implemented: 4-phase state machine (STARTUP/DRAIN/PROBE_BW/PROBE_RTT), deliveryRate measurement, windowed-max BtlBw, windowed-min RTprop, BDP-based chunk/depth. Server profile cache (~/.brz/profiles). brz_probe() API. 607 tests.

### Session 41b
Fixed three stubs: brz_set_autotune (real implementation with tune.enabled flag), brz_set_chunk_size (interacts with autotune — resets phase to PROBE_BW), brz_probe (deduplicated RTT table). 607 tests.

### Session 41c
Extracted tune_sample, tune_adapt, brz_clamp_u32, brz_align_down, brz_now_sec from sftp.c into tune.c/tune.h (testable). test_tune.c (31 tests), test_profile.c (8 tests). 646 tests.

### Sessions 42-43
Dynamic cipher scoring (_rank_ciphers shared helper). AES-256-GCM end-to-end working (compute_padding_aead fix). Cipher failure API (brz_cipher_failure_t, brz_exclude_cipher). NEON AES attempted — disabled due to AESE round key mismatch. Software GCM=41.8s, CTR=26s for dmp/.

### Session 43e
Atomic sftp_send_packet (memmove + single io.write). BreezeEngineOnly=true confirmed. 651 tests. Session 44 prompt prepared: NEON hardware AES (correct AESE round key schedule) + Breeze UI fixes (sidebar click, Zero KB/s, auto-scroll, file icons).

### Session 44 (2026-03-26)
NEON AES-256 hardware acceleration re-enabled and passing (9 gatekeeper tests confirm NEON output matches software byte-for-byte). AES-256-CTR now ~600 MB/s on Apple Silicon. AES-256-GCM bottlenecked at ~17 MB/s by software GHASH — NEON PMULL GHASH deferred to next session. Breeze rebuilt with NEON-accelerated libbrz.a (aese/aesmc confirmed in binary via nm). Breeze UI fixes: sidebar click restored (.simultaneousGesture replacing .onTapGesture(count:2)), transfer speed display (fallback calc when callback reports 0), file list auto-scroll (.id(currentPath)), file icons (30+ extensions mapped to semantic SF Symbols with subtle color coding). 660 tests. 15m 49s compute. libbrz: 3ab2573, 8020a2b, 0f52e4d, f15210d. Breeze: e160cc6, 9ad8126, 2ef8cff, ed38076, ecb351a.

### Session 45 (2026-03-26)
THE ARCHITECTURAL FIX. brz_mput enhanced with batch progress callback (brz_batch_progress_cb: file_index, filename, per-file and aggregate bytes/speed), result struct (brz_batch_result_t: succeeded/failed/skipped counts), streaming mkdir with directory cache (creates dirs on bearer SFTP session on-the-fly, 64-entry cache avoids duplicate calls), and error resilience (continues on per-file errors when result struct provided). Breeze wired: BrzTransport.uploadBatch() calls brz_mput in ONE DispatchQueue dispatch. FileBrowserView.uploadDirectory() replaced 283 individual enqueue() calls with 1 brz_mput call. Per-file mkdir phase eliminated — brz_mput handles it. Before: 283 × brz_put → 24 SSH connections → ~200s. After: 1 × brz_mput → 1 SFTP session → target <170s. 8m 4s compute. libbrz: e1a860b, ba6b6a3. Breeze: a7050c1, f66e3c3, 11d10d0/5ef7e25.

---

*Updated after each session. Add one paragraph, commit breeze-project.*
