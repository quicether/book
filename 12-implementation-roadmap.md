# Chapter 12: Implementation Roadmap

## Introduction

This chapter turns the architecture into a **concrete implementation plan**, informed by the lessons learned building httpf.

Because httpf already validated the core architecture (server mode, hubs, sessions, three-tier auth, firewall, policy, mesh, cascade, bridge mode, audit logging, mobile FFI), QuicEther's roadmap is accelerated. We are not exploring unknown territory — we are rebuilding proven patterns on a superior transport (native QUIC).

We define:
- Phased milestones (v0.1, v0.2, v0.3, v1.0)
- Scope of each phase
- What httpf already proved vs. what's new for QuicEther
- Non-goals for early versions

---

## 12.1 httpf Validation Summary

Before planning, let's acknowledge what httpf already proved works:

| Feature | httpf Status | QuicEther Adaptation |
|---------|-------------|---------------------|
| Single binary server/client | ✅ Proven | Port to QUIC transport |
| Hub-based multi-tenancy | ✅ Proven | Direct port |
| Session management | ✅ Proven | Direct port |
| Three-tier auth (Ed25519/password/service token) | ✅ Proven | Direct port |
| Firewall engine (Proxmox-style) | ✅ Proven | Direct port |
| Policy engine (per-identity L2/L3/L4) | ✅ Proven | Extend to L2 MAC rules |
| Virtual Hub + anti-spoofing | ✅ Proven | Port as MAC-based switching |
| Server mesh | ✅ Proven | Port to QUIC mesh protocol |
| Cascade connections | ✅ Proven | Direct port |
| Bridge mode | ✅ Proven | Extend to Local Bridge (physical LAN) |
| Audit logging (JSONL + syslog) | ✅ Proven | Direct port |
| Mobile FFI (iOS/Android) | ✅ Proven | Direct port |
| FrameBatch format | ✅ Proven | Port as Ethernet frame batches |
| Multipath | ⚠️ Parallel TCP | Native QUIC multipath (new) |
| 0-RTT reconnection | ❌ Not possible over TCP | Native QUIC feature (new) |
| Connection migration | ❌ Not possible over TCP | Native QUIC feature (new) |
| TAP interface | ✅ L2 parsing proven | Port with full TAP mode |
| Virtual TAP (L2↔L3 translation) | ✅ ARP/MAC primitives proven | New: userspace L2↔L3 for mobile |

**Bottom line:** ~80% of QuicEther is a direct port from httpf. The new work is primarily the QUIC transport layer, native multipath, and connection migration.

---

## 12.2 Guiding Principles for Delivery

- **Port, don't reinvent:**
  - Start by porting httpf's proven code to QUIC. Don't redesign what already works.
- **QUIC-first features early:**
  - Prioritize the features that justify QuicEther's existence over httpf: native multipath, 0-RTT, connection migration.
- **Ship fast, iterate:**
  - httpf's architecture is validated. v0.1 can ship with full feature parity minus multipath.
- **Compatibility discipline:**
  - Avoid breaking on-disk formats and network protocols without versioning.
- **Testability:**
  - Port httpf's test infrastructure. Each milestone must be testable.

---

## 12.3 Milestone Overview

### v0.1: "QUIC Core" (Feature Parity with httpf)

**Goal:** Port httpf's complete feature set from HTTP/WebSocket to native QUIC.

Core capabilities (all proven in httpf, ported to QUIC):
- Single binary `quicether` (server/connect/bridge modes)
- Linux (x86_64) + macOS
- TAP interface + Virtual Hub (MAC-based switching)
- **Device Abstraction Layer: Native TAP + Virtual TAP** (L2↔L3 translation for platforms without TAP)
- QUIC transport with TLS 1.3 (using `quinn`)
- Hub-based multi-tenancy (Ethernet segments)
- Session management
- Three-tier authentication (Ed25519, password, service token)
- Firewall engine (Proxmox-style ACL)
- Policy engine (per-identity L2/L3/L4)
- FrameBatch encapsulation over QUIC streams
- Audit logging (JSONL + syslog)
- CLI: server, connect, bridge, identity, hub, session, password, audit
- Prometheus metrics endpoint
- 0-RTT reconnection (new — QUIC native)

