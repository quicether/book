# Chapter 10: VPN Interface & Routing

## Introduction

This chapter explains how QuicEther integrates with the host networking stack:
- How the virtual VPN interface works (TUN device)
- How packets enter and leave the overlay
- How routing decisions are made (local vs remote vs gateway)
- How subnet advertisement and forwarding interact

We stay at the L3 (IP) layer; L2 bridging is out of scope for v0.1.

---

## 10.1 The Virtual Interface (TUN)

### 10.1.1 Why TUN (Not TAP)

As decided in Chapter 6:
- We operate at **L3**, not L2
- We route IP packets, not Ethernet frames

TUN characteristics:
- Presents a virtual IP interface (`quicether0`)
- Kernel routes IP packets to/from this interface
- Userspace (QuicEther) reads/writes raw IP packets via a file descriptor

### 10.1.2 Interface Creation

On Linux, QuicEther roughly performs:

```bash
# Conceptual equivalent
ip tuntap add dev quicether0 mode tun
ip addr add 100.64.0.1/16 dev quicether0
ip link set quicether0 up
```

On macOS and Windows, OS-specific APIs create an equivalent TUN.

### 10.1.3 Addressing

Default addressing model for the overlay:
- Use CGNAT range `100.64.0.0/10` for point‑to‑point overlay addressing by default
- For site‑to‑site, use private RFC1918 ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)

Examples:
- Personal mesh: each node gets `/32` in `100.64.0.0/10`
- Enterprise: sites advertise internal subnets (10.x.y.z) via QuicEther

---

## 10.2 Routing Model

QuicEther needs to decide, for each IP packet:
- Is it for **this node**?
- Is it for a subnet **we own**?
- Is it for a subnet **another node owns**?
- Should it go via a **gateway**?

### 10.2.1 Local Routing Table (Overlay View)

Conceptual structure:

```rust
struct OverlayRoute {
    destination: IpNet,      // 10.0.0.0/16
    next_hop: RouteNextHop,  // Local, NodeId, Gateway
}

enum RouteNextHop {
    Local,
    Node(NodeId),
    Gateway(NodeId),
}

struct OverlayRoutingTable {
    routes: Vec<OverlayRoute>,
    default_gateway: Option<NodeId>,
}
```

Lookup:
- Longest‑prefix match on `destination`
- If multiple routes match, choose most specific

### 10.2.2 Interaction with OS Routing

Two routing layers exist:

1. **OS Routing Table:**
   - Decides which packets go to `quicether0`
   - Configured with `ip route` (Linux) or equivalent

2. **Overlay Routing Table (inside QuicEther):**
   - Decides where to send packets read from `quicether0`

Example (Linux):

```bash
# Send all traffic for 10.0.0.0/16 to quicether0
ip route add 10.0.0.0/16 dev quicether0

# Default route remains via physical gateway
ip route add default via 192.168.1.1 dev eth0
```

### 10.2.3 Ownership of Subnets

Each node may **advertise** subnets it owns:
- e.g., `server1` advertises `192.168.1.0/24`

Advertising has two effects:
1. Published in DHT (`SUBNET:<CIDR>` record)
2. Installed in local `OverlayRoutingTable` as `Local` route

Other nodes:
- Query DHT for `SUBNET:<CIDR>`
- Install route `destination = CIDR, next_hop = Node(owner)`

---

## 10.3 Packet Flow: End‑to‑End Examples

### 10.3.1 Remote Access (Personal VPN)

Scenario: Sarah’s laptop → home server (192.168.1.10)

1. **Configuration:**
   - Home node:
     ```bash
     quicether start \
       --advertise-subnet 192.168.1.0/24
     ```
   - Sarah’s laptop:
     ```bash
     ip route add 192.168.1.0/24 dev quicether0
     ```

