# tpt-family-mesh: Project Todo

Source: spec.txt (v0.2). Owner: TPT Solutions. Solo developer, no dates.

## Decisions (from clarification)

- [x] Scope: whole platform (mesh, PWA hub, five micro-apps, tpt-cortex, release)
- [x] Browser transport: browser-native RTCPeerConnection via web-sys (PWA) and webrtc-rs (native peers, e.g. tpt-cortex)
- [x] Yjs: browser uses JS-side Yjs; native Rust peers use yrs. Same binary update format on the wire
- [x] Peer auth: add a Noise XX milestone (snow)
- [x] Crypto backend: no ring (its crates.io license is Apache-2.0 AND ISC). Use pure-Rust crypto. Verify with cargo metadata
- [x] Dual-licensed deps (MIT OR Apache-2.0): select the MIT clause. Apache-only deps are banned
- [x] Frames: max payload 60KB, keep u16 payload_length (fixes the 64KB vs u16 mismatch)
- [x] Signaling: Hub Mode (wss to tpt-cortex), P2P Magic Link, QR code, and LAN discovery
- [x] Hosting: local only. Magic Link points to the tpt-cortex LAN address, not hub.tpt
- [x] tpt-cortex: standalone service (not a Home Assistant add-on)
- [x] Vault: Argon2id with a random salt per vault, stored with the ciphertext (replaces the fixed salt)

## Spec issues to fix in spec.txt

- [ ] Channel count: spec says "10 logical channels" in one place and "16" in another. Use 16 (ids 0-15)
- [ ] Remove pako-sys (C zlib binding). Use pako (JS, MIT) in the PWA or flate2 in Rust (select MIT)
- [ ] Remove webrtc-rs from tpt-family-mesh-wasm. It is native-only (moves to tpt-cortex)
- [ ] Correct "MIT licensed" claims for argon2 and chacha20poly1305 (both are MIT OR Apache-2.0; select MIT)
- [ ] Assign a channel for Yjs state-vector exchange (the channel map does not include it)
- [ ] Fix the 64KB frame / u16 mismatch in the spec text (now 60KB)
- [ ] Replace the fixed Argon2 salt in section 5.1 with a per-vault random salt
- [ ] Correct the vault security claim ("mathematically unreadable"). It depends on passphrase strength

---

## Phase 0: Licensing and Governance

- [ ] Confirm copyright holder name for "TPT Solutions" (legal entity name, if any)
- [ ] Add LICENSE-MIT and LICENSE-APACHE to the repo root (outbound dual license, MIT OR Apache-2.0)
- [ ] Add license headers and an SPDX identifier convention (`MIT OR Apache-2.0`)
- [ ] Write a dependency license policy (`deny.toml` for cargo-deny):
  - [ ] Allowed: MIT, BSD-3-Clause, Zlib, ISC, Unlicense
  - [ ] Dual licenses: allowed only when the MIT clause is selected
  - [ ] Denied: Apache-2.0-only, GPL-3.0, AGPL-3.0, MPL-2.0
- [ ] Audit every planned crate against cargo metadata: snow, argon2, chacha20poly1305, serde, serde_json, postcard, wasm-bindgen, web-sys, yrs, webrtc-rs, flate2
- [ ] Confirm the crypto backend that snow uses, with no ring and no Apache-only code
- [ ] Decide the TURN relay implementation for tpt-cortex, with its license checked
- [ ] Decide the QR code library (JS or Rust), with its license checked
- [ ] Decide the mDNS approach for LAN discovery (tpt-cortex side), with its license checked

## Phase 1: Foundations and Tooling

- [ ] Create the repo structure:
  - [ ] `tpt-family-mesh-core` (pure Rust framing, mux, scheduler, crypto. Testable on native)
  - [ ] `tpt-family-mesh-wasm` (wasm-bindgen wrapper around the core)
  - [ ] `tpt-cortex` (standalone signaling and TURN service)
  - [ ] `tpt-family-hub` (Svelte/TS PWA shell)
  - [ ] `apps/` (tpt-chat, tpt-lists, tpt-calendar, tpt-album, tpt-vault)
- [ ] Cargo workspace setup, with shared dependency versions
- [ ] Set up wasm-pack / wasm-bindgen build and test harness
- [ ] Set up Svelte + TypeScript + Vite for the PWA
- [ ] Set up formatting and linting: rustfmt, clippy (deny warnings), eslint, prettier
- [ ] Set up tests: cargo test (native), wasm-bindgen-test (browser), Vitest (TS), Playwright (two-peer E2E)
- [ ] Set up CI: build, test, cargo-deny license check, wasm build, lint
- [ ] Write a README with build steps and the license summary
- [ ] Add a CHANGELOG.md

## Phase 2: Core Multiplexer and Framing (Spec Milestone 1)

