# Chapter 9: Security & Zero‑Trust

## Introduction

Security is not a bolt‑on for QuicEther—it is **foundational**.

This chapter expands Principle 7 (Security by Default) into a concrete security model:
- Identity and authentication (who are you?)
- Authorization and policy (what can you access?)
- Encryption (how is data protected in transit?)
- Audit and observability (who did what, when?)
- Operational practices (keys, rotation, compromise handling)

We explicitly **do not** aim for anonymity; we aim for **confidential, authenticated, authorized** connectivity.

---

## 9.1 Threat Model (High Level)

### 9.1.1 Adversaries

We assume the following potential adversaries:

1. **Passive Network Observer**
   - Can see packets on the wire
   - Cannot easily break modern crypto

2. **Active Network Attacker**
   - Can inject, drop, and replay packets
   - Can attempt on‑path attacks (MITM)

3. **Compromised Node**
   - An endpoint (laptop, server) is fully compromised
   - Attacker has access to node’s private key and config

4. **Malicious Participant**
   - Legitimate credentials, but behaves badly in the overlay
   - Attempts hub abuse, policy violations, IP spoofing, etc.

5. **Curious Infrastructure Provider**
   - ISP, cloud provider hosting gateways or bootstrap nodes
   - Can see source/destination IPs and traffic timing, but not contents

### 9.1.2 Non‑Goals

We **do not** aim to defend against:
- Global passive adversaries capable of large‑scale traffic correlation
- Nation‑state actors targeting individuals over long time horizons
- Local compromise of the endpoints themselves (if your laptop is owned, all bets are off)

For those threat models, systems like Tor and specialized hardened OSes are more appropriate.

---

## 9.2 Identity & Authentication

### 9.2.1 Node Identity

Each node has a long-lived Ed25519 keypair:
- `priv`: private signing key (must be protected)
- `pub`: public key (registered with server or exchanged via config)

NodeId derived using BLAKE3 (not SHA-1):
- `NodeId = BLAKE3(pub)` (256-bit identifier, truncated to 160-bit for compatibility)

BLAKE3 was chosen over SHA-1 for:
- Cryptographic strength (no known attacks)
- Speed (3x faster than SHA-256 on modern CPUs)
- Tree-hashing support for future extensibility

### 9.2.2 Three-Tier Authentication (Proven in httpf)

QuicEther supports three authentication methods, validated in httpf:

| Tier | Method | Use Case |
|------|--------|----------|
| 1 | **Ed25519 Identity** | Cryptographic identity; highest security |
| 2 | **Password** (Argon2id) | Simple setup for personal/small deployments |
| 3 | **Service Token** | Pre-shared key for server mesh peers |

Authentication flow:
1. QUIC connection established with TLS 1.3
2. Client sends auth credentials on control stream
3. Server validates against configured auth method
4. On success: assigns hub, allocates virtual IP, opens data streams
5. On failure: connection rejected, event logged to audit trail

### 9.2.3 TLS 1.3 Transport Security

Every QUIC connection is protected by TLS 1.3.

- Each side presents a certificate containing its public key
- Certificates may be:
  - Self‑signed (small/private networks)
  - Signed by an internal CA (enterprise networks)

Verification steps:
1. Validate TLS certificate (chain of trust or pinning)
2. Extract public key → compute `NodeId`
3. Map connection to `NodeId` for policy and logging

This prevents classic MITM attacks as long as key distribution is correct.

### 9.2.4 Key Distribution Models

Different deployments can choose different models:

1. **Personal/Small Deployment:**
   - Password auth or static Ed25519 keys
   - Public keys exchanged via out-of-band channels (QR codes, SSH, email)
   - Trust anchor: user themselves

2. **Small Business:**
   - Password auth with Argon2id hashing
   - Simple internal CA or static trust store
   - Admin provisions keys/certs on devices

3. **Enterprise:**
   - Ed25519 identity with PKI integration (Active Directory, Vault, etc.)
   - Automated enrollment/renewal
   - Service tokens for server mesh interconnection

QuicEther does **not** hardcode a single PKI model; it exposes flexible hooks.

---

## 9.3 Authorization & Zero‑Trust Policy

Authentication says **who** you are. Authorization answers **what** you can access.

### 9.3.1 Zero‑Trust Principle

Default stance:
- **Deny by default, allow by policy**
- No implicit trust based on network location or IP address alone

### 9.3.2 Policy Model (Proven in httpf)

QuicEther implements two complementary policy engines, validated in httpf:

**Firewall Engine** (Proxmox-style ACL):
- Per-hub packet filtering rules
- First-match evaluation order
- Supports: allow/deny, src/dst IP, port, protocol, ICMP type

```toml
# Firewall rules (first match wins)
[[firewall.rules]]
action = "allow"
src = "10.100.0.0/24"
dst = "10.100.0.0/24"
comment = "Allow intra-hub traffic"

[[firewall.rules]]
action = "allow"
src = "10.100.0.0/24"
dst = "0.0.0.0/0"
port = "80,443"
protocol = "tcp"
comment = "Allow web traffic"

[[firewall.rules]]
action = "deny"
src = "0.0.0.0/0"
dst = "0.0.0.0/0"
comment = "Default deny"
```

