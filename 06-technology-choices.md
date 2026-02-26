# Chapter 6: Technology Choices & Trade-offs

## Introduction

This chapter explains **why** QuicEther uses specific technologies and **what we consciously did *not* choose**.

We focus on decisions that shape the architecture:
- Transport protocol
- Cryptography and identity
- Discovery mechanism
- Data plane integration (L2 vs L3)
- Implementation language and runtime

For each decision we cover:
- Options considered
- Rationale for the chosen option
- Trade-offs and limitations

---

## 6.1 Transport: Why QUIC (Not TCP, Not Raw UDP, Not WireGuard)

### Options Considered

1. **TCP + TLS**
2. **Raw UDP + custom reliability layer**
3. **WireGuard protocol**
4. **QUIC (UDP-based, RFC 9000)**

### 6.1.1 TCP + TLS

**Pros:**
- Mature, battle-tested
- Easy to use (sockets API)
- TLS 1.3 widely implemented

**Cons:**
- Head-of-line blocking across all streams
- No built-in multipath
- Connection breaks on IP/port change (bad for roaming)
- Kernel-controlled congestion control (harder to experiment)

**Verdict:** Not suitable as the core tunnel transport for a roaming, multipath VPN.

### 6.1.2 Raw UDP + Custom Protocol

**Pros:**
- Maximum flexibility
- Can design exactly what we need
- Kernel bypass friendly (DPDK, XDP)

**Cons:**
- Re-implement reliability, congestion control, loss recovery
- Re-implement security handshake (error-prone)
- Hard to get right; decades of research already exist

**Verdict:** Reinventing TCP/QUIC is a bad idea. Security and performance are fragile.

### 6.1.3 WireGuard Protocol

**Pros:**
- Very simple, elegant design
- Extremely fast in kernel implementations
- Widely deployed, audited

**Cons:**
- No multipath
- No connection migration
- No concept of streams (packet-based only)
- Tightly coupled to specific key management model

**Verdict:** Fantastic for single-path, static tunnels; misfit for a dynamic, multipath overlay.

### 6.1.4 QUIC (Chosen)

**Pros:**
- Runs over UDP (user-space control)
- Built-in:
  - Reliable streams
  - Congestion control
  - Loss recovery
  - Connection migration (IP/port changes)
  - 0-RTT resumption
- Ongoing multipath extensions (IETF drafts)
- Integrated with TLS 1.3 (no custom crypto)

**Cons:**
- More complex than TCP or WireGuard
- User-space implementation adds CPU overhead compared to kernel VPNs
- Multipath not yet fully standardized across all stacks

**Why QUIC Fits QuicEther:**
- Connection migration directly solves roaming (Sarah, Maria)
- Multipath extensions enable Priya's and James's use cases
- User-space control aligns with need for rapid iteration

**Implementation Detail:**
- Rust `quinn` or similar library as the base
- Abstraction layer to allow swapping QUIC library if needed

---

## 6.2 Crypto & Identity: Why TLS 1.3 + Ed25519

### Options Considered

1. Custom handshake + symmetric keys
2. TLS 1.2
3. TLS 1.3 with RSA certificates
4. TLS 1.3 with Ed25519 keys (Chosen)

### 6.2.1 Custom Handshake

**Pros:**
- Complete control over protocol

**Cons:**
- Easy to introduce critical vulnerabilities
- Requires constant security review
- Reinvents what TLS 1.3 already provides

**Verdict:** Too risky and unnecessary.

### 6.2.2 TLS 1.2

**Pros:**
- Widely supported

**Cons:**
- More complex, legacy ciphersuites
- Slower handshakes
- More attack surface than TLS 1.3

**Verdict:** No reason to start with a legacy protocol.

### 6.2.3 TLS 1.3 + RSA

**Pros:**
- Familiar to many ops teams

**Cons:**
- Larger keys and signatures
- Slower on constrained devices

**Verdict:** Acceptable but not ideal for a global, node-to-node mesh.

### 6.2.4 TLS 1.3 + Ed25519 (Chosen)

**Pros:**
- Fast on both servers and clients
- Small keys and signatures
- Well-studied and widely recommended

**Identity Model (proven in httpf):**
- Each node has an Ed25519 keypair
- IdentityId = `qe_<blake3(public_key)[..16]>`
- BLAKE3 for identity hashing (fast, secure, no length-extension attacks)
- TLS 1.3 mutual authentication binds identity to transport connection

**Additional Crypto (validated in httpf):**
- **BLAKE3** for all hashing (identity derivation, integrity)
- **ChaCha20-Poly1305** for optional per-packet encryption layer
- **X25519** for ECDH key exchange
- **Argon2id** for password hashing (legacy auth fallback)
- **ML-KEM-768** post-quantum readiness (future)

**Three-Tier Auth Model (proven in httpf):**
1. **Ed25519 Identity** — primary, challenge-response
2. **Password Auth** — Argon2id-hashed, legacy fallback
3. **Service Tokens** — machine-to-machine (mesh, cascade, admin)