- [ ] Implement `MeshFrame` encode/decode: `channel_id: u8`, `sequence: u32`, `payload_length: u16`, `payload`
- [ ] Enforce the 60KB max payload. Reject oversized and malformed frames
- [ ] Define the channel map (0-15) with priority tiers, and reject unknown channel ids
- [ ] Build the channel demuxer. Routing must not parse payloads
- [ ] Implement per-channel sequence numbers and ordering for reliable channels
- [ ] Implement unreliable and unordered delivery for channels 0 and 6-15
- [ ] Implement the weighted fair queuing scheduler:
  - [ ] Channels 0-2 get 60%
  - [ ] Channels 3-4 get 30%
  - [ ] Channels 5-15 get 10%, bursting to 100% when others are idle
- [ ] Implement heartbeats and presence on channel 0
- [ ] Verify: chat (channel 1) and lists (channel 3) flow at the same time without blocking (loopback, native)
- [ ] Verify: the same test passes in the browser (wasm)
- [ ] Unit tests for framing edge cases (zero-length, max-length, truncated, bad channel id)

## Phase 3: Transport and Signaling (Spec Section 7)

### Browser transport (tpt-family-mesh-wasm, web-sys)

- [ ] Wrap browser RTCPeerConnection via web-sys (no webrtc-rs in the wasm build)
- [ ] Open a single RTCDataChannel and multiplex all channels over it
- [ ] Handle ICE candidate gathering and connection state changes
- [ ] Handle data channel backpressure, so the scheduler is not overrun

### tpt-cortex signaling server (standalone)

- [ ] Set up the tpt-cortex service (Rust, standalone process)
- [ ] Serve `wss://` on port 9911 with a trusted certificate for LAN use
- [ ] Implement peer registration (peer_id) and the online-peer list
- [ ] Relay WebRTC ICE candidates and SDP as JSON messages
- [ ] Implement TURN relay for when direct P2P fails (license-checked implementation)
- [ ] Implement LAN discovery (mDNS or equivalent, per Phase 0 decision)
- [ ] Add authentication for peer registration (decide the scheme)
- [ ] Serve the PWA static files on the LAN (per the local-only hosting decision)
- [ ] Add a health endpoint and logging

### Hub Mode (WebSocket adapter)

- [ ] Implement the wss client in the PWA: send peer_id, receive the peer list
- [ ] Exchange ICE candidates over the WebSocket
- [ ] Reconnect with backoff on drop

### P2P Mode (Magic Link, spec 7.2)

- [ ] Generate an SDP offer in the browser
- [ ] Compress the SDP (pako or flate2, per Phase 0 decision) and encode as URL-safe base64
- [ ] Build the link: `https://<cortex-lan-address>/#join=<base64>`
- [ ] Decode the link on open, generate the SDP answer, and produce the reply link
- [ ] Parse and validate incoming SDP (size limits, malformed input)
- [ ] Verify two peers connect with only copy-paste of links

### QR and LAN join

- [ ] Show the Magic Link as a QR code
- [ ] Scan a QR code with the camera to join (permission handling)
- [ ] Test QR join between a phone and a laptop

### Mode switching

- [ ] Detect environment: is tpt-cortex reachable? Use Hub Mode if yes, P2P if no
- [ ] Allow manual mode override in settings
- [ ] Verify the PWA switches modes cleanly without losing peers

## Phase 4: Peer Authentication (Noise XX)

- [ ] Implement the Noise XX handshake with snow (pure-Rust crypto backend, per Phase 0)
- [ ] Generate and persist device identity keys (see the storage decision in Phase 7)
- [ ] Run the handshake over the data channel before any app traffic
- [ ] Pin peer static keys after first trust, and show a warning on key change
- [ ] Define the peer trust UI (approve a new peer, revoke a peer)
- [ ] Test: a MITM peer with a wrong key is rejected
- [ ] Test: rekey and reconnect behavior

## Phase 5: Yjs Transport and State Vector Sync (Spec Milestone 2, Section 6)

### Update batching

- [ ] Batch Yjs updates for tpt-lists: flush every 50ms
- [ ] Batch Yjs updates for tpt-calendar: flush every 500ms
- [ ] Send tpt-chat updates immediately (append-only)
- [ ] Verify batching timing with fake timers and in browser

### State vector sync

- [ ] Implement the state vector exchange on connect (channel per Phase 0 decision)
- [ ] Compute and send only the missing updates from each state vector
- [ ] Handle concurrent edits and merge correctly (CRDT tests)
- [ ] Handle a peer that is far behind (large update, multi-frame)

### Engine integration

- [ ] JS-side Yjs integration in the PWA (browser)
- [ ] yrs integration on the native side (tpt-cortex or native peers)
- [ ] Verify cross-engine sync: a JS Yjs peer and a yrs peer converge

### Verification

- [ ] Verify: two peers sync a large Yjs doc (e.g. 10k+ items) in under 5 seconds

## Phase 6: Blob Streaming for tpt-album (Spec Milestone 3, Section 4)

- [ ] Chunk blobs into 60KB pieces, each with a `blob_id` (UUID) and `chunk_index`
- [ ] Send chunks on channels 6-15 (unreliable, unordered)
- [ ] Receiver: track missing chunks with a bitmap
- [ ] Implement retransmit requests for missing chunks only (use channel 5 for control)
- [ ] Reassemble the blob and verify its hash before handing to tpt-album
- [ ] Handle abandoned or partial transfers (timeouts, cleanup)
- [ ] Resumable transfer: resume after reconnect using the bitmap
- [ ] Verify: a 5MB photo streams P2P without blocking a concurrent chat conversation