Primary personas:
- Sarah (remote developer) — connect to office server
- Alex (homelab) — run personal VPN server
- James (small business) — server with hubs and policy

**Timeline advantage:** httpf code can be directly ported. Transport layer is the only new code.

### v0.2: "Multipath & Migration"

**Goal:** Deliver QuicEther's core differentiators over httpf.

Additions:
- Native QUIC multipath (multiple interfaces)
- Path discovery, monitoring, and scheduling
- Performance profiles (latency/balanced/throughput/max_performance)
- Connection migration (WiFi → LTE seamless handoff)
- Path failure detection and automatic recovery

Primary personas:
- Maria (digital nomad) — WiFi + LTE bonding
- Alex (homelab) — multiple ISP aggregation

### v0.3: "Server Mesh & Scale"

**Goal:** Multi-server deployments and enterprise features.

Additions:
- Server mesh over QUIC (porting httpf's mesh to native QUIC)
- Cascade connections across mesh peers
- Cross-hub frame forwarding via mesh
- Rate limiting (token bucket per identity)
- Advanced firewall (ICMP types, port ranges)
- Admin API (REST over local socket)

Primary personas:
- James (multi-location business)
- David (enterprise IT)

### v1.0: "Production-Ready"

**Goal:** Stable, documented, battle-tested.

Additions:
- Windows support (TAP driver + service integration)
- Mobile SDKs (iOS/Android via FFI — ported from httpf)
- Hardened security (key rotation, revocation hooks)
- Performance optimizations (profiling, hot-path tuning)
- Comprehensive documentation and deployment guides

Primary personas:
- All personas in supported configurations

---

## 12.4 Detailed Phase Breakdown

### 12.4.1 Phase 1: QUIC Transport Core (v0.1-A)

Deliverable: Two nodes can establish QUIC connection and tunnel Ethernet frames.

Tasks:
1. Port httpf's project structure to QUIC:
   - Replace `hyper`/`tokio-tungstenite` with `quinn`
   - Keep all other crates intact
2. QUIC listener and connector:
   - Server: `quicether server` listens on UDP port
   - Client: `quicether connect` connects to server
3. TLS 1.3 with Ed25519 certificates
4. FrameBatch over QUIC unidirectional streams
5. TAP interface (port from httpf with L2 mode)
6. Virtual TAP abstraction (TUN + L2↔L3 translation for mobile/restricted platforms)

Testing:
- Server + client on same LAN
- DHCP through tunnel from physical router
- Virtual TAP mode: verify Ethernet frame synthesis from IP packets
- `ping` across overlay
- `iperf3` throughput baseline

### 12.4.2 Phase 2: Hub & Session Management (v0.1-B)

Deliverable: Multi-tenant server with Virtual Hubs, auth, and sessions.

Tasks (all direct ports from httpf):
1. Hub manager with Virtual Hub (Ethernet segments)
2. Session manager (create, track, destroy)
3. Three-tier auth (Ed25519, password/Argon2id, service token)
4. MAC learning table with anti-spoofing
5. CLI: `hub`, `session`, `identity`, `password`

Testing:
- Multiple clients connecting to one server
- Each joins hub, gets DHCP from physical router
- Clients can communicate via Virtual Hub switching

### 12.4.3 Phase 3: Security & Audit (v0.1-C)

Deliverable: Complete security stack.

Tasks (all direct ports from httpf):
1. Firewall engine (Proxmox-style ACL, first-match)
2. Policy engine (per-identity L2/L3/L4 rules)
3. Audit logger (JSONL file + optional syslog)
4. Prometheus metrics endpoint
5. 0-RTT reconnection support (QUIC session tickets)

Testing:
- Verify firewall blocks unauthorized traffic
- Verify audit logs capture events
- Verify 0-RTT reconnection works

### 12.4.4 Phase 4: Bridge Mode + v0.1 Release

Deliverable: Packaged v0.1 release with full httpf feature parity.

Tasks:
1. Local Bridge (client + physical LAN bridging to hub)
2. CLI: `bridge`, `audit`, admin commands
3. Docker image + systemd unit file
4. Documentation: install guide, server setup, client setup
5. CI/CD: automated builds and tests

---

### 12.4.5 Phase 5: Multipath (v0.2)

Deliverable: Working multipath over QUIC.

Tasks (this is genuinely new — httpf only had parallel TCP):
1. Interface discovery (enumerate local network interfaces)
2. Path representation and metrics (RTT, loss, bandwidth)
3. QUIC multipath extension integration
4. Scheduler implementations:
   - Round-robin
   - Weighted
   - Latency-aware
   - Redundant
5. Performance profiles (latency/balanced/throughput/max_performance)
6. Connection migration (IP/port change without session loss)
7. Path failure detection and automatic recovery

Testing:
- Dual-interface node (eth0 + wlan0)
- Bandwidth aggregation verification
- Interface failover test (unplug cable)
- Connection migration test (switch WiFi networks)

### 12.4.6 Phase 6: Server Mesh (v0.3)

Deliverable: Multi-server mesh with cross-hub routing.

Tasks (port httpf's mesh to QUIC):
1. Mesh peer connections via QUIC
2. Hub membership advertisement protocol
3. Cross-hub frame forwarding
4. Cascade connections (multi-hop)
5. Service token authentication for mesh peers
6. Rate limiting (token bucket)
7. Admin API (REST over local socket)

Testing:
- 3-node server mesh
- Client on server A reaches client on server B
- Verify cascade connections with 3-hop path

---

### 12.4.7 Phase 7: Production Hardening (v1.0)

Tasks:
1. Windows support (TAP driver, named pipe IPC, Windows service)
2. Mobile FFI (port httpf's iOS/Android bindings to QUIC, using Virtual TAP for L2↔L3 translation)
3. Performance profiling and optimization
4. Key rotation and revocation patterns
5. Comprehensive documentation
6. Security audit

---

## 12.5 Non-Goals for Early Versions

To avoid scope creep:

- **DHT / P2P discovery** → deferred to v2.0+ (server mesh is sufficient)
- **Kernel bypass (DPDK/XDP)** → performance lab project after core stabilizes
- **L3 routing / TUN mode** → optional future addition after L2 core is mature
- **Web UI dashboard** → can be added once admin API is stable
- **Post-quantum crypto activation** → ML-KEM-768 prepared but not enabled until ecosystem matures

---

## 12.6 Mapping to Personas

| Persona | After v0.1 | After v0.2 | After v0.3 | After v1.0 |
|---------|-----------|-----------|-----------|-----------|
| Sarah (developer) | ✅ Full use | ✅ + multipath | ✅ | ✅ |
| Alex (homelab) | ✅ Full use | ✅ + multi-ISP | ✅ | ✅ |
| James (small biz) | ✅ Full use | ✅ | ✅ + multi-site | ✅ |
| Maria (nomad) | ✅ Basic | ✅ Full use | ✅ | ✅ |
| Priya (rural) | ✅ Basic | ✅ Full use | ✅ | ✅ |
| David (enterprise) | ⚠️ Eval only | ⚠️ | ✅ Full use | ✅ Production |

---

## 12.7 Developer Workflow & Tooling

Practices carried forward from httpf:

- Use **Rust stable** (no nightly dependency)
- Enforce `cargo fmt` and `cargo clippy` in CI
- Integration tests that spin up server + multiple clients
- Docker Compose for multi-node testing (reuse httpf's CI setup)
- Benchmark harness using `criterion` (iperf3-equivalent)
- Example configs in `configs/` matching book scenarios
- Structured logging with `tracing` crate

---

## Summary

This roadmap leverages httpf's proven architecture to accelerate delivery:

- **v0.1:** Port httpf to QUIC — full feature parity with 0-RTT reconnection
- **v0.2:** Native QUIC multipath + connection migration (QuicEther's core differentiator)
- **v0.3:** Server mesh over QUIC + enterprise features
- **v1.0:** Cross-platform, mobile SDKs, production hardening

Because ~80% of the architecture is a direct port from httpf, QuicEther can reach feature parity faster than a greenfield project. The genuinely new work — native multipath and connection migration — is concentrated in v0.2.

**Next Chapter:** We will outline **testing and validation strategies** for QuicEther across unit, integration, performance, and chaos testing.

---

**Chapter Navigation:**
- [← Previous: Chapter 11 - Daemon & CLI Architecture](./11-daemon-and-cli.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 13 - Testing & Validation](./13-testing-and-validation.md)