**Trade-offs:**
- Requires TLS library support for Ed25519
- Some enterprise PKI tooling is still RSA-centric (bridge tools may be needed later)

---

## 6.3 Discovery: Why Server Mesh (Not DHT, Not Gossip)

> **httpf validation:** HTTP Fabric started with a simple server-based model
> and extended it to server mesh for multi-region. This proved simpler, more
> reliable, and easier to operate than a DHT would have been. Kademlia DHT
> is preserved as a future extension for very large (10,000+ node) deployments.

### Options Considered

1. Central database / coordination server
2. Static config files (hosts lists)
3. Gossip protocol (SWIM, HyParView)
4. Kademlia-style DHT
5. Server mesh with cascade routing (Chosen)

### 6.3.1 Central Database

**Pros:**
- Simple to implement
- Easy to query

**Cons:**
- Single point of failure
- Doesn't scale without significant ops burden

**Verdict:** Works for single-server deployments (which QuicEther supports).

### 6.3.2 Static Config

**Pros:**
- Very simple
- Works for tiny deployments

**Cons:**
- No auto-discovery
- Manual management overhead

**Verdict:** OK for early testing, not for multi-site setups.

### 6.3.3 Gossip Protocols

**Pros:**
- Great for membership information and failure detection
- Naturally resilient

**Cons:**
- Harder to answer queries like "who owns subnet 10.1.0.0/16?"
- Typically approximate / eventually consistent views

**Verdict:** Useful for liveness but not sufficient as the primary lookup structure.

### 6.3.4 Kademlia DHT (Deferred to v2.0+)

**Pros:**
- Logarithmic lookup time (O(log N))
- Proven at Internet scale (BitTorrent, IPFS)
- Naturally distributed with no central authority

**Cons:**
- Complex to implement correctly (Sybil resistance, churn handling)
- Not needed for hub-based deployments up to thousands of nodes
- httpf proved server mesh handles real-world scale

**Verdict:** Excellent technology, but overkill for v1.0. Deferred to future extension.

### 6.3.5 Server Mesh with Cascade Routing (Chosen)

**Pros:**
- Simple configuration (just list peer URLs + tokens)
- Proven by httpf in production
- Cascade routing handles cross-region traffic naturally
- No complex protocol — just QUIC connections between servers
- Subnet-based routing tables build automatically from mesh config

**Data Stored (per mesh peer):**
- Peer URL and subnet ownership
- Connection health (RTT, state)
- Routing table entries for cascade forwarding

**Trade-offs:**
- Requires at least one server (not purely P2P)
- Mesh topology is configured, not discovered

**Why It Fits:**
- Directly supports the hub-based multi-tenancy model
- Simple to deploy and debug
- httpf validated this at scale

---

## 6.4 Data Plane: L2 vs L3, TUN vs TAP

### Options Considered

1. L2 bridge (Ethernet, TAP devices)
2. L3 routed overlay (IP, TUN devices) — **Chosen**

### 6.4.1 L2 Bridge (TAP)

**Pros:**
- Transparent to applications (acts like a big Ethernet switch)
- No need to think about IP addressing per node

**Cons:**
- Broadcast / multicast storms in large networks
- Poor scalability across WANs
- Difficult policy enforcement (everything shares a flat L2 domain)

**Verdict:** Good for small homelabs; dangerous at larger scales.

### 6.4.2 L3 Overlay (TUN) — Chosen

**Pros:**
- Clear routing semantics (IP subnets)
- Works naturally with existing routing protocols
- Easier to enforce policies (subnet-based)
- Better fit for enterprise site-to-site use cases

**Cons:**
- Requires planning IP address space
- Some L2-only protocols won't work directly (e.g., some discovery protocols)

**Why TUN over TAP:**
- Personas focus on **site-to-site** and **remote access**, not arbitrary L2 extension
- Helps avoid misconfiguration that can melt networks

**Mitigation for L2 Needs:**
- Optional L2 proxy/bridge component in specific subnets (future work)

---

## 6.5 Implementation Language: Why Rust (Not Go, Not C++)

### Options Considered

1. C / C++
2. Go
3. Rust (Chosen)

### 6.5.1 C / C++

**Pros:**
- Maximum control over performance
- Huge ecosystem

**Cons:**
- Memory safety issues (use-after-free, buffer overflows)
- Harder to write correct concurrent code
- Security review burden much higher

**Verdict:** Too risky for a security-critical networking project.

### 6.5.2 Go

**Pros:**
- Easy concurrency (goroutines)
- Fast iteration, good tooling
- Large standard library

**Cons:**
- Garbage collector (GC) adds latency jitter
- Harder to hit 100 Gbps line-rate
- Less control over memory layout and zero-copy operations

**Verdict:** Good productivity, but GC and performance ceiling conflict with performance-as-a-feature.

### 6.5.3 Rust (Chosen)

**Pros:**
- Memory safety without GC
- Zero-cost abstractions
- Excellent async story (`tokio`, `async-std`)
- Strong type system prevents many bugs

**Cons:**
- Steeper learning curve
- Compile times longer than Go