**Policy Engine** (per-identity L3/L4 rules):

```toml
# /etc/quicether/policy.toml

[[rules]]
id = "dev-to-hq"
subject = "node_sarah_laptop"    # NodeId or alias
object = "10.0.0.0/16"           # Subnet
action = "allow"

[[rules]]
id = "sales-to-hq-readonly"
subject = "group:sales"
object = "10.0.0.0/16"
action = "allow"
constraints = { methods = ["GET"], ports = [80, 443] }

[[rules]]
id = "deny-all-else"
subject = "*"
object = "*"
action = "deny"
```

### 9.3.3 Policy Evaluation (httpf's Proven Model)

Implementation from httpf:

```rust
enum Action { Allow, Deny }

struct RequestContext {
    src_node: NodeId,
    dest_ip: IpAddr,
    dest_port: u16,
    protocol: TransportProto, // TCP/UDP/ICMP
    time: DateTime<Utc>,
}

trait PolicyEngine {
    fn evaluate(&self, ctx: &RequestContext) -> Action;
}
```

Evaluation steps:
1. Translate NodeId → policy subject (node, group, role)
2. Match against rules in priority order
3. First matching rule wins
4. Default: deny

This aligns with enterprise practices (firewalls, IAM systems).

### 9.3.4 Groups and Roles

To avoid per‑node explosion in large deployments:

- Nodes can belong to **groups** (e.g., `developers`, `sales`, `admins`)
- Policies can target groups instead of individual nodes

```toml
[groups.developers]
members = ["node_sarah_laptop", "node_alex_desktop"]

[groups.sales]
members = ["node_maria_laptop", "node_remote_sales1"]
```

---

## 9.4 Encryption & Confidentiality

### 9.4.1 Encryption Scope

All traffic across QuicEther tunnels is encrypted:
- Control plane (mesh protocol, control streams)
- Data plane (encapsulated IP packets via PacketBatch)

There is **no plaintext mode**.

### 9.4.2 Algorithms

Primary cipher suite (chosen for httpf, carried forward):
- **ChaCha20-Poly1305** — AEAD encryption (fast on non-AES-NI hardware, e.g., mobile)
- **X25519** — Ephemeral key exchange (ECDH)
- **BLAKE3** — Hashing (identity derivation, integrity)
- **Ed25519** — Signing (identity keys)

TLS 1.3 cipher suites supported:
- `TLS_CHACHA20_POLY1305_SHA256` (preferred)
- `TLS_AES_128_GCM_SHA256` (fallback on AES-NI hardware)

Post-quantum readiness:
- **ML-KEM-768** (NIST FIPS 203) hybrid key exchange prepared for future activation
- Architecture supports cipher agility without protocol changes

Properties:
- AEAD (Authenticated Encryption with Associated Data)
- Integrity and confidentiality in one step
- No plaintext mode — ever

### 9.4.3 Forward Secrecy

TLS 1.3 provides forward secrecy via ephemeral key exchange:
- Compromise of long‑term keys does **not** automatically allow decryption of past sessions
- However, a live compromised node can of course see its current plaintext traffic

---

## 9.5 Audit Logging & Observability

Enterprises (and many small businesses) need an audit trail.

### 9.5.1 What We Log (httpf's Proven Model)

Without leaking sensitive payloads, httpf validated logging:
- Connection events:
  - `timestamp, identity, src_ip, virtual_ip, hub, bytes_sent, bytes_recv, duration`
- Policy decisions:
  - `timestamp, identity, src_ip, dst_ip, action (allow|deny), rule_id`
- Security-relevant events:
  - Failed authentications (with rate limiting on log volume)
  - Firewall denials
  - Session creation/destruction
  - Admin API access

### 9.5.2 Log Format

Structured logs (JSONL — JSON Lines, one event per line):

```json
{"ts":"2025-11-20T10:15:32Z","event":"session_created","identity":"sarah","hub":"office","virtual_ip":"10.100.0.5","src_ip":"203.0.113.42"}
{"ts":"2025-11-20T10:15:33Z","event":"firewall_deny","identity":"sarah","src":"10.100.0.5","dst":"10.200.0.1","port":22,"rule":"deny-ssh"}
```

Logs can be shipped to:
- Local file (default) — rotated by size/time
- Syslog (RFC 5424) — native integration
- Central log collectors (Elastic, Loki, Splunk) via sidecar agents
- Prometheus metrics endpoint for real-time monitoring

### 9.5.3 Privacy Considerations

We **never** log:
- Packet payloads
- Application‑level data (HTTP paths, SQL queries, etc.)

We aim for:
- Enough metadata for troubleshooting and compliance
- Minimal exposure of sensitive content

---

## 9.6 Key Management & Rotation

### 9.6.1 Storage of Private Keys