2. **Data Flow:**
   ```text
   app (laptop) → TCP connect to 192.168.1.10
             ↓
   kernel: route 192.168.1.0/24 via quicether0
             ↓
   TUN quicether0: QuicEther reads IP packet
             ↓
   overlay routing: dest 192.168.1.10 ∈ 192.168.1.0/24 → next_hop = Node(home)
             ↓
   QUIC: send to Node(home)
             ↓
   Home node: receives packet, writes to its kernel via TUN
             ↓
   kernel (home): route to 192.168.1.10 via local LAN
             ↓
   home server: receives packet
   ```

### 10.3.2 Site‑to‑Site

Scenario: HQ (10.0.0.0/16) ↔ Factory (10.1.0.0/16)

1. **Configuration:**
   - HQ gateway:
     ```bash
     quicether start --advertise-subnet 10.0.0.0/16
     ```
   - Factory gateway:
     ```bash
     quicether start --advertise-subnet 10.1.0.0/16
     ```
   - On HQ LAN routers/hosts:
     ```bash
     # Route 10.1.0.0/16 via HQ gateway (overlay entry)
     ip route add 10.1.0.0/16 via 10.0.0.1  # HQ gateway IP
     ```

2. **Data Flow (HQ → Factory):**
   ```text
   host at HQ sends packet to 10.1.0.50
             ↓
   HQ router: dest 10.1.0.0/16 → next hop = 10.0.0.1 (HQ gateway)
             ↓
   HQ gateway kernel: sees packet for 10.1.0.50
             ↓
   OS route: 10.1.0.0/16 via quicether0
             ↓
   TUN → QuicEther → overlay route: 10.1.0.0/16 owned by Node(factory)
             ↓
   QUIC tunnel to factory
             ↓
   factory gateway writes packet to local LAN
   ```

---

## 10.4 Gateway Behavior (Non‑Relay, Routing Only)

Gateways are normal nodes configured to **forward** for specific subnets or nodes.

### 10.4.1 Role Configuration

Example:

```bash
# Cloud gateway for Alex’s home subnet
quicether start \
  --role gateway \
  --forward-subnet 192.168.1.0/24
```

Gateway effects:
- Advertises in DHT that it can forward for that subnet
- Installs overlay route: `192.168.1.0/24` → `Local` (because it forwards into local LAN or through some path)

Other nodes lacking direct path to subnet owner:
- Query DHT for `GW_SUBNET:192.168.1.0/24`
- Route via gateway NodeId when direct connection fails

### 10.4.2 Packet Processing at Gateway

Gateway receives encapsulated packet destined for some IP not local to the gateway’s own TUN address but within a forwarded subnet.

Path:
- QuicEther decapsulates IP packet
- Checks `OverlayRoutingTable`:
  - If `dest ∈ forwarded_subnet`:
    - Writes packet to TUN or appropriate local interface
  - Else:
    - Forwards via overlay to another node per overlay routing

### 10.4.3 Policy Enforcement

Gateways apply zero‑trust policy as in Chapter 9:
- Each forwarded packet checked against policy rules
- If not permitted: drop + log

---

## 10.5 Routing Algorithms & Conflict Resolution

### 10.5.1 Longest‑Prefix Match

When multiple routes could match a destination:
- Always choose the most specific subnet

Example:

- Routes:
  - `10.0.0.0/8 → Node(A)`
  - `10.0.1.0/24 → Node(B)`
- Destination: `10.0.1.42`
  - Use `/24` route → Node(B)

### 10.5.2 Route Priority

If there is a **Local** and a **Gateway** route for the same prefix:
- Prefer `Local` (direct ownership)

If there is a **Node** and a **Gateway** route:
- Prefer `Node` (direct connection) when possible

This aligns with Principle 3 (Direct Connection is Sacred).

### 10.5.3 Default Gateway in Overlay

Some nodes may set a default overlay gateway for unknown subnets:

```toml
[overlay]
default_gateway = "node_datacenter_gateway"
```