**Why It Fits:**
- Security by default (no UB) + performance
- Already good ecosystem for QUIC, TLS, async I/O

**Implementation Notes:**
- `tokio` as async runtime
- `quinn`/`s2n-quic` (or similar) for QUIC
- `rustls` for TLS 1.3

---

## 6.6 Multipath: Why QUIC Multipath (Not MPTCP, Not VPN Bonding Tricks)

### Options Considered

1. MPTCP (Multipath TCP)
2. Traditional VPN bonding (multiple tunnels + ECMP)
3. QUIC with multipath extensions (Chosen)

### 6.6.1 MPTCP

**Pros:**
- IETF standard
- Implemented in Linux kernel

**Cons:**
- Requires kernel support at *both* ends
- Hard to deploy in heterogeneous environments (Windows, macOS, mobile)
- Tightly coupled to TCP semantics

**Verdict:** Not portable enough for our targets.

### 6.6.2 Traditional VPN Bonding

**Pattern:**
- Create multiple VPN tunnels, use ECMP or application-level striping.

**Pros:**
- Works with existing protocols

**Cons:**
- Reordering and head-of-line issues
- Complex configuration
- No integrated path awareness at transport level

**Verdict:** Too brittle and operationally complex.

### 6.6.3 QUIC Multipath (Chosen)

**Pros:**
- Operates at transport level, aware of RTT/loss per path
- Plays well with QUIC's stream and congestion control model
- Can blend with connection migration

**Cons:**
- Standardization still in progress
- Libraries differ in level of support

**Why It Fits:**
- Aligns perfectly with direct connection + roaming + multipath combo
- User-space control lets us experiment and iterate

---

## 6.7 NAT Traversal: Server-Based (with STUN for Future P2P)

### Options Considered

1. STUN + UDP hole punching
2. TURN-style relays
3. Server-based routing (Chosen)

### 6.7.1 Server-Based Routing (Chosen)

**Pros (validated in httpf):**
- Clients connect to server — server always has a public IP (or port-forwarded)
- No NAT traversal needed for client → server direction
- Server mesh handles server-to-server routing
- Cascade routing enables multi-hop through servers

**Cons:**
- Requires a server with reachable IP
- All traffic passes through server

**Why It Fits:**
- httpf proved this model works for all deployment scenarios
- A $5/month VPS solves the reachability problem
- QUIC connection migration handles client roaming

### 6.7.2 STUN + UDP Hole Punching (Future)

**Pros:**
- Would enable direct client-to-client (P2P) connections
- Reduces server load

**Status:** Deferred to v2.0+ when DHT-based P2P extension is built.

### 6.7.3 TURN-Style / Vendor Relays

**Verdict:** Conflicts with user sovereignty; explicit non-goal.

---

## 6.8 Hashing: Why BLAKE3 (Not SHA-1, Not SHA-256)

QuicEther uses **BLAKE3** for all hashing operations:

| Use Case | Algorithm | Why |
|----------|-----------|-----|
| Identity ID derivation | BLAKE3 | Fast, no length-extension attacks |
| Data integrity | BLAKE3 | 10× faster than SHA-256 on modern CPUs |
| Key derivation (HKDF) | BLAKE3 | Native KDF mode |
| Password hashing | Argon2id | Memory-hard, resistant to GPU attacks |

**Why Not SHA-1:**
- Broken (collision attacks demonstrated)
- 160-bit output is too small

**Why Not SHA-256:**
- BLAKE3 is faster (especially on modern CPUs with SIMD)
- BLAKE3 has a native KDF and MAC mode
- Both are equally secure for our use cases

**Validated in httpf:** BLAKE3 used throughout for identity derivation and integrity checks.

---

## 6.9 Summary of Choices

| Area | Chosen | Alternatives Rejected | Primary Reason |
|------|--------|-----------------------|----------------|
| Transport | QUIC | TCP, raw UDP, WireGuard | Multipath, migration, user-space control |
| Crypto | TLS 1.3 + Ed25519 + BLAKE3 | Custom, TLS 1.2, RSA, SHA-1 | Security, performance |
| Discovery | Server mesh (explicit peers) | Central DB, DHT, gossip | Proven in httpf, simple, debuggable |
| Data Plane | L3 TUN | L2 TAP | Scalability, policy control |
| Language | Rust | Go, C/C++ | Safety + performance |
| Multipath | QUIC multipath | MPTCP, bonding hacks | Portability, integration |
| NAT | Server-based routing | TURN, vendor relays, hole punching | User sovereignty, proven model |
| Hashing | BLAKE3 | SHA-1, SHA-256 | Speed, security, native KDF |

These technology choices directly implement the principles defined in Chapter 4, support the architecture in Chapter 5, and have been validated through building httpf.

**Next Chapter:** We will zoom into the server mesh and discovery layer in detail (mesh protocol, peer management, route exchange).

---

**Chapter Navigation:**
- [← Previous: Chapter 5 - High-Level Architecture](./05-architecture.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 7 - Server Mesh & Discovery](./07-server-mesh-and-discovery.md)
