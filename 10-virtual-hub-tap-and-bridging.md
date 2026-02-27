# Chapter 10: Virtual Hub, TAP & Bridging

## Introduction

This chapter explains how QuicEther integrates with the host networking stack at Layer 2:
- How the TAP device works (virtual Ethernet interface)
- How Ethernet frames enter and leave the overlay
- How the Virtual Hub performs MAC-based switching
- How Local Bridge connects physical LANs to the overlay

This is the core of the "virtual Ethernet switch over QUIC" model.

---

## 10.1 The Virtual Interface (TAP)

### 10.1.1 Why TAP (Not TUN)

As decided in Chapter 6:
- We operate at **L2**, not L3
- We switch Ethernet frames, not route IP packets
- Physical routers handle DHCP, ARP, DNS — no server-side IP management

TAP characteristics:
- Presents a virtual Ethernet interface (`quicether0`)
- Kernel treats it like a physical NIC
- Userspace (QuicEther) reads/writes raw Ethernet frames via a file descriptor
- Full L2 transparency: ARP, DHCP, broadcast all pass through natively

### 10.1.2 Interface Creation

On Linux, QuicEther roughly performs:

```bash
# Conceptual equivalent
ip tuntap add dev quicether0 mode tap
ip link set quicether0 up
# No IP address assigned by QuicEther — physical router provides via DHCP
```

On macOS, `feth` (fake Ethernet) or third-party TAP drivers create an equivalent virtual Ethernet adapter. On Windows, WinTap or similar virtual Ethernet adapters are used.

### 10.1.3 The Platform Problem: Virtual TAP

**The challenge:** TAP devices provide raw Ethernet frame access, but TAP is not available on every platform. iOS and Android only expose TUN (L3/IP) interfaces via their VPN APIs (`NEPacketTunnelProvider` on iOS, `VpnService` on Android). Containers and some restricted environments similarly lack TAP support.

This creates a fundamental tension: QuicEther's overlay protocol is L2 (Ethernet frames), but some clients can only produce and consume L3 (IP packets).

**The solution: Virtual TAP** — a userspace abstraction layer that translates between L2 and L3 at the client edge, allowing platforms without native TAP to participate fully in the L2 overlay.

#### Device Abstraction Layer

QuicEther defines a **Device Abstraction Layer** with two modes:

| Mode | Underlying Device | Platforms | How It Works |
|------|------------------|-----------|-------------|
| **Native TAP** | Real TAP device | Linux, macOS (feth), Windows (WinTap) | Read/write raw Ethernet frames directly |
| **Virtual TAP** | TUN device + userspace L2↔L3 translation | iOS, Android, containers, restricted environments | Wrap IP packets in Ethernet headers (outbound), strip headers (inbound) |

The overlay protocol is **always L2** — the server always sees Ethernet frames. The translation happens at the client edge only.

```rust
/// Device abstraction — both modes produce/consume Ethernet frames
/// for the QUIC tunnel, regardless of the underlying OS interface.
enum DeviceMode {
    /// Real TAP device. Reads/writes raw Ethernet frames.
    NativeTap(TapInterface),
    /// TUN device with userspace L2↔L3 translation.
    VirtualTap(VirtualTapInterface),
}

impl DeviceMode {
    /// Read an Ethernet frame (from TAP directly, or synthesized from TUN).
    async fn read_frame(&mut self) -> Result<Vec<u8>>;
    /// Write an Ethernet frame (to TAP directly, or stripped to IP for TUN).
    async fn write_frame(&mut self, frame: &[u8]) -> Result<()>;
}
```

#### Virtual TAP: L2↔L3 Translation

The Virtual TAP performs bidirectional translation so the overlay sees Ethernet frames while the OS sees IP packets:

**Outbound (L3→L2): App sends IP packet → Virtual TAP wraps it in an Ethernet frame**

