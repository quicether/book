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
   - Legitimate node key, but behaves badly in the overlay
   - Attempts DHT poisoning, policy abuse, etc.

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

Each node has a long‑lived Ed25519 keypair:
- `priv`: private signing key (must be protected)
- `pub`: public key (distributed via DHT and config)

NodeId derived as in Chapter 7:
- `NodeId = SHA1(pub)` (160‑bit identifier)

### 9.2.2 TLS 1.3 Mutual Authentication

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

### 9.2.3 Key Distribution Models

Different deployments can choose different models:

1. **Personal/Small Mesh:**
   - Keys generated locally
   - Public keys exchanged via out‑of‑band channels (QR codes, SSH, email)
   - Trust anchor: user themselves

2. **Small Business:**
   - Simple internal CA or static trust store
   - Admin provisions keys/certs on devices

3. **Enterprise:**
   - Integration with existing PKI (Active Directory Certificate Services, Vault, etc.)
   - Automated enrollment/renewal

QuicEther does **not** hardcode a single PKI model; it exposes flexible hooks.

---

## 9.3 Authorization & Zero‑Trust Policy

Authentication says **who** you are. Authorization answers **what** you can access.

### 9.3.1 Zero‑Trust Principle

Default stance:
- **Deny by default, allow by policy**
- No implicit trust based on network location or IP address alone

### 9.3.2 Policy Model

Policy is evaluated per **packet** (or per flow) using:
- Source NodeId
- Destination IP / subnet
- Optional metadata (tags, time of day, etc.)

Conceptual structure:

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

### 9.3.3 Policy Evaluation

Implementation concept:

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
- Control plane (DHT RPCs over QUIC, control streams)
- Data plane (encapsulated IP packets)

There is **no plaintext mode**.

### 9.4.2 Algorithms

Relying on TLS 1.3 and QUIC, typical cipher suites:
- `TLS_AES_128_GCM_SHA256`
- `TLS_CHACHA20_POLY1305_SHA256`

Properties:
- AEAD (Authenticated Encryption with Associated Data)
- Integrity and confidentiality in one step

### 9.4.3 Forward Secrecy

TLS 1.3 provides forward secrecy via ephemeral key exchange:
- Compromise of long‑term keys does **not** automatically allow decryption of past sessions
- However, a live compromised node can of course see its current plaintext traffic

---

## 9.5 Audit Logging & Observability

Enterprises (and many small businesses) need an audit trail.

### 9.5.1 What We Log

Without leaking sensitive payloads, we log:
- Connection events:
  - `timestamp, src_node, dest_node, src_ip, dest_ip, protocol, bytes_sent, bytes_recv, duration`
- Policy decisions:
  - `timestamp, src_node, dest_ip, action (allow|deny), rule_id`
- Security‑relevant events:
  - Failed authentications
  - Suspicious patterns (many denied attempts)

### 9.5.2 Log Format

Structured logs (JSON lines):

```json
{
  "ts": "2025-11-20T10:15:32Z",
  "event": "connection",
  "src_node": "node_sarah_laptop",
  "dest_node": "node_hq_gateway",
  "src_ip": "100.64.0.10",
  "dest_ip": "10.0.0.5",
  "protocol": "tcp",
  "bytes_sent": 123456,
  "bytes_recv": 234567,
  "duration_ms": 5023
}
```

Logs can be shipped to:
- Local file (default)
- Syslog
- Central log collectors (Elastic, Loki, Splunk) via sidecar agents

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
   - Periodic key rollover via management API
   - Coordinated update of DHT records and policy

### 9.6.3 Handling Compromise

If a node key is suspected compromised:

1. Mark its NodeId as **revoked** in central policy or revocation list
2. Propagate revocation:
   - Update DHT with a revocation record (for public meshes)
   - Distribute updated policy to all nodes
3. Rotate key on affected node (if still controllable)
4. Monitor logs for suspicious activity from that NodeId

---

## 9.7 Security of Gateways

Gateway nodes have special responsibilities:
- They forward traffic for others when direct paths fail

Risks:
- Gateway becomes attractive target
- Gateway sees metadata (who talks to whom, how much)

Mitigations:
- Strong hardening (OS, firewall, minimal services)
- Strict policy on what they can forward and for whom
- Robust monitoring and alerting on gateway behavior

Note: Even gateways **cannot** decrypt end‑to‑end payload beyond what is visible as IP metadata; the QUIC tunnel between endpoints remains encrypted.

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

- No global Sybil resistance for fully public meshes
- Basic revocation model (policy‑based, not cryptographically enforced at protocol level)
- No built‑in hardware token support initially (YubiKey, smartcards)

Planned directions:

- Optional PoA chain for registered nodes in public overlays
- Stronger identity binding (hardware authenticators)
- Fine‑grained policy conditions (device posture, geo, time windows)

---

## Summary

This chapter defined QuicEther’s security model:

- **Identity:** Ed25519 keypairs, NodeId derived from public keys
- **Authentication:** TLS 1.3 mutual auth on all QUIC connections
- **Authorization:** Zero‑trust, policy‑driven access per NodeId and subnet
- **Confidentiality:** Encrypted by default, no plaintext mode
- **Audit:** Structured logs for connections and policy decisions
- **Operations:** Clear guidance on key management, rotation, and gateways

QuicEther aims to be **safe by default**, while remaining flexible enough for small labs and large enterprises alike.

**Next Chapter:** We will focus on the VPN interface and packet handling (L3 overlay, routing, and interaction with OS networking).

---

**Chapter Navigation:**
- [← Previous: Chapter 8 - QUIC Transport & Multipath](./08-quic-and-multipath.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 10 - VPN Interface & Routing](./10-vpn-interface-and-routing.md)
