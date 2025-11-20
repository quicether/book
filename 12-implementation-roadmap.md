# Chapter 12: Implementation Roadmap

## Introduction

This chapter turns the architecture into a **concrete implementation plan**.

We define:
- Phased milestones (v0.1, v0.2, v0.3, v1.0)
- Scope of each phase
- Dependencies between components
- Non‑goals for early versions

The goal is to **ship something useful early**, then iterate without breaking the core principles.

---

## 12.1 Guiding Principles for Delivery

- **Vertical slices over horizontal layers:**
  - Prefer delivering end‑to‑end functionality (even if minimal) rather than building complete subsystems in isolation.
- **MVP, then iterate:**
  - v0.1 should already solve at least one real persona’s use case (Sarah/Alex).
- **Compatibility discipline:**
  - Avoid breaking on‑disk formats and network protocols without versioning.
- **Testability:**
  - Each milestone must be testable in isolation (unit + integration).

---

## 12.2 Milestone Overview

### v0.1: "Developer Mesh" (Local + Simple Remote)

**Goal:** Usable for homelab / developer scenarios on Linux.

Core capabilities:
- Single binary `quicether`
- Linux only (x86_64)
- TUN interface + basic routing
- Embedded Kademlia DHT (simplified)
- QUIC transport (single path, no multipath yet)
- TLS 1.3 mutual auth with self‑signed certs
- Basic CLI (`start`, `status`, `peers`, `routes`)

Primary personas:
- Sarah (remote developer)
- Alex (homelab enthusiast) — without multipath initially

### v0.2: "Multipath Alpha"

**Goal:** Introduce working multipath for advanced users.

Additions:
- Path discovery (multiple interfaces)
- Basic multipath scheduler (round‑robin or simple weighted)
- Path monitoring (RTT, loss)
- Metrics endpoint (Prometheus)

Primary personas:
- Alex (homelab) with colo gateway
- Maria (nomad) at an experimental level

### v0.3: "Gateway & Small Business"

**Goal:** Make it viable for small office/site‑to‑site setups.

Additions:
- Gateway configuration (`--role gateway`, `--forward-subnet`)
- Subnet advertisement via DHT
- Overlay routing table and subnet ownership
- Basic policy engine (allow/deny by NodeId and subnet)
- Audit logging to file

Primary personas:
- James (small business owner)

### v1.0: "Production‑Ready Core"

**Goal:** Stable, documented, with clear upgrade path.

Additions:
- Cross‑platform support (Linux, macOS; Windows if feasible)
- Robust CLI & IPC protocol (versioned)
- Mature multipath schedulers (weighted, latency‑aware)
- Hardened security (key management patterns, revocation hooks)
- Systemd/launchd/Windows service integration docs

Primary personas:
- Sarah, Alex, James, Maria in supported configurations

---

## 12.3 Detailed Phase Breakdown

### 12.3.1 Phase 1: Core Daemon Skeleton (v0.1‑A)

Deliverable: A process that starts, creates TUN, and passes packets to itself (loopback overlay).

Tasks:
1. Project scaffolding:
   - Rust workspace, crates for `daemon`, `cli`, `config`, `dht`, `transport`.
2. Config loading:
   - Parse minimal TOML config file.
   - Merge with env + flags.
3. TUN integration (Linux):
   - Create `quicether0`.
   - Read/write raw IPv4 packets.
4. Local echo path:
   - Embed minimal router that sends packets back to same node.

Testing:
- `ping 100.64.0.1` into TUN and see responses.

### 12.3.2 Phase 2: QUIC Transport (v0.1‑B)

Deliverable: Two nodes can establish secure QUIC connection and tunnel packets.

Tasks:
1. Integrate QUIC library (`quinn`/`s2n-quic`).
2. Implement TLS 1.3 with Ed25519 certificates.
3. Implement connection manager:
   - Map NodeId ↔ QUIC connection.
4. Encapsulate IP packets over QUIC streams.

Testing:
- Two nodes on LAN, static config (no DHT):
  - Node A: `quicether start --peer 192.168.1.20:9000`
  - Node B: `quicether start --listen 0.0.0.0:9000`
  - Ping across overlay addresses.

### 12.3.3 Phase 3: Basic DHT (v0.1‑C)

Deliverable: Nodes auto‑discover each other without static peer lists.

Tasks:
1. Implement minimal Kademlia:
   - NodeId generation
   - Buckets
   - `PING`, `FIND_NODE`, `STORE`, `FIND_VALUE` (subset)
2. Node metadata records only (`NODE:<NodeId>`).
3. Bootstrap logic using a static list of bootstrap nodes.
4. Integrate with connection manager:
   - Given NodeId → DHT lookup → addresses → connect.

Testing:
- 3+ nodes in network with only bootstrap IPs configured.
- All can discover and connect to each other without manual IP config.

### 12.3.4 Phase 4: Usable v0.1 Release

