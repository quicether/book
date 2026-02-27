# Chapter 8: QUIC Transport & Multipath

## Introduction

This chapter dives into how QuicEther uses QUIC as its transport layer and how we build **multipath** on top of it.

We cover:
- QUIC connection model (sessions, streams, migration)
- Packet format as seen by QuicEther
- Multipath design (paths, schedulers, monitoring)
- Interaction with the TAP interface and bridging
- Failure handling and performance considerations

---

## 8.1 QUIC Primer (In Our Context)

QUIC is a UDP-based, encrypted transport protocol standardized by the IETF (RFC 9000).

Key properties we rely on:
- **User-space implementation** → rapid iteration, advanced features
- **TLS 1.3 integrated** → no separate handshake protocol
- **Streams** → logical channels within a single connection
- **Connection migration** → survive IP/port changes
- **Multipath (draft)** → multiple network paths per connection

### 8.1.1 Connection Model

A QUIC connection in QuicEther:
- Identified by a **Connection ID** (CID) chosen by each side
- Carries multiple **unidirectional** and **bidirectional streams**

We conceptually structure streams as:

| Stream Type | Direction | Purpose |
|------------|-----------|---------|
| Control    | Bi-dir    | Authentication, keepalives, config negotiation, heartbeats |
| Data       | Uni-dir   | Encapsulated Ethernet frames (FrameBatch tunnel traffic) |
| Mesh       | Bi-dir    | Server mesh protocol (hub updates, MAC sync) |
| Metrics    | Uni-dir   | Optional stats, monitoring, audit events |

Implementation detail:
- Specific stream IDs and framing are an internal concern; architecture assumes logical separation.

### 8.1.2 Connection Migration

When a node’s IP/port changes (e.g., WiFi → LTE):
- QUIC uses connection IDs to route packets to the correct connection
- Endpoint validates new path (path validation)
- In-flight streams continue without teardown

QuicEther leverages this to support:
- Laptop roaming (Sarah, Maria)
- Mobile clients switching between networks without dropping sessions

---

## 8.2 QuicEther Packet Encapsulation

### 8.2.1 Logical View

From QuicEther’s perspective:

```text
[Ethernet frame from TAP]
          ↓ encapsulated
[QUIC STREAM frame payload]
          ↓ encrypted
[UDP datagrams]
          ↓
[IP over the Internet]
```

We do **not** define a new VPN protocol from scratch. Instead, we:
- Treat each Ethernet frame (or small batch) as the payload of a QUIC stream frame
- Optionally add a small header for metadata (e.g., QoS flags, flow ID)

### 8.2.2 FrameBatch Format (Proven in httpf)

Instead of sending one Ethernet frame per QUIC frame, QuicEther uses a **FrameBatch** format validated in httpf for amortizing per-frame overhead:

```text
+----------+----------------+--------+------------------+--------+------------------+-----+
| ver:u8   | num_frames:u16 | size:u16| Ethernet Frame 1 | size:u16| Ethernet Frame 2 | ... |
+----------+----------------+--------+------------------+--------+------------------+-----+
```

```rust
struct FrameBatch {
    version: u8,     // Protocol version (currently 1)
    num_frames: u16,
    frames: Vec<SizedFrame>,
}

struct SizedFrame {
    size: u16,       // Length of the Ethernet frame
    data: Vec<u8>,   // Raw Ethernet frame bytes
}
```

Design rationale:
- **1-byte version prefix** enables protocol evolution without breaking backward compatibility
- **Batching** reduces per-frame QUIC stream overhead
- **Simple `size:u16` prefix** per frame enables zero-copy parsing
- **LZ4 compression** applied to the entire batch payload before QUIC encryption
- httpf proved that batching 4-8 frames per QUIC stream write improves throughput by ~15-20%

---

## 8.3 Connection Lifecycle in QuicEther

### 8.3.1 Establishing a Connection