```text
Application sends IP packet
  ↓
TUN interface: QuicEther reads raw IP packet
  ↓
Virtual TAP: lookup destination IP → ARP cache → destination MAC
  ↓
Virtual TAP: prepend 14-byte Ethernet header
  (dst MAC from ARP cache, src MAC = node's assigned MAC, ethertype = 0x0800)
  ↓
Result: complete Ethernet frame → send via QUIC tunnel to server
```

**Inbound (L2→L3): Server sends Ethernet frame → Virtual TAP strips it to IP packet**

```text
QUIC tunnel delivers Ethernet frame from server
  ↓
Virtual TAP: parse Ethernet header (14 bytes)
  ↓
Virtual TAP: learn source MAC → source IP mapping from frame
  ↓
Virtual TAP: strip Ethernet header → raw IP packet
  ↓
TUN interface: write IP packet to OS network stack
```

**ARP Proxy: Virtual TAP handles ARP locally**

Since there is no real Ethernet interface, ARP requests from the overlay cannot reach the OS network stack. The Virtual TAP intercepts and responds to ARP:

```text
Overlay sends ARP request: "Who has 10.0.0.5?"
  ↓
Virtual TAP: check ARP cache for 10.0.0.5
  ↓
If known: build ARP reply (10.0.0.5 is at <cached MAC>) → send back to overlay
If unknown: generate ARP reply with node's own MAC → learn the mapping
```

The Virtual TAP also **learns MAC-to-IP mappings** by observing:
- ARP requests and replies flowing through the overlay
- Source MAC + source IP from Ethernet frame payloads

```rust
struct VirtualTapInterface {
    tun: TunInterface,              // OS-level TUN device (L3)
    mac_address: MacAddress,        // Assigned MAC for this node (02:xx:xx:xx:xx:xx)
    arp_cache: HashMap<Ipv4Addr, MacAddress>,   // IP → MAC mappings
    mac_to_ip: HashMap<MacAddress, Ipv4Addr>,   // Reverse: MAC → IP
    gateway_mac: MacAddress,        // Virtual gateway MAC (for default route)
}

impl VirtualTapInterface {
    /// Outbound: read IP packet from TUN, wrap in Ethernet frame.
    async fn read_frame(&mut self) -> Result<Vec<u8>> {
        let ip_packet = self.tun.read_packet().await?;
        let dst_ip = extract_dst_ip(&ip_packet)?;
        let dst_mac = self.arp_cache.get(&dst_ip)
            .unwrap_or(&self.gateway_mac);
        
        let mut frame = Vec::with_capacity(ETH_HEADER_SIZE + ip_packet.len());
        frame.extend_from_slice(&dst_mac.0);        // 6 bytes dst MAC
        frame.extend_from_slice(&self.mac_address.0); // 6 bytes src MAC
        frame.extend_from_slice(&ETHERTYPE_IPV4);    // 2 bytes ethertype
        frame.extend_from_slice(&ip_packet);         // IP payload
        Ok(frame)
    }
    
    /// Inbound: receive Ethernet frame, strip header, write IP to TUN.
    async fn write_frame(&mut self, frame: &[u8]) -> Result<()> {
        if frame.len() < ETH_HEADER_SIZE {
            return Ok(()); // Runt frame
        }
        
        let ethertype = u16::from_be_bytes([frame[12], frame[13]]);
        
        match ethertype {
            ETHERTYPE_ARP => {
                // Handle ARP locally — learn mappings, send replies
                self.handle_arp(&frame[ETH_HEADER_SIZE..]).await
            }
            ETHERTYPE_IPV4 | ETHERTYPE_IPV6 => {
                // Learn source MAC→IP from frame
                let src_mac = MacAddress::from(&frame[6..12]);
                if let Some(src_ip) = extract_src_ip(&frame[ETH_HEADER_SIZE..]) {
                    self.arp_cache.insert(src_ip, src_mac);
                }
                // Strip Ethernet header, write IP packet to TUN
                self.tun.write_packet(&frame[ETH_HEADER_SIZE..]).await
            }
            _ => Ok(()) // Drop non-IP ethertypes
        }
    }
}
```