## Phase 7: Application-Layer E2EE for tpt-vault (Spec Milestone 4, Section 5)

- [ ] Implement Argon2id key derivation in Wasm (per-vault random salt, stored with ciphertext)
- [ ] Choose and document Argon2id parameters (memory, time, parallelism) for the browser
- [ ] Measure key derivation time on a low-end phone. Adjust parameters
- [ ] Keep the derived key only in Wasm memory. Never write it to disk
- [ ] Implement XChaCha20-Poly1305 encrypt/decrypt (chacha20poly1305 crate, MIT clause)
- [ ] Use unique nonces per message. Test nonce handling
- [ ] Store vault payloads as opaque ciphertext in the Yjs document
- [ ] Decide where device identity keys are stored (IndexedDB, encrypted or not)
- [ ] Verify: vault data in IndexedDB is ciphertext only
- [ ] Verify: data decrypts only with the correct passphrase, and fails cleanly with a wrong one
- [ ] Add a passphrase strength check and warning in the UI
- [ ] Add a passphrase change flow (re-encrypt the vault)

## Phase 8: Micro-Apps

### tpt-chat (append-only log, channel 1)

- [ ] Message model and append-only log
- [ ] Immediate send, delivery and read state
- [ ] History sync on connect (uses Phase 5 state vectors)

### tpt-lists (live CRDT with presence, channel 3)

- [ ] List and item model on Yjs
- [ ] Live presence (who is editing)
- [ ] 50ms batching verified end-to-end

### tpt-calendar (event state, channel 4)

- [ ] Event model on Yjs
- [ ] Event create, edit, delete with conflict-free merge
- [ ] 500ms batching verified end-to-end

### tpt-album (metadata on channel 5, blobs on 6-15)

- [ ] Album and photo metadata model
- [ ] Thumbnail generation and sync via channel 5
- [ ] Full-size photo transfer via Phase 6 blob streaming
- [ ] Photo import from device, with EXIF handling (decide whether to strip location data)

### tpt-vault (E2EE, channel 2)

- [ ] Vault entry model (encrypted payloads)
- [ ] Encrypt on write, decrypt on read via Phase 7
- [ ] Lock and unlock flow (passphrase prompt, auto-lock timeout)

## Phase 9: PWA Hub Shell (tpt-family-hub)

- [ ] App shell, navigation, and per-app routing
- [ ] Service worker for offline shell and install
- [ ] Web app manifest (name, icons, theme)
- [ ] Mode indicator (Hub / P2P) and connection status
- [ ] Peer list and pairing UI (Hub and P2P flows)
- [ ] IndexedDB storage layer for app data (ciphertext for vault)
- [ ] Settings page (mode, peers, trust, passphrase)
- [ ] Responsive layout for phone and desktop
- [ ] Accessibility pass (keyboard, screen reader labels, contrast)
- [ ] Verify install on Android Chrome and iOS Safari (WebRTC support on iOS)

## Phase 10: Integration and End-to-End Testing

- [ ] Three-peer scenario: chat, lists, calendar, album, vault all running at once
- [ ] Offline and reconnect scenarios (peer drops and rejoins)
- [ ] Hub Mode E2E test (Playwright, two browsers, tpt-cortex running)
- [ ] P2P Mode E2E test (copy-paste links, no tpt-cortex)
- [ ] Mixed mode test (one peer Hub, one P2P)
- [ ] Load test: many channels saturated, check chat latency stays low
- [ ] Fuzz the frame parser and SDP/Magic Link decoder
- [ ] Performance budgets: first sync, frame latency, memory on a phone

## Phase 11: Security Review and Hardening

- [ ] Threat model document (peer, network, device seizure, malicious peer)
- [ ] Review Noise XX usage and key lifecycle
- [ ] Review vault crypto: nonces, salt, parameters, passphrase handling
- [ ] Check for secrets in logs, URLs, and storage (Magic Link fragments contain SDP)
- [ ] Review the tpt-cortex service: auth, rate limits, input limits, TLS config
- [ ] Dependency audit: cargo-deny, cargo audit, npm audit
- [ ] Content Security Policy for the PWA
- [ ] External review of the crypto code (recommended before relying on vault data)

## Phase 12: Release

- [ ] Final license check: cargo-deny passes, no Apache-only deps, dual deps resolved to MIT
- [ ] Publish tpt-family-mesh-core and tpt-family-mesh-wasm (crates.io or npm, decide which)
- [ ] Tag version 0.2.0 and write release notes
- [ ] Publish the PWA build and tpt-cortex release artifacts
- [ ] Install and setup guide for family members (Hub Mode and P2P Mode)
- [ ] Architecture and protocol docs (from spec.txt, updated)
- [ ] Post-release checklist: backup and restore of tpt-cortex, key recovery plan