Scenario: Client A connects to Server S.

1. **Server Address:**
   - Client has server address from config (e.g., `server = "vpn.example.com:4433"`)
   - No DHT lookup needed — direct connection to known server

2. **QUIC Handshake:**
   - Client creates QUIC endpoint, connects to server address
   - TLS 1.3 with server certificate validation
   - 0-RTT supported for reconnections (session tickets)

3. **Authentication (Three-Tier):**
   - On control stream, client authenticates via one of:
     - **Ed25519 identity** — cryptographic identity verification
     - **Password** — Argon2id-hashed credential
     - **Service token** — pre-shared key for server mesh peers
   - Server validates and assigns client to a hub

4. **Hub Assignment & TAP Setup:**
   - Server assigns client a MAC address and adds to Virtual Hub
   - Sends hub configuration (MTU, firewall rules)
   - Data streams are opened for FrameBatch tunnel traffic

5. **Ready State:**
   - Client configures TAP interface
   - Bidirectional Ethernet tunnel is established

### 8.3.2 Maintaining a Connection

- Periodic keepalives (PING frames or custom control messages)
- Path probing (for multipath, see below)
- Idle timeout based on observed usage

### 8.3.3 Tearing Down

- Graceful:
  - QUIC CLOSE frame, flush pending frames
- Ungraceful:
  - Timeouts, error conditions; both sides clean up local state

---

## 8.4 Multipath Design

Multipath turns one logical QUIC connection into **multiple network paths**:
- Example: WiFi + LTE + Ethernet simultaneously

### 8.4.1 Terminology

- **Path:** A tuple of (local IP/port, remote IP/port, interface)
- **Active Path:** Currently used for sending data
- **Probing Path:** Being tested, not yet trusted
- **Failed Path:** Marked unusable due to repeated timeouts

```rust
struct NetworkPath {
    id: PathId,
    interface: String,         // e.g., "eth0"
    local_addr: SocketAddr,
    remote_addr: SocketAddr,
    state: PathState,
    metrics: PathMetrics,
}

enum PathState {
    Probing,
    Active,
    Failed,
}

struct PathMetrics {
    rtt: Duration,
    loss: f32,
    bandwidth_estimate: u64,   // bytes/sec
    last_updated: Instant,
}
```

### 8.4.2 Path Discovery

Paths are discovered by:
- Enumerating local interfaces (`eth0`, `wlan0`, `wwan0`, etc.)
- For each, building candidate `(local_addr, remote_addr)` pairs

Remote address choices:
- Server address from configuration
- Additional server addresses from mesh discovery or DNS

Path lifecycle:
1. Create candidate path in **Probing** state
2. Send small test payload (PING)
3. Measure RTT/loss
4. If acceptable → mark **Active**
5. If repeated failures → mark **Failed**

### 8.4.3 Scheduling Strategies

We use flexible schedulers (Chapter 5 introduced them, here we detail behavior):

#### 8.4.3.1 Round-Robin

- Simplest: cycle through active paths per frame or per batch
- Good for paths with similar RTT and bandwidth

#### 8.4.3.2 Weighted

- Each path has a weight proportional to estimated bandwidth
- Scheduler sends frames such that long-term proportion matches weights

```rust
struct WeightedScheduler {
    weights: Vec<f64>,   // per-path
    cumulative: Vec<f64>,
    rng: rand::rngs::ThreadRng,
}

impl Scheduler for WeightedScheduler {
    fn select_path(&mut self) -> PathId {
        let x: f64 = self.rng.gen();
        for (i, &c) in self.cumulative.iter().enumerate() {
            if x < c { return PathId(i as u32); }
        }
        PathId((self.cumulative.len() - 1) as u32)
    }
}
```

#### 8.4.3.3 Latency-Aware

- Preference toward low RTT paths, but still utilize high-bandwidth paths for bulk traffic
- Potential strategy:
  - Small/interactive flows → prefer lowest RTT
  - Large/bulk flows → weighted by bandwidth