On each node:
- Private key stored on disk, ideally:
  - File permissions: owner‑only
  - Optionally encrypted at rest (passphrase, OS keyring, TPM)

Location example:
- Linux: `/var/lib/quicether/keys/node.key`
- User mode: `~/.config/quicether/node.key`

### 9.6.2 Rotation Strategy

Goals:
- Limit impact window if a key is compromised
- Avoid unnecessary downtime

Possible strategies:

1. **Manual Rotation:**
   - Admin generates new keypair
   - Updates trust store / PKI
   - Restarts node

2. **Automated Rotation:**
   - Periodic key rollover via admin API
   - Coordinated update of server configuration and policy

### 9.6.3 Handling Compromise

If a node key is suspected compromised:

1. Mark its identity as **revoked** in server policy or revocation list
2. Propagate revocation:
   - Update server configuration to reject the identity
   - Distribute updated policy to all mesh peers
3. Rotate key on affected node (if still controllable)
4. Review audit logs for suspicious activity from that identity

---

## 9.7 Security of Servers & Hubs

Server nodes have special responsibilities:
- They manage hubs, sessions, and route all client traffic
- Mesh peers forward traffic across server instances

Risks:
- Server becomes attractive target
- Server sees metadata (who connects, hub assignments, traffic volume)

Mitigations:
- Strong hardening (OS, firewall, minimal services)
- Strict policy on what hubs allow and for whom
- Rate limiting (token bucket per identity) to prevent abuse
- Anti-spoofing on virtual NAT (reject packets with wrong source IP)
- Robust monitoring and alerting via Prometheus metrics + audit logs

Note: Servers see IP metadata but the QUIC tunnel between client and server provides encryption. For cross-hub traffic via mesh, servers only route PacketBatch payloads.

---

## 9.8 Interaction with Existing Security Controls

QuicEther should integrate with, not replace, existing security frameworks.

### 9.8.1 Firewalls

- QuicEther typically listens on a single UDP port (e.g., 443/udp or 51820/udp)
- Per‑subnet and per‑host controls still enforced by local firewalls behind QuicEther

### 9.8.2 IDS/IPS

- Encrypted transport reduces deep inspection capabilities
- However, IDS/IPS can still:
  - Inspect metadata (flow counts, sizes, anomalies)
  - Monitor gateway nodes for abnormal patterns

### 9.8.3 SIEM Integration

- Audit logs and metrics can feed into SIEM systems
- Enables correlation with other infrastructure events

---

## 9.9 Security Posture Across Personas

Mapping back to Chapter 3 personas:

- **Sarah (Developer):**
  - Needs secure remote access without complex VPN configs
  - Benefit: TLS 1.3 by default, zero‑trust policy defined by her employer

- **James (Small Business Owner):**
  - Needs simple but robust security
  - Benefit: Default‑deny rules with clear allowlists for office resources

- **Alex (Homelab Enthusiast):**
  - Wants to expose services safely to friends
  - Benefit: Node‑based policies instead of port‑knocking hacks

- **Maria (Digital Nomad):**
  - Needs to trust WiFi networks as little as possible
  - Benefit: End‑to‑end encryption over any network, no plaintext

- **Priya (Rural User):**
  - Primary concern: reliability and cost, but still benefits from strong defaults

- **David (Enterprise IT):**
  - Needs auditability, compliance, centralized policy
  - Benefit: First‑class IAM‑like controls, logs, and integration points

---

## 9.10 Security Limitations & Future Work

Honest limitations in v0.1:

- No peer-to-peer direct connections (all traffic routes through server)
- Basic revocation model (policy-based, not cryptographically enforced at protocol level)
- No built-in hardware token support initially (YubiKey, smartcards)

Planned directions:

- Direct peer connections via STUN/TURN for reduced latency
- Stronger identity binding (hardware authenticators)
- Fine-grained policy conditions (device posture, geo, time windows)
- ML-KEM-768 hybrid key exchange activation when ecosystem matures

---

## Summary

This chapter defined QuicEther's security model (grounded in httpf's implementation):

- **Identity:** Ed25519 keypairs, NodeId derived via BLAKE3
- **Authentication:** Three-tier model (Ed25519 + Password/Argon2id + Service Tokens)
- **Authorization:** Zero-trust with dual engines — Firewall (Proxmox-style ACL) + Policy (per-identity L3/L4)
- **Confidentiality:** ChaCha20-Poly1305/X25519, encrypted by default, no plaintext mode
- **Audit:** JSONL structured logs + syslog + Prometheus metrics
- **Operations:** Clear guidance on key management, rotation, and server hardening
- **Post-quantum:** ML-KEM-768 readiness

QuicEther aims to be **safe by default**, while remaining flexible enough for small labs and large enterprises alike.

**Next Chapter:** We will focus on the VPN interface and packet handling (L3 overlay, routing, and interaction with OS networking).

---

**Chapter Navigation:**
- [← Previous: Chapter 8 - QUIC Transport & Multipath](./08-quic-and-multipath.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 10 - VPN Interface & Routing](./10-vpn-interface-and-routing.md)
