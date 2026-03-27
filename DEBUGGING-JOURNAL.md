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
- **S44 — NEON AES re-enabled and PASSING:** Round keys ARE identical to standard AES — no modification needed. 9 gatekeeper tests confirm NEON matches software byte-for-byte. AES-256-CTR now ~600 MB/s on Apple Silicon
- **S44 — GCM bottleneck is GHASH, not AES:** With NEON AES, CTR hits 600 MB/s but GCM only 17 MB/s. Bottleneck is software GHASH (polynomial multiply in GF(2^128)). Fix: NEON PMULL — deferred
- **S43 — Error code collision:** BRZ_ESFTP and BRZ_ESFTP_INIT both defined as 30. Fix: new codes BRZ_ESFTP_FAILURE=45, BRZ_ESFTP_UNSUPPORTED=46

### Auth
- **S34 — Agent auth hardcoded key type:** brz_ssh_agent_auth hardcoded "ssh-ed25519" for ALL key types. RSA keys need "rsa-sha2-256" algorithm name and SSH_AGENT_RSA_SHA2_256 flag (0x02). Fix: extract key type from agent key blob
- **S34 — Ed25519 pubkey auth from file:** keyparse returned error 73. Separate issue from agent auth. Agent auth path works and is primary

### SFTP Operations
- **S34 — brz_ls returns BRZ_ENET_CLOSED:** sftp_send_packet checks `rc != 4` (expects bytes written), but _brz_ssh_channel_write returns 0 (BRZ_OK). 0 != 4 → false "connection closed" error. Fix: align return value convention
- **S35 — Folder upload fails:** BrzTransport.mkdir() always called brz_mkdir even when recursive=true. Fix: call brz_mkdir_p when recursive=true
- **S35 — handleRemoteMkdir wrong handle:** Was using BTP signaling handle instead of plugin session handle. Fix: use plugin session first, BTP signaling as fallback
- **S43e — Packet corruption on concurrent send:** sftp_send_packet built packet in buffer then did multiple io.write calls. Fix: memmove + single io.write (atomic send)

### Adaptive Engine
- **S41 — BBR STARTUP window exhaustion:** brz_set_autotune was a stub. BBR doubled chunk to 256KB uncapped. Fix: real implementation with tune.enabled flag
- **S41 — server_window never propagated to tune:** tune.server_window=0 at init, never set from ssh->channel.remote_window. Fix: propagate after sftp_session_init
- **S41 — OpenSSH initial_window=0:** Real window arrives via WINDOW_ADJUST during subsystem handshake. server_window is correct AFTER sftp_session_init
- **S41 — 32KB chunk success:** Pipeline 8 writes before ~262KB window exhaustion. 256KB barely fits. 32KB discovered empirically
- **S41 — brz_probe RTT duplication:** Fix: single lookup table (rtt_tbl[]) shared by both

### Connection Pool / Transfer Architecture
- **S35 — Maintenance thread dead code:** pthread_create was deferred. Fix: wire it up in brz_pool_connect
- **S41 — Per-file SSH overhead:** Each TransferQueue task = new brz_open_ex (~500ms). 283 files = 283 handshakes = 12+ seconds wasted
- **S45 — 24 SSH connections for 283 files FIXED:** Root cause: Breeze called brz_put 283 times individually through TransferQueue, each dispatched separately. brz_mput EXISTED in libbrz but was never called. Fix: BrzTransport.uploadBatch() calls brz_mput ONCE for entire folder. FileBrowserView.uploadDirectory() replaced 283 enqueue() calls with 1 brz_mput call. Per-file mkdir phase eliminated — brz_mput creates dirs on-the-fly with 64-entry directory cache. Before: 24 SSH connections, ~200s. After: 1 SFTP session, target <170s

### Performance
- **S43 — Software GCM vs CTR:** GCM=41.8s, CTR=26s for dmp/. GHASH polynomial multiply in software is the bottleneck
- **S44 — NEON AES-CTR benchmark:** 600 MB/s on Apple Silicon. ~15x speedup over software
- **S43 — Dynamic cipher scoring:** _rank_ciphers shared helper. GCM > CTR when hw available
- **S43 — Cipher failure API:** brz_cipher_failure_t per-server. Persisted in ~/.brz/profiles

---

## Breeze.app — SwiftUI / AppKit Layer

### Critical Architecture
- **S29 — Drag-drop stale closures (THE BREAKTHROUGH):** SwiftUI struct recreation makes closures stale. Fix: TransferHandler as @Observable @MainActor final class
- **S35 — Beachball on folder upload:** uploadDirectory() blocked MainActor with sequential mkdir. Fix: Task.detached
- **S45 — Folder upload architecture FIXED:** Replaced 283 individual TransferQueue.enqueue() calls with 1 BrzTransport.uploadBatch() → 1 brz_mput. Eliminated per-file Swift→C boundary crossing, per-file DispatchQueue dispatch, per-file BTP routing decision. Single batch task in transfer panel instead of 283 individual rows

### Sidebar
- **S13+ — Sidebar text contrast:** .listStyle(.sidebar) vibrancy overrides foregroundStyle(). Partially fixed
- **S44 — Sidebar click unresponsive FIXED:** .onTapGesture(count:2) intercepted single clicks. Fix: .simultaneousGesture

### File Browser
- **S13 — SFTP navigation crash:** libssh2 session handle lost. Fix: NSLock on LibSSH2Session
- **S35 — FTP intercepting SFTP:** Fix: explicitly filter FTP out for SFTP configs
- **S44 — File list auto-scroll FIXED:** Fix: .id(currentPath)
- **Loading UX:** "Drifting to..." full-screen takeover. Fix needed: keep old listing, overlay spinner

### Transfers
- **S44 — Progress "Zero KB/s" FIXED:** Fix: fallback calc bytesTransferred/elapsed
- **S35 — Transfer duration misleading:** Wall-clock includes queue wait time
- **S35 — Shimmer log spam:** ~200+ activations in 3 seconds

### Themes
- **S13 — Default theme wrong:** Launches Midnight not Mono
- **S21 — Sidebar dark on light themes:** Fix: .preferredColorScheme
- **S21 — Theme card selection unclear:** Fix: accent border + checkmark badge

### Status Bar
- **S35 — "via OpenSSH" when using libbrz:** Fix: read from BTP link manager state

---

## Build / Tooling
- **S3 — Test crash on first run:** SwiftData needed ModelContainer in test setup
- **S11 — Claude Code permissions:** Fix: Bash(*) in settings.local.json
- **S17 — Git identity mismatch:** Fix: git config per-repo
- **S35 — libbrz compiler warning:** maintenance_thread_fn unused. Fix: wire pthread_create

---

*Updated after each session. One line per bug. Check here before re-diagnosing.*