#### 8.4.3.4 Redundant (for Critical Traffic)

- Certain frames (e.g., small control or interactive voice) can be sent on **multiple paths simultaneously** to reduce perceived latency

Trade-off:
- Higher bandwidth cost for increased reliability/latency improvement

### 8.4.4 Performance Profiles (from httpf)

httpf validated four named profiles that map to scheduler configurations:

| Profile | Scheduler | Behavior |
|---------|-----------|----------|
| `latency` | Latency-Aware | Prefer lowest-RTT path, backup only on failure |
| `balanced` | Weighted | Distribute across paths proportional to bandwidth |
| `throughput` | Weighted + Aggressive | Maximize aggregate bandwidth, tolerate reordering |
| `max_performance` | All paths + Redundant | Use every path, duplicate critical frames |

```toml
[performance]
profile = "balanced"  # latency | balanced | throughput | maxperformance
```

These profiles are syntactic sugar over the raw scheduler parameters, providing users a simple knob while preserving full customizability via `[network.multipath]` config.

### 8.4.5 Reordering & Congestion

Multipath introduces reordering:
- Packets on high-RTT paths may arrive later than those on low-RTT paths

Mitigations:
- QUIC’s own reordering tolerance (on streams)
- Small reordering buffer per flow at QuicEther encapsulation level (if necessary)

Congestion control:
- We rely primarily on QUIC’s per-path congestion control mechanisms (as provided by library)
- Scheduler should consider congestion feedback when assigning frames

---

## 8.5 Integration with TAP & Bridging

### 8.5.1 Outbound Path

1. Application writes to socket → kernel builds Ethernet frame → TAP `quicether0`
2. QuicEther reads Ethernet frame from TAP
3. All frames go through QUIC tunnel to server (Virtual Hub)
4. Connection manager selects existing QUIC connection to server
5. Multipath scheduler chooses active path for this frame
6. Frame encapsulated and sent via QUIC

### 8.5.2 Inbound Path

1. QUIC receives encapsulated payload on any path
2. QuicEther decrypts & decapsulates to original Ethernet frame
3. Write to TAP (kernel delivers to local application)
4. On server side: Virtual Hub switches frame by destination MAC

Interaction detail:
- Routing layer does not care which path was used; it only deals with logical connection
- Multipath layer sits between QUIC endpoint and UDP socket operations

---

## 8.6 Failure Handling in Multipath

### 8.6.1 Single Path Failure

Detection:
- Missing acknowledgments
- Repeated PING timeouts
- QUIC library signals path failure

Response:
1. Mark path as **Failed**
2. Remove from scheduler’s active set
3. Continue sending on remaining paths
4. Periodically probe the failed path for recovery

User impact:
- Increased latency / reduced throughput
- No connection drops

### 8.6.2 All Paths Fail

If all paths become Failed:

1. Mark connection as **Degraded**
2. Enter exponential backoff reconnect attempts
3. Notify higher layers (for status reporting)

Application behavior:
- Depending on app, may experience timeouts
- Once any path recovers, traffic resumes

### 8.6.3 Path Quality Changes

Paths can degrade without fully failing:
- Increased RTT
- Increased loss

Multipath manager:
- Updates `PathMetrics`
- Scheduler shifts more traffic to better paths
- Potentially demotes poor paths to backup / low-priority use

---

## 8.7 Performance Considerations

### 8.7.1 Overheads

Compared to kernel VPNs (e.g., WireGuard):
- User-space QUIC adds:
  - Encryption overhead (TLS 1.3 via QUIC, plus optional ChaCha20-Poly1305 inner layer)
  - Stream framing overhead
  - Copy overhead between user and kernel
  - LZ4 compression/decompression cost

