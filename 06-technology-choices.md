# Chapter 6: Technology Choices & Trade‑offs

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
- Re‑implement reliability, congestion control, loss recovery
- Re‑implement security handshake (error‑prone)
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

**Verdict:** Fantastic for single‑path, static tunnels; misfit for a dynamic, multipath overlay.

### 6.1.4 QUIC (Chosen)

**Pros:**
- Runs over UDP (user‑space control)
- Built‑in:
  - Reliable streams
  - Congestion control
  - Loss recovery
  - Connection migration (IP/port changes)
  - 0‑RTT resumption
- Ongoing multipath extensions (IETF drafts)
- Integrated with TLS 1.3 (no custom crypto)

**Cons:**
- More complex than TCP or WireGuard
- User‑space implementation adds CPU overhead compared to kernel VPNs
- Multipath not yet fully standardized across all stacks

**Why QUIC Fits QuicEther:**
- Connection migration directly solves roaming (Sarah, Maria)
- Multipath extensions enable Priya's and James's use cases
- User‑space control aligns with need for rapid iteration

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

**Verdict:** Acceptable but not ideal for a global, node‑to‑node mesh.

### 6.2.4 TLS 1.3 + Ed25519 (Chosen)

**Pros:**
- Fast on both servers and clients
- Small keys and signatures
- Well‑studied and widely recommended

**Identity Model:**
- Each node has an Ed25519 keypair
- NodeId = hash(public_key)
- TLS 1.3 mutual authentication binds NodeId to transport connection

**Benefits:**
- Enables simple mapping: "which NodeId is this QUIC connection?"
- Policy engine can work purely in terms of NodeIds

**Trade‑offs:**
- Requires TLS library support for Ed25519 (we must pick an implementation that has this)
- Some enterprise PKI tooling is still RSA‑centric (bridge tools may be needed later)

---

## 6.3 Discovery: Why Kademlia DHT (Not Central DB, Not Gossip)

### Options Considered

1. Central database / coordination server
2. Static config files (hosts lists)
3. Gossip protocol (SWIM, HyParView)
4. Kademlia‑style DHT (Chosen)

### 6.3.1 Central Database

**Pros:**
- Simple to implement
- Easy to query

**Cons:**
- Single point of failure
- Re‑introduces vendor control if hosted by project
- Doesn’t scale to millions of nodes without significant ops burden

**Verdict:** Violates user sovereignty and distributed‑by‑default principles.

### 6.3.2 Static Config

**Pros:**
- Very simple
- Works for tiny deployments

**Cons:**
- No auto‑discovery
- Manual management overhead
- Breaks down beyond a handful of nodes

**Verdict:** OK for early testing, not for a self‑organizing network.

### 6.3.3 Gossip Protocols

**Pros:**
- Great for membership information and failure detection
- Naturally resilient

**Cons:**
- Harder to answer queries like "who owns subnet 10.1.0.0/16?"
- Typically approximate / eventually consistent views

**Verdict:** Useful for liveness but not sufficient as the primary lookup structure.

### 6.3.4 Kademlia DHT (Chosen)

**Pros:**
- Logarithmic lookup time (O(log N))
- Proven at Internet scale (BitTorrent, IPFS)
- Naturally distributed with no central authority

**Data Stored:**
- Node metadata: `NodeId → {addresses, capabilities}`
- Subnet advertisements: `CIDR → NodeId`
- Optional: bootstrap lists, gateway availability flags

**Trade‑offs:**
- Eventual consistency (not instant updates)
- Requires protection against Sybil attacks (later: integration with optional blockchain or PKI)

**Why It Fits:**
- Directly supports queries we need for routing
- Matches the "every node participates" principle

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
- Easier to enforce policies (subnet‑based)
- Better fit for enterprise site‑to‑site use cases

**Cons:**
- Requires planning IP address space
- Some L2‑only protocols won’t work directly (e.g., some discovery protocols)

**Why TUN over TAP:**
- Personas focus on **site‑to‑site** and **remote access**, not arbitrary L2 extension
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
- Memory safety issues (use‑after‑free, buffer overflows)
- Harder to write correct concurrent code
- Security review burden much higher