> **Existing code validation:** httpf's `src/network/packet.rs` already contains
> all the building blocks for Virtual TAP: `ETH_HEADER_SIZE = 14`, `MacAddress`
> type with `BROADCAST`/`ZERO`/`random()`/`httpf_random()`, `ArpPacket` struct,
> `parse_arp()`, `build_arp_reply()`, and ethertype constants. The Virtual TAP
> assembles these proven primitives into a coherent L2↔L3 translation layer.

#### Auto-Detection

QuicEther selects the device mode automatically based on platform capabilities:

| Platform | Device Mode | Reason |
|----------|-------------|--------|
| Linux (desktop) | Native TAP | Full TAP support via `/dev/net/tun` |
| macOS | Native TAP | `feth` or third-party TAP drivers |
| Windows | Native TAP | WinTap or similar |
| iOS | Virtual TAP | Only `NEPacketTunnelProvider` (TUN/L3) available |
| Android | Virtual TAP | Only `VpnService.Builder` (TUN/L3) available |
| Docker / containers | Virtual TAP (fallback) | TAP may require `--cap-add=NET_ADMIN`; Virtual TAP works without |

Config override (see Chapter 15):
```toml
[network]
device_mode = "auto"   # "auto" | "tap" | "virtual_tap"
```

#### Transparency Guarantee

From the server's perspective, there is no difference between a Native TAP client and a Virtual TAP client:
- Both send and receive Ethernet frames over QUIC
- Both have assigned MAC addresses
- Both participate in the same Virtual Hub broadcast domain
- The MAC learning table treats them identically

The only difference is invisible: on Virtual TAP clients, ARP is handled in userspace rather than by the kernel, and Ethernet headers are synthesized rather than produced by the OS network stack.

### 10.1.4 Addressing (Physical Router via DHCP)

QuicEther does **not** assign IP addresses. Instead:
- The TAP interface joins a Virtual Hub (Ethernet segment)
- DHCP requests from the client pass through the overlay as normal Ethernet frames
- The physical router (or DHCP server) on the bridged LAN responds
- The client receives its IP address, gateway, DNS — same as being on the physical LAN

This is the key difference from L3 VPNs:
- No CIDR pool management on the server
- No IP allocation logic
- The physical network infrastructure handles all L3 concerns

Optional: If no physical router is available (e.g., cloud-only deployment), **SecureNAT** mode provides built-in DHCP and NAT within the Virtual Hub.

Examples:
- Personal server: single hub `home`, clients DHCP from home router and get `192.168.1.x`
- Enterprise: multiple hubs per department (`office`, `dev`), each bridged to their physical LAN

---

## 10.2 Switching & Forwarding Model

QuicEther needs to decide, for each Ethernet frame:
- Is it for a MAC address on **this hub**?
- Is it for a MAC address on a **cascaded hub**?
- Is it **broadcast/multicast** (flood to all ports)?
- Is it **unknown unicast** (flood or drop)?

### 10.2.1 MAC Learning Table

Conceptual structure:

```rust
struct MacEntry {
    mac: MacAddress,
    port: SwitchPort,     // Which session/bridge owns this MAC
    last_seen: Instant,   // For aging
}

enum SwitchPort {
    Session(SessionId),           // Connected client
    LocalBridge(BridgeId),        // Physical LAN bridge
    CascadeTunnel(PeerId),        // Mesh peer hub
}

struct MacTable {
    entries: HashMap<MacAddress, MacEntry>,
    aging_timeout: Duration,      // Default: 300s
    max_entries: usize,           // Default: 8192
}
```

Lookup:
- Exact MAC address match (no prefix matching like IP routing)
- If MAC not found → unknown unicast → flood to all ports except source
- Broadcast/multicast → always flood to all ports except source

### 10.2.2 Interaction with OS Networking