Mitigations (validated in httpf):
- **FrameBatch** — amortize per-frame overhead by batching 4-8 Ethernet frames per QUIC stream write
- **LZ4 compression** — fast compression reduces bytes on wire; net win for most traffic
- Batch reads/writes from TAP via `sendmmsg`/`recvmmsg` where applicable
- Use send/recv buffers tuned to NIC and MTU
- Use Rust with `tokio` and async I/O for high concurrency

### 8.7.2 MTU and Fragmentation

- Underlying network MTU often 1500 bytes
- QUIC + encapsulation overhead reduces effective MTU for tunneled IP

Strategy:
- TAP MTU configured to something like 1420 bytes (accounts for Ethernet + QUIC overhead)
- Encourage Path MTU Discovery on QUIC side
- Avoid IP fragmentation in the overlay

### 8.7.3 CPU Utilization

Target:
- Keep CPU <50% at 10 Gbps on modern server CPUs

Approaches:
- Parallelism: multiple cores handling independent flows
- Lock-free data structures where appropriate
- Avoid per-frame heap allocations

---

## 8.8 Configuration Examples

### 8.8.1 Simple Single-Path QUIC

```bash
# Connect to server with single interface, no multipath
quicether connect \
  --server vpn.example.com:4433 \
  --interfaces eth0 \
  --multipath disable
```

Use case:
- Simpler deployments where multi-ISP is not needed

### 8.8.2 Basic Multipath

```bash
# Connect with WiFi + LTE for aggregation
quicether connect \
  --server vpn.example.com:4433 \
  --interfaces wlan0,wwan0 \
  --multipath split
```

Behavior:
- Both interfaces used concurrently
- Scheduler defaults to weighted or round-robin

### 8.8.3 Advanced Multipath with Policies

```toml
[performance]
profile = "balanced"

[network.multipath]
mode = "split"             # or "failover"
interfaces = ["eth0", "eth1", "wwan0"]

[network.multipath.weights]
eth0 = 1.0   # Office LAN
eth1 = 2.0   # Fast fiber
wwan0 = 0.5  # Expensive LTE

[network.multipath.policies]
# Interactive traffic prefer low-latency
interactive_max_rtt_ms = 50
# Bulk traffic can use all paths
bulk_min_bandwidth_mbps = 5
```

---

## 8.9 Testing & Validation

### 8.9.1 Lab Scenarios

1. **Roaming:**
   - Move a laptop between WiFi networks and LTE
   - Verify connections remain established

2. **Path Failure:**
   - Keep two interfaces up
   - Yank cable on one, simulate outage
   - Confirm flows continue on remaining path

3. **Bandwidth Aggregation:**
   - Two 100 Mbps links → expect ~180–190 Mbps effective

4. **High Latency Path:**
   - Combine local broadband with high-latency satellite
   - Verify scheduler prefers broadband for interactive flows

### 8.9.2 Metrics to Watch

- Per-path RTT and loss
- Aggregate throughput
- Reordering events
- CPU usage per core

---

## Summary

In this chapter we:
- Described how QuicEther uses QUIC connections and streams for client-server tunneling
- Detailed the **FrameBatch** encapsulation format (proven in httpf)
- Designed a multipath layer with paths, schedulers, and **performance profiles**
- Explained server-based connection lifecycle with three-tier authentication
- Covered failure handling and performance considerations

Multipath over QUIC is a **core differentiator** of QuicEther, enabling:
- Aggregation of multiple ISPs
- Seamless roaming
- Resilient connections across unstable networks
- **Named performance profiles** (latency/balanced/throughput/max_performance) for simple tuning

**Next Chapter:** We will focus on the security and zero-trust aspects (authentication, authorization, and policy enforcement) in depth.

---

**Chapter Navigation:**
- [← Previous: Chapter 7 - Server Mesh & Discovery](./07-server-mesh-and-discovery.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 9 - Security & Zero-Trust](./09-security-and-zero-trust.md)