**Verdict:** Too risky for a security‑critical networking project.

### 6.5.2 Go

**Pros:**
- Easy concurrency (goroutines)
- Fast iteration, good tooling
- Large standard library

**Cons:**
- Garbage collector (GC) adds latency jitter
- Harder to hit 100 Gbps line‑rate
- Less control over memory layout and zero‑copy operations

**Verdict:** Good productivity, but GC and performance ceiling conflict with performance‑as‑a‑feature.

### 6.5.3 Rust (Chosen)

**Pros:**
- Memory safety without GC
- Zero‑cost abstractions
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
- Create multiple VPN tunnels, use ECMP or application‑level striping.

**Pros:**
- Works with existing protocols

**Cons:**
- Reordering and head‑of‑line issues
- Complex configuration
- No integrated path awareness at transport level

**Verdict:** Too brittle and operationally complex.

### 6.6.3 QUIC Multipath (Chosen)

**Pros:**
- Operates at transport level, aware of RTT/loss per path
- Plays well with QUIC’s stream and congestion control model
- Can blend with connection migration

**Cons:**
- Standardization still in progress
- Libraries differ in level of support

**Why It Fits:**
- Aligns perfectly with direct connection + roaming + multipath combo
- User‑space control lets us experiment and iterate

---

## 6.7 NAT Traversal: STUN + Hole Punching (Not TURN, Not Vendor Relays)

### Options Considered

1. STUN + UDP hole punching (Chosen)
2. TURN‑style relays
3. Proprietary relay network

### 6.7.1 STUN + UDP Hole Punching (Chosen)

**Pros:**
- Works in majority of NAT cases
- No vendor data plane required
- Lightweight, well‑understood

**Cons:**
- Fails for some symmetric NAT / firewall combinations

**Mitigation:**
- User‑deployed gateways for the hard cases

### 6.7.2 TURN‑Style Relays

**Pros:**
- Works even when hole punching fails

**Cons:**
- Requires relay servers that see all traffic
- Drifts toward vendor‑controlled infrastructure

**Verdict:** Conflicts with user sovereignty; we avoid generic relays.

### 6.7.3 Proprietary Relay Network

**Pros:**
- Monetization opportunity

**Cons:**
- Direct contradiction of project values

**Verdict:** Explicit non‑goal.

---

## 6.8 Optional Blockchain: When and Why

QuicEther can function entirely without a blockchain. The **optional** blockchain is considered only for:
- Hardening identity (preventing Sybil attacks in public meshes)
- Providing tamper‑evident logs for high‑compliance environments

### Why Not Always On

- Adds operational and cognitive complexity
- Not needed for personal / small business meshes
- Many users do not want to run blockchain infrastructure

### Design Direction (Future Work)

- Light‑weight Proof‑of‑Authority (PoA) chain for public node registry
- Used only in high‑trust, high‑compliance deployments

For MVP, we **do not implement** blockchain. It remains a design option for v1.x.

---

## 6.9 Summary of Choices

| Area | Chosen | Alternatives Rejected | Primary Reason |
|------|--------|-----------------------|----------------|
| Transport | QUIC | TCP, raw UDP, WireGuard | Multipath, migration, user‑space control |
| Crypto | TLS 1.3 + Ed25519 | Custom, TLS 1.2, RSA | Security, performance |
| Discovery | Kademlia DHT | Central DB, static, gossip‑only | Distributed, scalable lookups |
| Data Plane | L3 TUN | L2 TAP | Scalability, policy control |
| Language | Rust | Go, C/C++ | Safety + performance |
| Multipath | QUIC multipath | MPTCP, bonding hacks | Portability, integration |
| NAT | STUN + hole punching | TURN, vendor relays | User sovereignty |
| Blockchain | Optional later | Mandatory L1 dependency | Simplicity, pragmatism |

These technology choices directly implement the principles defined in Chapter 4 and support the architecture in Chapter 5.

**Next Chapter:** We will zoom into the DHT and discovery layer in detail (wire format, bucket management, security considerations).

---

**Chapter Navigation:**
- [← Previous: Chapter 5 - High-Level Architecture](./05-architecture.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 7 - DHT & Discovery](./07-dht-and-discovery.md)
