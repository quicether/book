# Chapter 10: VPN Interface & Routing

## Introduction

This chapter explains how QuicEther integrates with the host networking stack:
- How the virtual VPN interface works (TUN device)
- How packets enter and leave the overlay
- How routing decisions are made (local vs server vs bridge vs mesh)
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

### 10.1.3 Addressing (Hub-Based IP Allocation)

QuicEther uses **hub-based** IP allocation, proven in httpf:
- Each hub defines a CIDR pool (e.g., `10.100.0.0/24`)
- Server assigns virtual IPs from the pool when clients connect
- Anti-spoofing ensures clients only use their assigned IP

Default ranges:
- Hub networks: private RFC1918 ranges (10.x.y.z/24, 172.16.x.y/24)
- CGNAT range `100.64.0.0/10` available for overlay point-to-point addressing
- For site-to-site bridge mode: advertise internal subnets through the hub

Examples:
- Personal server: single hub `10.100.0.0/24`, clients get `10.100.0.2-254`
- Enterprise: multiple hubs per department (`office: 10.100.0.0/24`, `dev: 10.100.1.0/24`)

---

## 10.2 Routing Model

QuicEther needs to decide, for each IP packet:
- Is it for **this node**?
- Is it for a subnet **we own**?
- Is it for a subnet **another node owns**?
- Should it go via a **bridge** or **mesh peer**?

### 10.2.1 Local Routing Table (Overlay View)

Conceptual structure:

```rust
struct OverlayRoute {
    destination: IpNet,      // 10.100.0.0/24
    next_hop: RouteNextHop,  // Local, Server, Bridge
}

enum RouteNextHop {
    Local,                    // This hub's subnet
    Server(SocketAddr),       // Route via QUIC server
    Bridge(SocketAddr),       // Route via bridge node
    MeshPeer(String),         // Route via mesh peer server
}

struct OverlayRoutingTable {
    routes: Vec<OverlayRoute>,
    default_route: Option<RouteNextHop>,
}
```

Lookup:
- Longest-prefix match on `destination`
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

### 10.2.3 Hub Subnet Ownership

Subnets are defined per hub in the server configuration:

```toml
[[hubs]]
name = "office"
subnet = "10.100.0.0/24"

[[hubs]]
name = "factory"
subnet = "10.100.1.0/24"
```

Routing effects:
1. Server installs hub subnets in its routing table as `Local`
2. Clients receive their hub's subnet info during connection setup
3. For cross-hub traffic: server routes via internal hub lookup or mesh peer
4. Bridge mode clients advertise additional LAN subnets through their hub

---

## 10.3 Packet Flow: End-to-End Examples

### 10.3.1 Remote Access (Personal VPN)

Scenario: Sarah's laptop → home server (192.168.1.10)

1. **Configuration:**
   - Home server:
     ```bash
     quicether server --config server.toml
     # server.toml defines hub with subnet 192.168.1.0/24
     ```
   - Sarah's laptop:
     ```bash
     quicether connect --server home.example.com:4433
     # TUN and routes configured automatically from hub info
     ```

2. **Data Flow:**
   ```text
   app (laptop) → TCP connect to 192.168.1.10
             ↓
   kernel: route 192.168.1.0/24 via quicether0
             ↓
   TUN quicether0: QuicEther reads IP packet
             ↓
   overlay routing: dest 192.168.1.10 ∈ hub subnet → route via server
             ↓
   QUIC: send to server
             ↓
   Home node: receives packet, writes to its kernel via TUN
             ↓
   kernel (home): route to 192.168.1.10 via local LAN (bridge mode)
             ↓
   home server: receives packet
   ```

### 10.3.2 Site-to-Site

Scenario: HQ (10.0.0.0/16) ↔ Factory (10.1.0.0/16)

1. **Configuration:**
   - HQ server:
     ```bash
     quicether server --config hq-server.toml
     # Defines hub "hq" with subnet 10.0.0.0/16
     ```
   - Factory server:
     ```bash
     quicether server --config factory-server.toml
     # Defines hub "factory" with subnet 10.1.0.0/16
     ```
   - Mesh connection between them:
     ```toml
     # In hq-server.toml
     [[mesh.peers]]
     address = "factory.example.com:4433"
     service_token = "mesh-secret"
     ```

2. **Data Flow (HQ → Factory):**
   ```text
   host at HQ sends packet to 10.1.0.50
             ↓
   HQ router: dest 10.1.0.0/16 → next hop via HQ server overlay
             ↓
   HQ server: sees packet for 10.1.0.50
             ↓
   OS route: 10.1.0.0/16 via quicether0
             ↓
   TUN → QuicEther → overlay route: 10.1.0.0/16 owned by factory hub (via mesh)
             ↓
   QUIC tunnel to factory server (via mesh peer connection)
             ↓
   factory server writes packet to local LAN
   ```

---

## 10.4 Bridge Mode (Site-to-Site Forwarding)

In httpf, **bridge mode** was validated as the way to connect on-premise LANs through the overlay. Bridge nodes are clients that also forward traffic for local subnets.

### 10.4.1 Role Configuration

Example:

```bash
# Bridge client connecting to cloud server, forwarding local LAN
quicether bridge \
  --server vpn.example.com:4433 \
  --local-subnet 192.168.1.0/24
```