Bundle Phases 1–3:
- Package as v0.1
- Add CLI commands: `status`, `peers`, `routes` (read‑only at first).
- Documentation:
  - Install on Linux
  - Basic personal VPN setup

---

### 12.3.5 Phase 5: Multipath Foundations (v0.2‑A)

Deliverable: Multiple interfaces recognized and used, but may default to a simple strategy.

Tasks:
1. Interface discovery (enumerate `eth*`, `wlan*`, etc.).
2. Represent network paths and metrics structures.
3. Simple scheduler (round‑robin).
4. Path probing and basic failure detection.

Testing:
- Node with eth0 + wlan0.
- Start large file transfer, unplug one interface, see continuity.

### 12.3.6 Phase 6: Multipath Tuning (v0.2‑B)

Deliverable: More efficient multipath with basic configuration.

Tasks:
1. Weighted scheduler based on bandwidth estimations.
2. Basic latency‑aware adjustments.
3. Metrics export for per‑path RTT and throughput.

Testing:
- Synthetic lab with controlled link speeds.
- Validate aggregate throughput approaches sum of links.

---

### 12.3.7 Phase 7: Gateway & Subnet Routing (v0.3‑A)

Deliverable: Site‑to‑site and gateway forwarding.

Tasks:
1. Subnet advertisement:
   - `SUBNET:<CIDR>` records in DHT.
   - Local installation of `Local` routes.
2. Overlay routing table implementation (longest‑prefix match).
3. Gateway records:
   - `GW_SUBNET:<CIDR>` and node‑specific if needed.
4. Gateway forwarder logic:
   - Receive → policy check → forward into local subnet or onward.

Testing:
- Two sites with different 10.x subnets.
- Verify site‑to‑site via overlay and fallback via gateway when direct fails.

### 12.3.8 Phase 8: Policy & Audit (v0.3‑B)

Deliverable: Zero‑trust basics + logging.

Tasks:
1. Policy engine (allow/deny by NodeId + subnet).
2. Minimal policy file parser.
3. Enforcement hooks in router.
4. Structured audit logging to file.

Testing:
- Define policies that allow/deny subsets.
- Verify unauthorized traffic is dropped and logged.

---

### 12.3.9 Phase 9: Hardening for v1.0

Tasks:
1. Cross‑platform support:
   - macOS (TUN, launchd integration).
   - Optionally Windows (TUN driver, service).
2. IPC stabilization:
   - Versioned JSON‑RPC schema.
   - Backward‑compatible CLI.
3. Security documentation:
   - Key management guides.
   - Recommended policies and deployment patterns.
4. Performance optimizations:
   - Benchmark harness (iperf‑like).
   - Profiling and hot‑path optimizations.

Outcome:
- v1.0 release suitable for production in controlled environments.

---

## 12.4 Non‑Goals for Early Versions

To avoid scope creep, early versions will **not** include:

- Mobile apps (iOS/Android) → planned for post‑v1.0.
- Web UI dashboard → can be added once IPC is stable.
- Blockchain/PoA integration → optional, later milestone.
- Kernel bypass (DPDK/XDP) → performance lab project after core stabilizes.
- L2 bridging / TAP mode → separate design after L3 is mature.

These are explicitly **later** features.

---

## 12.5 Mapping to Personas

- **After v0.1:**
  - Sarah can use it for simple remote dev setups (Linux only).
  - Alex can build a small homelab mesh on Linux.

- **After v0.2:**
  - Alex can experiment with multipath.
  - Maria can start using WiFi + LTE bonding (early adopter).

- **After v0.3:**
  - James can deploy small site‑to‑site networks with gateways and basic policy.

- **After v1.0:**
  - All core personas can use QuicEther in some supported form, with docs and repeatable patterns.

---

## 12.6 Developer Workflow & Tooling

Recommended practices for the implementation phase:

- Use **Rust nightly only if necessary**; otherwise prefer stable.
- Enforce `cargo fmt` and `cargo clippy` in CI.
- Add integration tests under `tests/` that spin up multiple nodes.
- Use a simple **docker‑compose** file for multi‑node local testing.
- Maintain example configs in `examples/` matching the book’s scenarios.

---

## Summary

This roadmap turns the architecture into a staged delivery plan:

- v0.1: Core TUN + QUIC + DHT, Linux‑only, developer‑friendly mesh.
- v0.2: Multipath foundations and tuning.
- v0.3: Gateways, subnet routing, zero‑trust basics, suitable for small businesses.
- v1.0: Cross‑platform, hardened, production‑ready core.

Each phase delivers tangible value to at least one persona while building toward the full vision.

**Next Chapter:** We will outline **testing and validation strategies** for QuicEther across unit, integration, performance, and chaos testing.

---

**Chapter Navigation:**
- [← Previous: Chapter 11 - Daemon & CLI Architecture](./11-daemon-and-cli.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 13 - Testing & Validation](./13-testing-and-validation.md)
