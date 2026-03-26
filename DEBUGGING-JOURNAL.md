# Breeze — Debugging Journal
**Purpose:** One-line root-cause findings from every bug diagnosed across all sessions. When revisiting an area, check here first — saves hours of re-diagnosis.

---

## libbrz — SSH/Transport Layer

### KEX (Key Exchange)
- **S33 — KEX rejection by server:** Offered sntrup761x25519 without implementing it. Server picked it, got 32-byte x25519 key instead of 1155-byte hybrid. Fix: remove unimplemented algorithms from offer string, only offer curve25519-sha256
- **S33 — KEX_ECDH_INIT packet rejected:** Server closed connection after receiving our packet. Root cause was strict KEX + padding bug combined
- **S34 — Strict KEX seq reset timing:** Sequence numbers were reset after KEXINIT instead of after NEWKEYS. Every encrypted packet had wrong nonce/HMAC. Fix: move reset to newkeys.c
- **S34 — Strict KEX one-sided detection:** Enabled strict mode when only server offered it, without checking if we also offered it. Fix: require BOTH sides to signal support
- **S34 — EXT_INFO not handled:** Server sends SSH_MSG_EXT_INFO (type 7) after NEWKEYS. libbrz expected SERVICE_ACCEPT as first encrypted packet and choked. Fix: handle/skip type 7
- **S34 — compute_padding forgot packet_length:** Padding calculation didn't include 4-byte packet_length field in alignment. Fix: unpadded = 4 + 1 + payload_len per RFC 4253 §6
- **S34 — Global request not handled:** Server sends SSH_MSG_GLOBAL_REQUEST (hostkeys-00@openssh.com), SSH_MSG_DEBUG, SSH_MSG_CHANNEL_WINDOW_ADJUST between expected messages. Fix: skip loop for unexpected message types

### Ciphers
- **S34 — chacha20-poly1305 send path:** Recv works perfectly, send fails. Exhaustive investigation (S34+) confirmed send path matches IETF draft spec exactly. PARKED — subtle bug remains, needs deeper investigation another day. Key assignment K_1=key[0..31] (main), K_2=key[32..63] (header) confirmed correct
- **S42 — AES-256-GCM padding:** compute_padding_aead was incorrect for AEAD ciphers (GCM has no separate MAC). Fix applied, GCM encrypts/decrypts correctly end-to-end
- **S43 — NEON AES disabled:** ARM AESE instruction fuses AddRoundKey+SubBytes (different from x86 AES-NI). Session 43 implementation had round key mismatch — likely applied AESMC on last round, or forgot final rk[14] XOR, or wrong round count (AES-256 = 14 rounds = 15 round keys)
- **S44 — NEON AES re-enabled and PASSING:** Round keys ARE identical to standard AES — no modification needed. The session 43 bug was confirmed as one of the suspected issues (a/b/c/d/e from diagnosis). 9 gatekeeper tests confirm NEON output matches software byte-for-byte. AES-256-CTR now ~600 MB/s on Apple Silicon
- **S44 — GCM bottleneck is GHASH, not AES:** With NEON AES, CTR hits 600 MB/s but GCM only 17 MB/s. The bottleneck is software GHASH (polynomial multiply in GF(2^128)). Fix: NEON PMULL carry-less multiply — deferred to session 45
- **S43 — Error code collision:** BRZ_ESFTP and BRZ_ESFTP_INIT both defined as 30. SSH_FX_FAILURE responses produced misleading "sftp subsystem initialization failed" message. Fix: new codes BRZ_ESFTP_FAILURE=45, BRZ_ESFTP_UNSUPPORTED=46

### Auth
- **S34 — Agent auth hardcoded key type:** brz_ssh_agent_auth hardcoded "ssh-ed25519" for ALL key types. RSA keys need "rsa-sha2-256" algorithm name and SSH_AGENT_RSA_SHA2_256 flag (0x02). Fix: extract key type from agent key blob
- **S34 — Ed25519 pubkey auth from file:** keyparse returned error 73. Separate issue from agent auth. Agent auth path works and is primary