Two layers exist:

1. **OS Network Stack:**
   - Treats `quicether0` as a regular Ethernet interface
   - ARP, DHCP, and all L2 protocols work normally
   - IP configuration comes from DHCP (physical router)

2. **Virtual Hub (inside QuicEther):**
   - Switches frames between sessions based on MAC addresses
   - Learns MACs from source addresses of incoming frames
   - Ages out stale entries

Unlike L3 VPNs, there is no overlay routing table — the Virtual Hub is a transparent Ethernet switch.

### 10.2.3 Virtual Hub Configuration

Hubs are defined in the server configuration:

```toml
[[hubs]]
name = "office"
# No CIDR — it's an Ethernet segment, not an IP subnet

[[hubs]]
name = "factory"

[hubs.secure_nat]
enabled = false  # Physical router handles DHCP/ARP
```

Switching effects:
1. Each hub is an independent broadcast domain (like a VLAN)
2. Clients joining a hub can communicate at L2 — as if on the same physical switch
3. Cross-hub traffic requires a router connected to both hubs (or cascade connections)
4. Local Bridge attaches physical LANs to a hub

---

## 10.3 Frame Flow: End-to-End Examples

### 10.3.1 Remote Access (Personal VPN)

Scenario: Sarah's laptop → home server (192.168.1.10)

1. **Configuration:**
   - Home server:
     ```bash
     quicether server --config server.toml
     # server.toml defines hub "home" with local bridge to eth0
     ```
   - Sarah's laptop:
     ```bash
     quicether connect --server home.example.com:4433
     # TAP interface created, joins "home" hub
     # DHCP request through tunnel → home router responds
     # Laptop gets 192.168.1.x from home router
     ```

2. **Data Flow:**
   ```text
   app (laptop) → TCP connect to 192.168.1.10
             ↓
   kernel: ARP for 192.168.1.10 → Ethernet frame
             ↓
   TAP quicether0: QuicEther reads Ethernet frame
             ↓
   QUIC: send frame to server
             ↓
   Virtual Hub: lookup dest MAC in MAC table
             ↓
   dest MAC found on Local Bridge port → forward to bridge
             ↓
   Local Bridge: write frame to physical LAN interface (eth0)
             ↓
   physical switch delivers to 192.168.1.10
   ```

### 10.3.2 Site-to-Site

Scenario: HQ LAN ↔ Factory LAN

1. **Configuration:**
   - HQ server:
     ```bash
     quicether server --config hq-server.toml
     # Hub "corp" with local bridge to HQ LAN
     ```
   - Factory server:
     ```bash
     quicether server --config factory-server.toml
     # Hub "corp" with local bridge to Factory LAN
     ```
   - Cascade connection between them:
     ```toml
     # In hq-server.toml
     [[mesh.peers]]
     address = "factory.example.com:4433"
     service_token = "mesh-secret"
     hub = "corp"
     ```

2. **Data Flow (HQ host → Factory host):**
   ```text
   host at HQ (10.0.0.5) sends frame to 10.1.0.50
             ↓
   ARP resolves 10.1.0.50 → MAC aa:bb:cc:dd:ee:ff
             ↓
   Ethernet frame enters HQ server via Local Bridge
             ↓
   Virtual Hub: lookup MAC aa:bb:cc:dd:ee:ff
             ↓
   MAC found on Cascade Tunnel port → forward to Factory peer
             ↓
   QUIC tunnel to Factory server
             ↓
   Factory Virtual Hub: lookup MAC → Local Bridge port
             ↓
   Factory Local Bridge: write frame to physical LAN
             ↓
   physical switch delivers to 10.1.0.50
   ```

---

## 10.4 Local Bridge (Physical LAN Bridging)

In QuicEther's L2 architecture, **Local Bridge** is the mechanism to connect a physical LAN to a Virtual Hub — equivalent to plugging a cable from a physical switch into the virtual switch.

### 10.4.1 Role Configuration