Effects:
- Packets whose destination does not match any specific overlay route go to this gateway
- Typical for:
  - Priya’s desktop (all internet traffic through cloud gateway)
  - Remote workers sending all corp‑bound traffic via HQ gateway

---

## 10.6 Interaction with NAT & Local Networks

### 10.6.1 Host‑Only Mode vs Router Mode

Nodes may operate in two broad modes:

1. **Host‑Only:**
   - Only traffic originating from the host uses QuicEther
   - OS routing table directs specific prefixes to `quicether0`

2. **Router/Gateway:**
   - Node forwards traffic for other machines (acts as router)
   - IP forwarding enabled in OS
   - OS routes from LAN interfaces into `quicether0`

Example enabling Linux forwarding:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### 10.6.2 NAT on Edge Gateways

For internet access via QuicEther gateway:
- Gateway may perform NAT between overlay addresses and public IP

Example (Linux iptables/nftables):

```bash
# Masquerade overlay subnet out to internet via eth0
iptables -t nat -A POSTROUTING -s 100.64.0.0/10 -o eth0 -j MASQUERADE
```

This is **outside** QuicEther; we leverage existing OS firewall/NAT tools.

---

## 10.7 Configuration Patterns

### 10.7.1 Personal Laptop

- Only routes specific subnets (home, office) through QuicEther

```bash
quicether start
ip route add 192.168.1.0/24 dev quicether0   # home
ip route add 10.0.0.0/16 dev quicether0      # office
```

### 10.7.2 Always‑On Gateway (Home)

```bash
# QuicEther gateway on home server
quicether start \
  --advertise-subnet 192.168.1.0/24 \
  --role gateway

# Home router: route overlay addresses to gateway
ip route add 100.64.0.0/10 via 192.168.1.10  # gateway IP
```

### 10.7.3 Split‑Tunnel vs Full‑Tunnel

- **Split‑Tunnel:** Only corporate/homelab subnets go through overlay
- **Full‑Tunnel:** All traffic goes through overlay gateway

Full‑tunnel example (remote worker):

```bash
# Route everything through QuicEther
ip route add default dev quicether0

# Exceptions (local LAN, captive portal, etc.)
ip route add 192.168.0.0/16 via 192.168.1.1 dev wlan0
```

---

## 10.8 Performance & Scaling Considerations

### 10.8.1 Packet Processing Path

We must keep per‑packet overhead minimal:
- Avoid unnecessary copies between TUN buffer and QUIC
- Batch packets when possible

Implementation hints:
- Pre‑allocated buffers for TUN reads
- Reuse encapsulation headers
- Use async I/O with bounded task queues

### 10.8.2 Route Table Size

For large deployments:
- Many subnets may be advertised
- Per‑node route table must still be efficient

Approaches:
- Use prefix tree (radix/patricia trie) for O(log N) lookups
- Cache hot routes

### 10.8.3 Control vs Data Separation

Overlay routing table is configured by:
- DHT results
- Static overrides from config
- Policy engine (may inject denies or exceptions)

Data plane should avoid heavy locks on the routing table:
- Use read‑optimized structures (e.g., RCU, lock‑free tries) where appropriate

---

## Summary

In this chapter we:
- Described how QuicEther uses a TUN interface (`quicether0`) to integrate with OS networking
- Defined the overlay routing model (local, node, gateway, default)
- Walked through end‑to‑end packet flows for personal VPN and site‑to‑site
- Clarified the role of gateways as **routing** nodes (not opaque relays)
- Covered routing decisions, conflict resolution, NAT interaction, and configuration patterns

This completes the core data plane story: from packets on the host, through QuicEther, across QUIC, and back into remote networks.

**Next Chapter:** We will move up to the **daemon architecture & CLI/API design**, explaining how users and operators interact with QuicEther day‑to‑day.

---

**Chapter Navigation:**
- [← Previous: Chapter 9 - Security & Zero‑Trust](./09-security-and-zero-trust.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 11 - Daemon & CLI Architecture](./11-daemon-and-cli.md)
