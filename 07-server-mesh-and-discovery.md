# Chapter 7: Server Mesh & Discovery

```mermaid
graph TD
  subgraph MeshCluster["Server Mesh"]
    A["Server A (Hub: office)"]
    B["Server B (Hub: dev)"]
    C["Server C (Hub: staging)"]
  end

  A <-->|"QUIC Cascade\n(service token auth)"| B
  B <-->|"QUIC Cascade\n(service token auth)"| C
  A <-->|"QUIC Cascade\n(service token auth)"| C

  C1["Client: alice\n(MAC 02:aa:01)"] --> A
  C2["Client: bob\n(MAC 02:aa:02)"] --> A
  C3["Client: charlie\n(MAC 02:bb:01)"] --> B
  C4["Client: dave\n(MAC 02:cc:01)"] --> C

  C1 -.->|"frame to 02:bb:01\nvia cascade tunnel"| C3
```

## Introduction

This chapter describes QuicEther's **server mesh** — the mechanism by which multiple QuicEther servers discover each other, exchange hub membership, and forward Ethernet frames between their respective client pools via cascade connections.

The server mesh replaces the Kademlia DHT that was originally envisioned in v1.0 of this book. After building HTTP Fabric (httpf), we learned that:

1. **Explicit server mesh is simpler and more reliable** than DHT-based discovery
2. **95% of real deployments** are served by client-server + mesh topology
3. **DHT adds complexity** without proportional benefit for initial versions
4. **Server mesh is trivially debuggable** — you can see exactly which peers are connected

DHT-based P2P discovery remains a future extension (see Chapter 16).

---

## 7.1 Mesh Architecture

### Overview

A QuicEther server mesh is a set of servers connected by persistent QUIC tunnels:

```
Server A (office hub — virtual Ethernet segment)
    ↕ QUIC cascade tunnel (service token auth, auto-reconnect)
Server B (dev hub — virtual Ethernet segment)
    ↕ QUIC cascade tunnel (service token auth, auto-reconnect)
Server C (staging hub — virtual Ethernet segment)
```

Each server:
- Manages its own hubs (Virtual Ethernet switches), clients, and sessions
- Connects to configured mesh peers via cascade tunnels
- Forwards Ethernet frames destined for peer hubs through cascade tunnels
- Automatically reconnects to peers on failure

### Mesh vs. Hub

| Concept | Scope | Purpose |
|---------|-------|---------|
| **Hub** | Single server | Virtual Ethernet switch (MAC table, firewall, policy) |
| **Mesh** | Multiple servers | Inter-server frame forwarding via cascade connections |

A server can have multiple hubs. The mesh connects servers so that clients in one server's hub can reach clients in another server's hub.

---

## 7.2 Mesh Configuration

### Server Config (TOML)

```toml
[server]
listen = "0.0.0.0:9443"

# Service token for authenticating mesh peers and cascade hops
service_token = "mesh-secret-token-abc123"

# Mesh peer configuration
[[mesh.peers]]
address = "server-b.example.com:9443"
token = "mesh-secret-token-xyz789"

[[mesh.peers]]
address = "server-c.example.com:9443"
token = "mesh-secret-token-def456"

# Local hubs
[[hubs]]
name = "office"
```

### Configuration Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mesh.peers[].address` | `String` | Yes | `host:port` of the mesh peer |
| `mesh.peers[].token` | `String` | Yes | Service token to authenticate to this peer |
| `service_token` | `String` | For mesh | Token that peers use to authenticate to us |

---

## 7.3 Mesh Connection Protocol

### Connection Establishment

```
Server A → Server B:
1. Open QUIC connection to Server B (TLS 1.3)
2. Send MeshConnect frame on control stream:
   {
     type: "mesh_connect",
     token: "mesh-secret-token-xyz789",
     server_id: "qe_server_a_id",
     hubs: [
       { name: "office" },
     ]
   }
3. Server B validates token against its service_token
4. Server B responds with MeshAccept:
   {
     type: "mesh_accept",
     server_id: "qe_server_b_id",
     hubs: [
       { name: "dev" },
     ]
   }
5. Both servers update their hub-to-peer mappings:
   - Server A: hub "dev" → mesh_peer_b
   - Server B: hub "office" → mesh_peer_a
6. Cascade tunnel is ACTIVE
```

### Wire Format (QUIC Streams)

Mesh connections use multiplexed QUIC streams:

| Stream | Purpose | Direction |
|--------|---------|-----------|
| Control stream (bi-directional) | Mesh handshake, keepalive, hub updates | Both |
| Data streams (uni-directional) | Frame forwarding | Both |