Example:

```toml
# In server.toml
[[hubs]]
name = "office"

[hubs.local_bridge]
interface = "eth0"        # Physical interface to bridge
```

Local Bridge effects:
- All Ethernet frames from the physical interface are forwarded into the Virtual Hub
- All frames from the Virtual Hub destined for MACs on the physical LAN are forwarded out
- Remote clients appear to be "on the same LAN" as physical devices
- DHCP, ARP, and broadcast traffic flow transparently

### 10.4.2 Frame Processing at Local Bridge

Physical LAN → Virtual Hub:
```text
1. Read Ethernet frame from physical interface (raw socket or bridge helper)
2. Learn source MAC → Local Bridge port
3. Forward frame into Virtual Hub for switching
4. Hub forwards to matching session/cascade port (or floods if unknown)
```

Virtual Hub → Physical LAN:
```text
1. Hub determines dest MAC is on Local Bridge port
2. Write frame to physical interface
3. Physical switch delivers to destination
```

### 10.4.3 Policy Enforcement

Local Bridge nodes apply hub policy:
- Each forwarded frame checked against L2/L3/L4 firewall rules
- Anti-spoofing: verify source MAC is registered to the correct port
- MAC limit per port prevents flooding attacks
- If not permitted: drop + log to audit trail

---

## 10.5 MAC Learning & Forwarding

### 10.5.1 Learning Process

When the Virtual Hub receives a frame:
1. Extract source MAC address
2. Record: `source_mac → ingress_port` with timestamp
3. If MAC was already known on a different port → update (MAC migration)
4. If MAC table is full → reject or evict oldest entry

### 10.5.2 Forwarding Decisions

For each frame's destination MAC:

| Destination | Action |
|---|---|
| Known unicast (in MAC table) | Forward to specific port |
| Unknown unicast (not in table) | Flood to all ports except source |
| Broadcast (`ff:ff:ff:ff:ff:ff`) | Flood to all ports except source |
| Multicast | Flood to all ports except source |

This mirrors how a physical Ethernet switch works.

### 10.5.3 MAC Table Aging

- Entries expire after `aging_timeout` (default: 300 seconds)
- Each frame from a MAC resets its timer
- Expired entries are removed → next frame to that MAC triggers flooding
- This handles device mobility and network changes automatically

### 10.5.4 MAC Table Limits & Security

- `max_entries` per hub (default: 8192)
- `max_macs_per_port` per session/bridge (default: 16)
- Exceeding limits: new MACs are rejected (prevents MAC flooding attacks)
- Static MAC entries can be configured for critical devices

---

## 10.6 Optional SecureNAT & Network Services

### 10.6.1 When No Physical Router Exists

For cloud-only or ad-hoc deployments where no physical LAN is bridged, the Virtual Hub can optionally provide network services via **SecureNAT**:

```toml
[[hubs]]
name = "cloud-only"

[hubs.secure_nat]
enabled = true
dhcp_start = "10.100.0.100"
dhcp_end = "10.100.0.200"
gateway = "10.100.0.1"
subnet_mask = "255.255.255.0"
dns = ["8.8.8.8", "8.8.4.4"]
```

SecureNAT provides:
- Built-in DHCP server for clients in the hub
- Virtual gateway for internet access (with NAT)
- ARP responses for the virtual gateway

**Default: disabled.** Physical router is preferred. SecureNAT is a convenience for environments without existing infrastructure.

### 10.6.2 NAT on Edge Servers

For internet access via a QuicEther server with Local Bridge:
- The physical router/gateway handles NAT naturally
- No special configuration in QuicEther — traffic exits via the bridge as normal LAN traffic

For SecureNAT mode:
- QuicEther performs NAT between virtual hub addresses and the server's physical interface

```bash
# Only needed for SecureNAT with internet access
iptables -t nat -A POSTROUTING -s 10.100.0.0/24 -o eth0 -j MASQUERADE
```

---

## 10.7 Configuration Patterns