### SFTP Operations
- **S34 — brz_ls returns BRZ_ENET_CLOSED:** sftp_send_packet checks `rc != 4` (expects bytes written), but _brz_ssh_channel_write returns 0 (BRZ_OK). 0 != 4 → false "connection closed" error. SFTP init worked because ssh.c sends FXP_INIT directly via channel_write bypassing sftp_send_packet. Fix: align return value convention
- **S35 — Folder upload fails:** BrzTransport.mkdir() always called brz_mkdir even when recursive=true. brz_mkdir fails on "directory exists." Fix: call brz_mkdir_p when recursive=true (ignores EEXIST, creates parents)
- **S35 — handleRemoteMkdir wrong handle:** Was using BTP signaling handle instead of plugin session handle for mkdir operations. Fix: use plugin session (handle #1) first, BTP signaling as fallback
- **S43e — Packet corruption on concurrent send:** sftp_send_packet built packet in buffer then did multiple io.write calls. Concurrent bearers could interleave. Fix: memmove + single io.write (atomic send)

### Adaptive Engine
- **S41 — BBR STARTUP window exhaustion:** brz_set_autotune was a stub `(void)b; (void)enabled;`. BBR STARTUP doubled chunk from 32→64→128→256KB uncapped. At 256KB: server window (~262KB) exhausted. Fix: real implementation with tune.enabled flag, checked in tune_sample/tune_adapt
- **S41 — server_window never propagated to tune:** sftp_session_new() initialized tune.server_window=0. channel.c didn't propagate ssh->channel.remote_window after SFTP init. Ternary `t->server_window ? clamp(...) : BRZ_TUNE_MAX_CHUNK` returned 256KB (uncapped). Fix: propagate remote_window after sftp_session_init succeeds
- **S41 — OpenSSH initial_window=0:** OpenSSH 9.6 sends initial_window=0 in CHANNEL_OPEN_CONFIRMATION. Real window arrives via WINDOW_ADJUST during subsystem handshake. So server_window is correctly non-zero AFTER sftp_session_init, not before
- **S41 — 32KB chunk success:** With 32KB chunks, pipeline 8 writes before window exhaustion (~262KB window). With 256KB chunks, one SFTP packet barely fits. 32KB was discovered empirically as the sweet spot
- **S41 — brz_probe RTT duplication:** bandwidth heuristic and network_class assignment both had hardcoded RTT threshold tables. Fix: single lookup table (rtt_tbl[]) shared by both

### Connection Pool
- **S35 — Maintenance thread dead code:** maintenance_thread_fn in channel.c was defined but never called — pthread_create was deferred with "unused function" compiler warning. Fix: wire pthread_create in brz_pool_connect when validation_interval_sec > 0. Thread validates idle channels, replaces dead ones, detects abandoned. Shutdown joins thread before freeing
- **S41 — Per-file SSH overhead:** Each TransferQueue task acquires BTP bearer = new brz_open_ex call (~500ms SSH handshake). 283 files = 283 handshakes. Root cause of slow large folder uploads. Architectural fix needed: BTP should share SFTPDataSource's brz_t* handle (one-handle architecture)

### Performance
- **S43 — Software GCM vs CTR:** GCM=41.8s, CTR=26s for dmp/ (283 files, 1.05GB). GCM overhead is GHASH polynomial multiply in software. NEON PMULL (carry-less multiply) expected to give 25-50x GHASH speedup
- **S44 — NEON AES-CTR benchmark:** 600 MB/s on Apple Silicon with hardware NEON. Software was ~40 MB/s. ~15x speedup on raw AES-CTR. GCM still 17 MB/s due to software GHASH bottleneck
- **S43 — Dynamic cipher scoring:** _rank_ciphers shared helper ranks ciphers by security×performance. AES-256-GCM ranked above CTR when hw acceleration available. Software fallback prefers CTR
- **S43 — Cipher failure API:** brz_cipher_failure_t records which cipher failed on which server. brz_exclude_cipher prevents re-attempting known-bad ciphers. Persisted in ~/.brz/profiles

---

## Breeze.app — SwiftUI / AppKit Layer

### Critical Architecture
- **S29 — Drag-drop stale closures (THE BREAKTHROUGH):** SwiftUI structs are recreated constantly. Closures captured on structs become stale. Drag-drop events fire from AppKit AFTER SwiftUI recreates the struct → closure points to dead copy. Fix: TransferHandler as @Observable @MainActor final class (reference type). AppKitDropTarget uses Coordinator pattern (Apple's NSViewRepresentable approach). Case study: Docs/Case-Study-AppKit-SwiftUI-Drag-Drop.md
- **S35 — Beachball on folder upload:** uploadDirectory() blocked MainActor with sequential SFTP mkdir round-trips. 92 directories × round-trip = ~26s frozen UI with no feedback. Fix: Task.detached — walk local tree first (pure filesystem, instant), create remote dirs off MainActor, enqueue all files in batch

### Sidebar
- **S13+ — Sidebar text contrast:** `.listStyle(.sidebar)` on macOS applies system vibrancy styling that overrides `foregroundStyle()` with NSColor.labelColor through vibrancy. VisualEffectView compounds it. Partially fixed — readable when selected (black bg, white text), too light when unselected. Needs explicit color override that beats vibrancy
- **S44 — Sidebar click unresponsive FIXED:** Root cause: `.onTapGesture(count: 2)` on connection rows was intercepting single clicks — macOS SwiftUI waits to disambiguate single vs double tap, causing perceived unresponsiveness. Fix: replaced with `.simultaneousGesture` so single-click selection works immediately alongside double-click actions

### File Browser
- **S13 — SFTP navigation crash:** "Lost the breeze — File not found: Cannot open directory." Root cause: libssh2 SFTP session handle lost after navigating. The sftpSession handle must persist across multiple list() calls — libssh2_sftp_close_handle should close DIR handle, NOT SFTP session. Fix: NSLock on LibSSH2Session for thread safety
- **S35 — FTP intercepting SFTP:** FTPTransport was picking up SFTP connections because protocol filter wasn't applied. Fix: explicitly filter FTP out for SFTP connection configs
- **S44 — File list auto-scroll FIXED:** File list retained scroll position from previous directory when navigating. Fix: `.id(currentPath)` on the List — SwiftUI recreates the view when ID changes, resetting scroll to top
- **Loading UX:** "Drifting to /path..." with FlowingDotsText takes entire screen, path bounces as length changes, dots animation adds visual noise. Transmit feels faster because it shows tiny spinner in breadcrumb and keeps PREVIOUS listing visible. Fix needed: keep old listing, overlay subtle spinner

### Transfers
- **S44 — Progress "Zero KB/s" FIXED:** Progress callback from libbrz reported speed=0. Fix: fallback calculation in Breeze — `bytesTransferred / elapsed` when callback reports 0. Now shows real speed
- **S35 — Transfer duration misleading:** Shows wall-clock time from enqueue to completion, not actual transfer time. With 283 files queued serially, each file's "duration" includes wait time for all previous files
- **S35 — Shimmer log spam:** ~200+ shimmer activations in 3 seconds during welcome screen transition. Every file row triggers shimmer on appear. Not a crash risk but wastes CPU cycles

### Themes
- **S13 — Default theme wrong:** Launches Midnight not Mono. hasSetTheme guard exists but may not fire correctly on fresh install
- **S21 — Sidebar stays dark on light themes:** Fixed with `.preferredColorScheme(BreezeColors.theme.isDark ? .dark : .light)` on sidebar so .ultraThinMaterial renders correctly
- **S21 — Theme card selection unclear:** Added 2.5px accent border, checkmark badge (18px circle), unselected at 75% opacity, selected name in accent color

### Status Bar
- **S35 — Status bar showed "via OpenSSH":** String was hardcoded or read from wrong source. Actual data was flowing through libbrz (confirmed by trace). Fix: read transport name from BTP link manager state

---

## Build / Tooling

- **S3 — Test crash on first run:** SwiftData models needed ModelContainer in test setup. Test target was missing dependency
- **S11 — Claude Code permission prompts:** Resolved by using Bash(*) in settings.local.json rather than specific command lists
- **S17 — Git identity mismatch:** Work Mac commits as `jagjit.singh@singhdigital.ca`, personal Mac as `chamkila@MacBook-Pro-5.local`. Fix: `git config user.name/email` per-repo
- **S35 — libbrz compiler warning:** maintenance_thread_fn defined but never called. Warning persisted because pthread_create was deferred. Fix: wire it up properly

---

*Updated after each session. One line per bug. Check here before re-diagnosing.*