#### Control Stream Messages

```rust
enum MeshControlMessage {
    // Initial handshake
    MeshConnect {
        token: String,
        server_id: String,
        hubs: Vec<HubInfo>,
    },
    MeshAccept {
        server_id: String,
        hubs: Vec<HubInfo>,
    },
    MeshReject {
        reason: String,
    },

    // Ongoing
    Keepalive,
    HubUpdate {
        hubs: Vec<HubInfo>,
    },
}

struct HubInfo {
    name: String,
    client_count: u32,
}
```

#### Data Stream Packet Format

Ethernet frames forwarded through cascade tunnels use the same FrameBatch format as client data:

```
FrameBatch (cascade forwarding):
┌──────────────────┐
│ num_frames: u16   │
├──────────────────┤
│ frame_1_size: u16 │
│ frame_1_data      │  ← Raw Ethernet frame
├──────────────────┤
│ frame_2_size: u16 │
│ frame_2_data      │
├──────────────────┤
│ ...               │
└──────────────────┘
```

Frames are optionally compressed with LZ4 and encrypted with ChaCha20-Poly1305 (on top of QUIC's TLS encryption).

---

## 7.4 Frame Forwarding

### MAC-to-Peer Table Construction

Each server builds its forwarding table from two sources:

1. **Local hubs**: MACs learned from this server's own clients
2. **Mesh peers**: MACs learned from frames arriving via cascade tunnels

```rust
struct MeshForwardingTable {
    local_hubs: HashMap<String, Hub>,        // Hub name → local hub
    peer_hubs: HashMap<String, MeshPeerId>,  // Hub name → mesh peer
    mac_to_peer: HashMap<MacAddress, MeshPeerId>,  // Learned MACs from mesh
}

impl MeshForwardingTable {
    fn forward(&self, hub_name: &str, dst_mac: &MacAddress) -> RouteDecision {
        // 1. Check local hub's MAC table
        if let Some(hub) = self.local_hubs.get(hub_name) {
            if hub.mac_table.lookup(dst_mac).is_some() {
                return RouteDecision::Local(hub_name.to_string());
            }
        }

        // 2. Check learned MAC-to-peer mappings
        if let Some(peer_id) = self.mac_to_peer.get(dst_mac) {
            return RouteDecision::CascadeForward(peer_id.clone());
        }

        // 3. Unknown MAC in mesh — flood to all mesh peers
        RouteDecision::Flood
    }
}
```

### Route Propagation

When a mesh peer's hubs change (new hub added, hub removed, CIDR changed), the peer sends a `HubUpdate` message on the control stream:

```
Server B adds new hub "qa" (10.50.0.0/24):
1. Server B sends HubUpdate to all mesh peers
2. Server A receives HubUpdate, updates routing table:
   - 10.200.0.0/24 → mesh_peer_b (existing)
   - 10.50.0.0/24  → mesh_peer_b (new)
3. Clients on Server A can now reach 10.50.0.0/24
```

### Multi-Hop Routing

In a mesh with 3+ servers, packets may need to traverse multiple hops:

```
Server A (10.100.0.0/24) ←→ Server B (10.200.0.0/24) ←→ Server C (10.50.0.0/24)

If Server A has no direct mesh tunnel to Server C:
- Server A knows: 10.50.0.0/24 → mesh_peer_b (learned via HubUpdate relay)
- Packet from A's client to 10.50.0.2:
  1. A forwards to B via mesh tunnel
  2. B's routing table: 10.50.0.0/24 → mesh_peer_c
  3. B forwards to C via mesh tunnel
  4. C delivers to local client

TTL limit: max 4 hops (configurable) to prevent routing loops
```

---

## 7.5 Cascade Connections

Cascade connections (proven in httpf) are a specialized form of multi-hop where servers form explicit chains for geographic or privacy bridging:

```
Client → Server A (US) → Server B (EU) → Server C (Asia) → Destination
```

### Cascade vs. Mesh

| Feature | Mesh | Cascade |
|---------|------|---------|
| Purpose | Connect server sites | Geographic/privacy bridging |
| Topology | Any (star, ring, full) | Linear chain |
| Auth | Service tokens | Service tokens |
| Use case | Multi-site enterprise | Privacy, geo-bridging |

### Cascade Configuration

```toml
# Server A (entry point)
[cascade]
next_hop = "server-b.example.com:9443"
next_hop_token = "cascade-token-b"

# Server B (middle hop)
[cascade]
next_hop = "server-c.example.com:9443"
next_hop_token = "cascade-token-c"

# Server C (exit point)
# No cascade config — traffic exits here
```

---

## 7.6 Mesh Security

### Authentication

All mesh connections use **service tokens** (proven in httpf):

```
Server A → Server B:
  MeshConnect { token: "mesh-secret-token-xyz789", ... }

Server B validates:
  if request.token != self.config.service_token {
      return MeshReject { reason: "invalid token" };
  }
```

Service tokens are:
- Pre-shared secrets configured in TOML
- Unique per mesh peer relationship (recommended)
- Can be rotated without disconnecting clients

### Anti-Spoofing

Cascade tunnels enforce frame source validation:

```
Server A receives frame from Client X (session MAC: 02:aa:01):
1. Verify source MAC matches session: src_mac=02:aa:01 ✓
2. Forward to mesh (if destination is on another server)

Server B receives cascade frame from Server A:
1. Trust Server A's anti-spoofing (mesh peers are trusted after auth)
2. Switch to local client by MAC
```

### Rate Limiting

Mesh tunnels have configurable rate limits:

```toml
[mesh]
max_frames_per_second = 100000
burst = 500000
```

---

## 7.7 Failure Handling

### Peer Disconnection

```
Detection: QUIC connection keepalive timeout
  ↓
Mark peer as DISCONNECTED
  ↓
Auto-reconnect with exponential backoff:
  Attempt 1: wait 1s
  Attempt 2: wait 2s
  Attempt 3: wait 4s
  ...
  Max wait: 60s
  ↓
On reconnect:
  Re-authenticate with service token
  Exchange hub membership info (may have changed)
  Resume forwarding
  ↓
If peer unreachable for extended period:
  Remove peer MAC entries from forwarding table
  Frames to unknown MACs → flood to remaining peers only
  Continue retry loop indefinitely
```

### Split-Brain

If mesh connectivity is partially lost:

```
Before: A ↔ B ↔ C (all connected)
After:  A ↔ B    C (A-C tunnel down)

Result:
  - A can still reach C via B (multi-hop)
  - Routing automatically adapts
  - When A-C reconnects, direct route restored
```

---

## 7.8 Monitoring & Observability

### Mesh Health Metrics

```
# HELP quicether_mesh_peers_total Number of configured mesh peers
# TYPE quicether_mesh_peers_total gauge
quicether_mesh_peers_total 3

# HELP quicether_mesh_peers_connected Number of connected mesh peers
# TYPE quicether_mesh_peers_connected gauge
quicether_mesh_peers_connected 2

# HELP quicether_mesh_frames_forwarded Frames forwarded through mesh
# TYPE quicether_mesh_frames_forwarded counter
quicether_mesh_frames_forwarded{peer="server-b"} 1234567
```

### CLI Commands

```bash
# List mesh peers and status
quicether admin mesh status --server vpn.example.com
# Output:
# Peer                          | Status    | Hubs  | Uptime | Packets
# server-b.example.com:9443    | Connected | dev   | 2d 4h  | 1.2M
# server-c.example.com:9443    | Reconnect | qa    | -      | 0
```

---

## 7.9 Future: DHT Extension

In a future version (v2.0+), QuicEther may add optional DHT-based peer discovery:

- **Use case:** Large-scale P2P networks where explicit server configuration is impractical
- **Approach:** Kademlia-style DHT running alongside server mesh
- **Compatibility:** DHT discovery would supplement (not replace) server mesh
- **Identity:** Same Ed25519 + BLAKE3 identity system

This is documented further in Chapter 16 (Future Directions).

---

## Summary

In this chapter we:

- Defined the server mesh architecture (replacing the original DHT design)
- Specified mesh connection protocol (QUIC cascade tunnels + service token auth)
- Designed MAC forwarding table construction and hub membership propagation
- Described cascade connections for geographic/privacy chains
- Addressed security, failure handling, and monitoring
- Noted DHT as a future extension

The server mesh is the **connectivity backbone** enabling:
- Multi-site deployments (enterprise, homelab)
- Cross-hub Ethernet frame forwarding between servers
- Cascade connections for privacy/geo chains
- Automatic failover and reconnection

All of these patterns were validated in HTTP Fabric (httpf) and are being ported to native QUIC.

**Next Chapter:** We will dive into the QUIC transport and multipath implementation details.

---

**Chapter Navigation:**
- [← Previous: Chapter 6 - Technology Choices](./06-technology-choices.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 8 - QUIC Transport & Multipath](./08-quic-and-multipath.md)