### 10.7.1 Personal Laptop (Remote Client)

- Joins a Virtual Hub, receives IP from physical router via DHCP through the tunnel

```bash
quicether connect --server home.example.com:4433
# TAP interface created
# DHCP through tunnel → gets IP from home router
# All LAN services accessible transparently
```

### 10.7.2 Home Server (with Local Bridge)

```toml
# server.toml
[server]
listen = "0.0.0.0:4433"

[[hubs]]
name = "home"

[hubs.local_bridge]
interface = "eth0"    # Bridge to home LAN
```

```bash
quicether server --config server.toml
```

### 10.7.3 Cloud Server (SecureNAT, No Physical LAN)

```toml
# server.toml
[server]
listen = "0.0.0.0:4433"

[[hubs]]
name = "team"

[hubs.secure_nat]
enabled = true
dhcp_start = "10.100.0.100"
dhcp_end = "10.100.0.200"
gateway = "10.100.0.1"
subnet_mask = "255.255.255.0"
```

### 10.7.4 Full-Tunnel vs Split-Tunnel

- **Split-Tunnel:** Only access bridged LAN resources (default with Local Bridge)
- **Full-Tunnel:** All traffic through overlay (requires gateway/SecureNAT)

```toml
# Full-tunnel: route all traffic through hub
[client]
server = "vpn.example.com:4433"
full_tunnel = true

# Split-tunnel: only hub traffic (default)
[client]
server = "vpn.example.com:4433"
```

---

## 10.8 Performance & Scaling Considerations

### 10.8.1 Frame Processing Path

Per-frame overhead must be minimal:
- Avoid unnecessary copies between TAP buffer and QUIC
- Batch frames when possible (FrameBatch)

Implementation hints:
- Pre-allocated buffers for TAP reads
- Zero-copy frame forwarding within the Virtual Hub
- Use async I/O with bounded task queues
- MAC table lookups via HashMap for O(1) average case

### 10.8.2 MAC Table Size

For large deployments:
- Each device adds one or more MACs to the table
- Hub-level and port-level limits prevent unbounded growth

Approaches:
- HashMap for O(1) lookup (exact match, not prefix)
- Periodic aging sweep (every 30s)
- Lazy eviction on lookup miss

### 10.8.3 Broadcast Storm Prevention

L2 networks are susceptible to broadcast storms:
- Rate-limit broadcast/multicast frames per port
- Track broadcast frame rates and throttle if exceeded
- Hub-level broadcast budget prevents cascade amplification

### 10.8.4 Control vs Data Separation

MAC table is updated by:
- Source MAC learning from data frames
- Static configuration
- Policy engine (MAC allow/deny lists)

Data plane should avoid heavy locks on the MAC table:
- Use read-optimized structures (e.g., DashMap, lock-free HashMap)
- Separate learning (write) path from forwarding (read) path

---

## Summary

In this chapter we:
- Described how QuicEther uses a TAP interface (`quicether0`) to integrate with OS networking at Layer 2
- Explained that IP addressing comes from physical routers via DHCP through the tunnel — no server-side IP management
- Defined the MAC-based switching model: learn source MACs, forward by destination MAC, flood unknowns
- Walked through end-to-end frame flows for personal VPN and site-to-site via cascade
- Described **Local Bridge** for connecting physical LANs to Virtual Hubs
- Covered MAC table aging, limits, broadcast storm prevention, and optional SecureNAT

This completes the core data plane story: from Ethernet frames on the host, through QuicEther's Virtual Hub, across QUIC, and back into remote networks — all at Layer 2.

**Next Chapter:** We will move up to the **daemon architecture & CLI/API design**, explaining how users and operators interact with QuicEther day-to-day.

---

**Chapter Navigation:**
- [← Previous: Chapter 9 - Security & Zero-Trust](./09-security-and-zero-trust.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 11 - Daemon & CLI Architecture](./11-daemon-and-cli.md)