Bridge effects:
- Connects to server like a normal client
- Advertises local subnet to the hub
- Forwards packets between local LAN and the overlay tunnel
- Other clients in the same hub can reach `192.168.1.0/24` through the bridge

### 10.4.2 Packet Processing at Bridge

Bridge receives encapsulated packet destined for an IP within its advertised local subnet.

Path:
- QuicEther decapsulates IP packet
- Checks destination against bridge's local subnet:
  - If `dest ∈ local_subnet`:
    - Writes packet to local network interface (not TUN)
  - Else:
    - Sends back to server for routing to correct hub/client

### 10.4.3 Policy Enforcement

Bridge nodes apply the same policy as the server:
- Each forwarded packet checked against firewall and policy rules
- Anti-spoofing: verify source IP matches bridge's assigned virtual IP or local subnet
- If not permitted: drop + log to audit trail

---

## 10.5 Routing Algorithms & Conflict Resolution

### 10.5.1 Longest-Prefix Match

When multiple routes could match a destination:
- Always choose the most specific subnet

Example:

- Routes:
  - `10.0.0.0/8 → Node(A)`
  - `10.0.1.0/24 → Node(B)`
- Destination: `10.0.1.42`
  - Use `/24` route → Node(B)

### 10.5.2 Route Priority

If there is a **Local** and a **MeshPeer** route for the same prefix:
- Prefer `Local` (direct ownership)

If there is a **Server** and a **Bridge** route:
- Prefer `Server` (direct connection) when possible

This aligns with Principle 3 (Direct Connection When Possible).

### 10.5.3 Default Route in Overlay

Some nodes may set a default overlay route for unknown subnets:

```toml
[routing]
default_route = "server"  # or specific mesh peer
```

Effects:
- Packets whose destination does not match any specific overlay route go to the server
- Typical for:
  - Priya's desktop (all internet traffic through server)
  - Remote workers sending all corp traffic via company server

---

## 10.6 Interaction with NAT & Local Networks

### 10.6.1 Client Mode vs Bridge Mode

Nodes may operate in two broad modes:

1. **Client (Host-Only):**
   - Only traffic originating from the host uses QuicEther
   - OS routing table directs specific prefixes to `quicether0`

2. **Bridge (Router):**
   - Node forwards traffic for local LAN machines
   - IP forwarding enabled in OS
   - OS routes from LAN interfaces into `quicether0`

Example enabling Linux forwarding:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### 10.6.2 NAT on Edge Servers

For internet access via QuicEther server:
- Server may perform NAT between overlay addresses and public IP

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
quicether connect --server home.example.com:4433
# Routes configured automatically from hub info
# Or manually add additional routes:
ip route add 10.0.0.0/16 dev quicether0      # office
```

### 10.7.2 Always-On Server (Home)

```bash
# QuicEther server on home server
quicether server --config server.toml
# server.toml defines hub with subnet and bridge support
```

### 10.7.3 Split-Tunnel vs Full-Tunnel

- **Split-Tunnel:** Only corporate/homelab subnets go through overlay (default)
- **Full-Tunnel:** All traffic goes through overlay server

```toml
# Full-tunnel configuration
[client]
server = "vpn.example.com:4433"
full_tunnel = true

# Split-tunnel with specific routes
[client]
server = "vpn.example.com:4433"
routes = ["10.0.0.0/8", "192.168.1.0/24"]
```

---

## 10.8 Performance & Scaling Considerations

### 10.8.1 Packet Processing Path

We must keep per-packet overhead minimal:
- Avoid unnecessary copies between TUN buffer and QUIC
- Batch packets when possible

Implementation hints:
- Pre-allocated buffers for TUN reads
- Reuse encapsulation headers
- Use async I/O with bounded task queues

### 10.8.2 Route Table Size

For large deployments:
- Many subnets may be advertised
- Per-node route table must still be efficient

Approaches:
- Use prefix tree (radix/patricia trie) for O(log N) lookups
- Cache hot routes

### 10.8.3 Control vs Data Separation

Overlay routing table is configured by:
- Server hub configuration
- Mesh peer route advertisements
- Bridge node subnet announcements
- Static overrides from config
- Policy engine (may inject denies or exceptions)

Data plane should avoid heavy locks on the routing table:
- Use read-optimized structures (e.g., RCU, lock-free tries) where appropriate

---

## Summary

In this chapter we:
- Described how QuicEther uses a TUN interface (`quicether0`) to integrate with OS networking
- Defined the hub-based IP allocation model with CIDR pools and anti-spoofing
- Defined the overlay routing model (local, server, bridge, mesh peer)
- Walked through end-to-end packet flows for personal VPN and site-to-site via mesh
- Clarified **bridge mode** for forwarding local LAN subnets through the overlay
- Covered routing decisions, conflict resolution, NAT interaction, and configuration patterns

This completes the core data plane story: from packets on the host, through QuicEther, across QUIC, and back into remote networks.

**Next Chapter:** We will move up to the **daemon architecture & CLI/API design**, explaining how users and operators interact with QuicEther day-to-day.

---

**Chapter Navigation:**
- [← Previous: Chapter 9 - Security & Zero-Trust](./09-security-and-zero-trust.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 11 - Daemon & CLI Architecture](./11-daemon-and-cli.md)
