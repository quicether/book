# Chapter 15: Configuration Reference

## Introduction

This chapter is a **reference** for QuicEther configuration.

It documents:
- Server config (`server.toml`) and client config (`client.toml`)
- All major sections and keys
- Types, defaults, and examples

> **httpf validation:** This configuration model is directly ported from HTTP Fabric's
> production-tested config system. The two-file split (server vs client), hub-based
> multi-tenancy, three-tier auth, firewall rules, and audit logging are all proven
> patterns. QuicEther replaces HTTP/WebSocket transport with native QUIC but keeps
> the same config structure.

This is not a tutorial; see previous chapters for narrative examples. This is the place you consult when you wonder: *"What does `performance.profile` do?"*

---

## 15.1 Config Files & Locations

### 15.1.1 Two-File Model

QuicEther uses separate config files for server and client roles:

| Role | Default File | Purpose |
|------|-------------|---------|
| Server | `server.toml` | Listener, hubs, auth, mesh, firewall, audit |
| Client | `client.toml` | Connection, auth, routing, performance |
| Bridge | `client.toml` | Same as client + `[bridge]` section |

### 15.1.2 Default Search Paths

**Server** looks for `server.toml` in:

1. CLI flag: `quicether server --config /path/to/server.toml`
2. Environment: `QUICETHER_CONFIG=/path/to/server.toml`
3. OS defaults:
   - Linux: `/etc/quicether/server.toml`
   - macOS: `/usr/local/etc/quicether/server.toml`
   - Windows: `%ProgramData%\QuicEther\server.toml`

**Client** looks for `client.toml` in:

1. CLI flag: `quicether connect --config /path/to/client.toml`
2. Environment: `QUICETHER_CONFIG=/path/to/client.toml`
3. OS defaults:
   - Linux: `~/.config/quicether/client.toml`, then `/etc/quicether/client.toml`
   - macOS: `~/Library/Application Support/QuicEther/client.toml`
   - Windows: `%APPDATA%\QuicEther\client.toml`

### 15.1.3 Format

- TOML is the primary format for configuration files.
- All examples below use TOML.

---

## 15.2 Server Config: Top-Level Structure

```toml
[server]
# Listener and admin settings

[[hubs]]
# Hub (network) definitions — one or more

[auth]
# Server-side authentication backends

[mesh]
# Multi-server mesh clustering

[logging]
# Log level and format

[audit]
# Audit log settings and syslog forwarding
```

---

## 15.3 Client Config: Top-Level Structure

```toml
[server]
# Connection target (host, port, hub)

[auth]
# Client authentication method

[network]
# TAP device, MTU, encryption, multipath

[performance]
# Batching profile and tuning

[routing]
# Full tunnel toggle

[reconnect]
# Auto-reconnect behavior

[bridge]
# Local Bridge mode (optional — bridge physical LAN to hub)

[proxy]
# Client-side proxy (optional — tunnel through SOCKS5/HTTP)

[logging]
# Log level and format
```

---

## 15.4 Server: `[server]` Section

Controls the QUIC listener and admin API.

```toml
[server]
listen_address = "0.0.0.0"          # Bind address (string, default "0.0.0.0")
port = 4433                          # QUIC port (integer, default 4433)
public_url = "quic://vpn.example.com:4433"  # External URL (string, optional)

# TLS (required for QUIC — uses TLS 1.3 built into QUIC)
[server.tls]
certificate_path = "/etc/quicether/server.crt"
private_key_path = "/etc/quicether/server.key"

# Admin API (optional — separate port for management)
admin_api_key = "my-secret-key"      # API key for admin endpoints (string, optional)
admin_port = 9090                    # Separate admin port (integer, optional)
admin_listen = "127.0.0.1"          # Admin bind address (default "127.0.0.1")
```

- `listen_address` (string):
  - Default: `"0.0.0.0"`.
  - Bind address for the QUIC listener.

- `port` (integer):
  - Default: `4433`.
  - UDP port for QUIC connections.

- `public_url` (string, optional):
  - External URL where this server is reachable.
  - Used by mesh peers and for self-identification.
  - If omitted, derived from `listen_address:port`.

- `[server.tls]`:
  - QUIC mandates TLS 1.3 — this is not optional.
  - `certificate_path`: Path to TLS certificate (PEM).
  - `private_key_path`: Path to TLS private key (PEM).

- `admin_api_key` (string, optional):
  - If set, all `/api/v1/*` and `/admin/*` endpoints require this key.

- `admin_port` (integer, optional):
  - If set, admin endpoints are served on this port instead of the QUIC port.

- `admin_listen` (string):
  - Default: `"127.0.0.1"` (localhost only for security).

---

## 15.5 Server: `[[hubs]]` Section

Defines virtual networks (Ethernet segments). A server can host multiple hubs for multi-tenancy.

```toml
[[hubs]]
name = "default"                     # Hub name (string, required)
session_timeout_secs = 60           # Idle session timeout (default: 60)
max_sessions = 253                   # Max concurrent sessions (default: 253)
max_macs_per_port = 16               # MAC limit per session (default: 16)
mac_aging_secs = 300                 # MAC table entry timeout (default: 300)

# Local Bridge (optional — connect physical LAN to this hub)
[hubs.local_bridge]
interface = "eth0"                   # Physical interface to bridge

# SecureNAT (optional — built-in DHCP/NAT when no physical router)
[hubs.secure_nat]
enabled = false                      # Default: false (physical router preferred)
dhcp_start = "10.100.0.100"          # DHCP range start
dhcp_end = "10.100.0.200"            # DHCP range end
gateway = "10.100.0.1"               # Virtual gateway IP
subnet_mask = "255.255.255.0"        # Subnet mask
dns = ["1.1.1.1", "8.8.8.8"]         # DNS servers

# MSS clamping
mss_clamp = 1360                     # Explicit MSS value (optional)
mss_clamp_auto = true                # Auto-calc from MTU (default: false)

# Access control (empty = allow all)
[hubs.access_control]
allowed_identities = ["hf_abc123"]
allowed_groups = ["engineering"]
```

- `name` (string, required):
  - Unique hub identifier. Clients specify this in `[server].hub`.
  - Each hub is an independent broadcast domain (like a VLAN).

- `session_timeout_secs` (integer):
  - Default: `60`. Sessions idle longer are disconnected.

- `max_sessions` (integer):
  - Default: `253`. Maximum concurrent client sessions in this hub.

- `max_macs_per_port` (integer):
  - Default: `16`. Maximum MAC addresses learned per session/bridge.
  - Prevents MAC flooding attacks.

- `mac_aging_secs` (integer):
  - Default: `300`. MAC table entries expire after this duration.

- `[hubs.local_bridge]` (optional):
  - `interface`: Physical network interface to bridge into this hub.
  - All Ethernet frames from the physical interface flow into the Virtual Hub.
  - DHCP, ARP, broadcast traffic pass through transparently.

- `[hubs.secure_nat]` (optional):
  - Default: disabled. Only enable when no physical router is available.
  - Provides built-in DHCP server and virtual gateway within the hub.
  - `dhcp_start` / `dhcp_end`: IP range for DHCP leases.
  - `gateway`: Virtual gateway IP for internet access.
  - `dns`: DNS servers to assign via DHCP.

### 15.5.1 Hub Firewall Rules

Proxmox-style firewall rules evaluated in order (first match wins). Rules can match at L2 (MAC), L3 (IP), and L4 (port):

```toml
[[hubs.firewall]]
enable = true
direction = "in"                     # "in" | "out"
action = "ACCEPT"                    # "ACCEPT" | "DROP" | "REJECT"
protocol = "tcp"                     # "tcp" | "udp" | "icmp" | "all"
dest_port = 22
name = "allow-ssh"

# MAC-based rule (L2 filtering):
[[hubs.firewall]]
enable = true
direction = "in"
action = "DROP"
src_mac = "aa:bb:cc:*"               # Wildcard MAC match
name = "block-rogue-mac-range"

# IP-based rule (L3 filtering, applied to frame payload):
[[hubs.firewall]]
enable = true
direction = "out"
action = "ACCEPT"
protocol = "all"
src_ip = "192.168.1.0/24"
name = "allow-lan-outbound"
```

Rule fields:
- `enable` (bool): Enable/disable without deletion (default: true)
- `direction` (string): `"in"` or `"out"`
- `action` (string): `"ACCEPT"`, `"DROP"`, `"REJECT"`
- `protocol` (string): `"tcp"`, `"udp"`, `"icmp"`, `"arp"`, `"all"`
- `src_mac` / `dest_mac` (string, optional): MAC address match (supports `*` wildcards)
- `src_ip` / `dest_ip` (string, optional): CIDR match (parsed from frame payload)
- `src_port` / `dest_port` (integer or range, optional): Port match
- `name` (string, optional): Human-readable name for logs

### 15.5.2 Hub Policies

Per-identity L2/L3/L4 ACLs:

```toml
[[hubs.policies]]
name = "restrict-guests"
src_identity = "guest-*"
dst_port = 22
action = "deny"
```

---

## 15.6 Server: `[auth]` Section

Three-tier authentication backends.

### 15.6.1 Identity Auth (Ed25519)

Identity authentication is always available (no config needed). Clients present
Ed25519 public keys; servers track them by BLAKE3-derived identity IDs.

### 15.6.2 Local Password Auth

```toml
[auth.local]
enabled = true
hash_algorithm = "argon2id"          # "argon2id" | "bcrypt" | "sha256"
max_failed_attempts = 5              # Lockout after N failures (0 = no lockout)
lockout_duration_secs = 900          # 15 minutes lockout
user_file = "/etc/quicether/users.toml"  # External user database (optional)

# Or inline users:
[[auth.local.users]]
username = "alice"
password_hash = "$argon2id$v=19$m=19456,t=2,p=1$..."
allowed_hubs = ["default"]
display_name = "Alice"
email = "alice@example.com"
disabled = false
```

- `enabled` (bool): Enable local password auth (default: false)
- `hash_algorithm` (string): Default `"argon2id"` (recommended)
- `max_failed_attempts` (integer): Lockout threshold (default: 5, 0 = disabled)
- `lockout_duration_secs` (integer): Lockout duration (default: 900)
- `user_file` (string, optional): Path to external user database
- `users` (array, optional): Inline user definitions

User fields:
- `username` (string, required)
- `password_hash` (string, required): Pre-hashed password
- `allowed_hubs` (array of strings): Restrict to specific hubs (empty = all)
- `disabled` (bool): Disable without deletion
- `display_name` / `email` (optional metadata)

### 15.6.3 Service Token Auth

```toml
[auth.service_token]
enabled = true

[[auth.service_token.tokens]]
name = "site-b-gateway"
token_hash = "sha256:a1b2c3..."      # echo -n "mytoken" | sha256sum
allowed_hubs = ["datacenter"]
expires = "2027-01-01T00:00:00Z"     # ISO 8601 (optional, empty = no expiry)
max_sessions = 1                     # Concurrent session limit (optional)
disabled = false
identity_id = "site-b"              # Override identity (optional)
```

- `enabled` (bool): Enable service token auth (default: false)
- `tokens` (array): Defined service tokens

Token fields:
- `name` (string, required): Human-readable name for audit logs
- `token_hash` (string, required): SHA-256 hash of the raw token (hex-encoded)
- `allowed_hubs` (array of strings): Restrict to specific hubs
- `expires` (string, optional): ISO 8601 expiration timestamp
- `max_sessions` (integer, optional): Max concurrent sessions
- `disabled` (bool): Revoke without deletion
- `identity_id` (string, optional): Override auto-derived identity

---

## 15.7 Server: `[mesh]` Section

Multi-server mesh clustering for geographic distribution.

```toml
[mesh]
enabled = true

[[mesh.peers]]
url = "quic://node-2.example.com:4433"
hub = "default"
token = "mesh-secret-token"

[[mesh.peers]]
url = "quic://node-3.example.com:4433"
hub = "default"
token = "mesh-secret-token"
```

- `enabled` (bool): Enable mesh mode (default: false)
- `peers` (array): List of mesh peer definitions

Peer fields:
- `url` (string, required): QUIC URL of the peer server
- `hub` (string, required): Hub name to cascade (connect hubs across servers)
- `token` (string, required): Shared authentication token

> The `node_id` is derived automatically from `[server].public_url` hostname
> or the machine hostname if not set.

---

## 15.8 Server: `[audit]` Section

Audit logging for security events.

```toml
[audit]
enabled = true                       # Enable audit logging (default: true)
max_entries = 10000                  # In-memory ring buffer size
file = "/var/log/quicether/audit.log"  # Persist to file (optional)
format = "json"                      # "json" (JSONL) | "text" (human-readable)

# Forward to syslog (optional):
[audit.syslog]
address = "127.0.0.1:514"           # Syslog server (UDP or unix:///dev/log)
facility = "local0"
app_name = "quicether"
rfc5424 = true                       # RFC 5424 format (default: true)
```

- `enabled` (bool): Default `true`.
- `max_entries` (integer): In-memory buffer size (default: 10000).
- `file` (string, optional): Path for JSONL audit log file.
- `format` (string): `"json"` (default) or `"text"`.
- `[audit.syslog]` (optional): Forward to syslog server.

---

## 15.9 Server: `[logging]` Section

```toml
[logging]
level = "info"                       # "trace" | "debug" | "info" | "warn" | "error"
format = "pretty"                    # "pretty" (human-readable) | "json" (structured)
```

- `level` (string): Minimum log level (default: `"info"`)
- `format` (string): Output format (default: `"pretty"`)

---

## 15.10 Client: `[server]` Section

Specifies the server to connect to.

```toml
[server]
host = "vpn.example.com"            # Server hostname or IP (string, required)
port = 4433                          # QUIC port (integer, default 4433)
hub = "default"                      # Hub to join (string, default "default")
```

- `host` (string, required): Server hostname or IP address.
- `port` (integer): Default `4433`.
- `hub` (string): Hub name on the server (default: `"default"`).

> **QUIC difference from httpf:** No `tls` boolean needed — QUIC mandates TLS 1.3.
> Certificate verification is always on by default.

---

## 15.11 Client: `[auth]` Section

Client authentication method. Exactly one method is configured.

### 15.11.1 Identity Auth (Default)

```toml
[auth]
method = "identity"
private_key_path = "identity.key"    # Ed25519 private key file
```

Generate a keypair:
```bash
quicether identity generate -o identity.key
```

### 15.11.2 Password Auth

```toml
[auth]
method = "password"
username = "alice"
password = "secret"
```

> For non-interactive use, set `QUICETHER_PASSWORD` environment variable
> instead of putting passwords in config files.

---

## 15.12 Client: `[network]` Section

TAP device, encryption, and multipath settings.

```toml
[network]
device_mode = "auto"                  # Device mode: "auto" | "tap" | "virtual_tap"
mtu = 1400                           # TAP device MTU (576-9000, default 1400)
encryption = true                    # Data-plane ChaCha20-Poly1305 (default true)
max_connections = 1                  # Parallel QUIC connections (1-32, default 1)
```

- `device_mode` (string):
  - Default: `"auto"` (auto-detect based on platform).
  - `"auto"`: Use Native TAP where available (Linux, macOS, Windows), Virtual TAP elsewhere (iOS, Android, containers).
  - `"tap"`: Force Native TAP. Fails on platforms without TAP support.
  - `"virtual_tap"`: Force Virtual TAP (TUN + userspace L2↔L3 translation). Works on any platform with TUN support. Useful for testing or when TAP drivers are unavailable.
  - See Chapter 10, Section 10.1.3 for the full Virtual TAP design.

- `mtu` (integer):
  - Default: `1400`.
  - QUIC MTU calculation: physical(1500) - UDP/IP(28) - QUIC overhead(~42) - Ethernet header(14) - margin(~16) = 1400.
  - Valid range: 576 (minimum IPv4) to 9000 (jumbo frames).

- `encryption` (bool):
  - Default: `true`.
  - ChaCha20-Poly1305 AEAD with X25519 key exchange, layered on QUIC's TLS 1.3.

- `max_connections` (integer):
  - Default: `1` (single full-duplex connection).
  - `2+`: Parallel QUIC connections for throughput and resilience.
  - Clamped to range 1-32.

### 15.12.1 Multipath

```toml
[network.multipath]
mode = "failover"                    # "disabled" | "failover" | "redundant" | "split"
interfaces = ["en0", "pdp_ip0"]      # Network interfaces (empty = auto-detect)
probe_interval_ms = 1000             # Path probe interval (default: 1000)
path_timeout_ms = 5000               # Mark path unhealthy after (default: 5000)
dedup_window = 256                   # Dedup sequence window (default: 256)
```

Modes:
- `"disabled"` (default): Single path only.
- `"failover"`: Use best path; switch on failure. No data duplication.
- `"redundant"`: Send on all paths for maximum reliability.
- `"split"`: Round-robin distribution for aggregated bandwidth.

> **QUIC advantage:** Native QUIC connection migration (RFC 9000 §9) provides
> seamless failover without application-layer reconnection. Combined with
> multipath QUIC, this replaces httpf's parallel TCP/WebSocket approach.

---

## 15.13 Client: `[performance]` Section

Batching profiles for different workloads.

```toml
[performance]
profile = "balanced"                 # "latency" | "balanced" | "throughput" | "maxperformance"
```

Profile defaults:

| Setting | latency | balanced | throughput | maxperformance |
|---------|---------|----------|------------|----------------|
| `max_batch_size` | 4 | 16 | 128 | 256 |
| `max_batch_bytes` | 4KB | 16KB | 128KB | 512KB |
| `timeout_ms` | 1 | 2 | 10 | 20 |
| `immediate_flush` | icmp,arp,dhcp | icmp,dhcp | (none) | (none) |

Override individual settings:

```toml
[performance]
profile = "balanced"
max_batch_size = 32                  # Override profile default
max_batch_bytes = 32768
timeout_ms = 3
immediate_flush = ["icmp"]
```

- `profile` (string):
  - `"latency"`: VoIP, gaming, interactive — minimize delay.
  - `"balanced"` (default): General use — reasonable latency and throughput.
  - `"throughput"`: Bulk transfers, streaming — maximize bandwidth.
  - `"maxperformance"`: Benchmarks — very large batches, aggressive buffering.

- `max_batch_size` (integer, optional): Override profile packet limit.
- `max_batch_bytes` (integer, optional): Override profile byte limit.
- `timeout_ms` (integer, optional): Override profile batch timeout.
- `immediate_flush` (array of strings, optional): Packet types to flush immediately.

---

## 15.14 Client: `[routing]` Section

Full tunnel configuration. In L2 mode, split-tunneling is based on whether the hub has a Local Bridge to a physical LAN.

```toml
[routing]
route_all_traffic = false            # Full tunnel (default: false)
```

- `route_all_traffic` (bool): Default `false`. When `true`, all traffic goes through the overlay hub.
  - Requires the hub to have a gateway (Local Bridge with router, or SecureNAT).
  - When `false` (default), only traffic to devices on the bridged LAN flows through the tunnel.

---

## 15.15 Client: `[reconnect]` Section

Auto-reconnect behavior on connection loss.

```toml
[reconnect]
enabled = true                       # Auto-reconnect (default: true)
max_attempts = 0                     # 0 = unlimited retries
min_backoff_ms = 1000                # Initial retry delay
max_backoff_ms = 60000               # Maximum retry delay (exponential backoff)
```

- `enabled` (bool): Default `true`.
- `max_attempts` (integer): Default `0` (unlimited).
- `min_backoff_ms` / `max_backoff_ms` (integer): Exponential backoff bounds.

---

## 15.16 Client: `[bridge]` Section (Optional)

Local Bridge mode — bridge a physical LAN interface into the hub.

```toml
[bridge]
interface = "eth0"                   # Physical interface to bridge
```

Start in bridge mode:
```bash
quicether bridge --config client.toml
```

- `interface` (string): Physical network interface to bridge into the hub.
  - All Ethernet frames from this interface flow into the Virtual Hub.
  - Remote clients can access devices on the physical LAN transparently.

---

## 15.17 Client: `[proxy]` Section (Optional)

Tunnel the QUIC connection through an upstream proxy.

```toml
[proxy]
type = "socks5"                      # "socks5" | "http_connect"
address = "127.0.0.1:1080"
username = "user"
password = "pass"
```

- `type` (string): `"socks5"` (RFC 1928) or `"http_connect"` (RFC 7231).
- `address` (string): Proxy server `host:port`.
- `username` / `password` (string, optional): Proxy authentication.

> **Note:** QUIC over SOCKS5 uses UDP ASSOCIATE. HTTP CONNECT proxies tunnel
> QUIC-over-TCP as a fallback.

---

## 15.18 Client: `[logging]` Section

Same format as server logging:

```toml
[logging]
level = "info"                       # "trace" | "debug" | "info" | "warn" | "error"
format = "pretty"                    # "pretty" | "json"
```

---

## 15.19 Environment Variables

Environment variables override config for quick changes or secrets.

**Server:**
- `QUICETHER_CONFIG` — config file path
- `QUICETHER_LOG_LEVEL` — override `[logging].level`
- `QUICETHER_ADMIN_API_KEY` — override `[server].admin_api_key`

**Client:**
- `QUICETHER_CONFIG` — config file path
- `QUICETHER_LOG_LEVEL` — override `[logging].level`
- `QUICETHER_PASSWORD` — password for `method = "password"` auth (avoids plaintext in config)

Mapping convention:
- `QUICETHER_SECTION_KEY` maps to `[section].key`
- Example: `QUICETHER_SERVER_PORT=4433` maps to `[server].port = 4433`

---

## 15.20 CLI Flags

CLI flags override both config and environment variables.

**Server:**
```bash
quicether server \
  --config /etc/quicether/server.toml \
  --log-level debug \
  --port 4433 \
  --admin-port 9090
```

**Client:**
```bash
quicether connect \
  --config /etc/quicether/client.toml \
  --server vpn.example.com \
  --port 4433 \
  --hub default \
  --log-level debug
```

**Bridge:**
```bash
quicether bridge \
  --config /etc/quicether/client.toml \
  --bridge-interface eth0
```

Full flag list: `quicether server --help`, `quicether connect --help`, `quicether bridge --help`.

---

## 15.21 Minimal Config Examples

### 15.21.1 Minimal Server

```toml
[server]
port = 4433

[server.tls]
certificate_path = "/etc/quicether/server.crt"
private_key_path = "/etc/quicether/server.key"

[[hubs]]
name = "default"
```

### 15.21.2 Minimal Client (Identity Auth)

```toml
[server]
host = "vpn.example.com"
port = 4433
hub = "default"

[auth]
method = "identity"
private_key_path = "identity.key"
```

### 15.21.3 Client with Full Tunnel

```toml
[server]
host = "vpn.example.com"
port = 4433

[auth]
method = "identity"
private_key_path = "identity.key"

[routing]
route_all_traffic = true
```

### 15.21.4 Bridge (Site-to-Site)

```toml
[server]
host = "vpn.example.com"
port = 4433

[auth]
method = "identity"
private_key_path = "identity.key"

[bridge]
interface = "eth0"
```

### 15.21.5 Multi-Hub Server with Auth

```toml
[server]
listen_address = "0.0.0.0"
port = 4433
public_url = "quic://vpn.example.com:4433"
admin_api_key = "change-me-in-production"

[server.tls]
certificate_path = "/etc/quicether/server.crt"
private_key_path = "/etc/quicether/server.key"

[[hubs]]
name = "engineering"

[[hubs]]
name = "guests"
session_timeout_secs = 30

[auth.local]
enabled = true
user_file = "/etc/quicether/users.toml"

[auth.service_token]
enabled = true

[[auth.service_token.tokens]]
name = "ci-runner"
token_hash = "sha256:a1b2c3d4..."
allowed_hubs = ["engineering"]

[audit]
enabled = true
file = "/var/log/quicether/audit.log"
format = "json"

[logging]
level = "info"
```

### 15.21.6 Server Mesh (3 Regions)

```toml
# Region 1: US-East
[server]
listen_address = "0.0.0.0"
port = 4433
public_url = "quic://us-east.vpn.example.com:4433"

[server.tls]
certificate_path = "/etc/quicether/server.crt"
private_key_path = "/etc/quicether/server.key"

[[hubs]]
name = "default"

[hubs.local_bridge]
interface = "eth0"

[mesh]
enabled = true

[[mesh.peers]]
url = "quic://eu-west.vpn.example.com:4433"
hub = "default"
token = "mesh-secret"

[[mesh.peers]]
url = "quic://ap-south.vpn.example.com:4433"
hub = "default"
token = "mesh-secret"

[logging]
level = "info"
```

---

## Summary

This chapter provided a **reference** for QuicEther configuration:

**Server config** (`server.toml`):
- `[server]` — QUIC listener, TLS, admin API
- `[[hubs]]` — Virtual Hubs (Ethernet segments) with Local Bridge, SecureNAT, firewall, policies
- `[auth]` — Three-tier: identity (Ed25519), local passwords (Argon2id), service tokens
- `[mesh]` — Multi-server clustering with cascade connections
- `[audit]` — JSONL file + syslog forwarding
- `[logging]` — Level and format

**Client config** (`client.toml`):
- `[server]` — Connection target (host, port, hub)
- `[auth]` — Identity keypair or password
- `[network]` — MTU, encryption, multipath
- `[performance]` — Batching profiles (latency/balanced/throughput/maxperformance)
- `[routing]` — Full tunnel toggle
- `[reconnect]` — Auto-reconnect with exponential backoff
- `[bridge]` — Optional Local Bridge (physical LAN interface)
- `[proxy]` — Optional SOCKS5/HTTP CONNECT upstream proxy
- `[logging]` — Level and format

Config precedence: CLI flags > environment variables > config file > defaults.

**Next Chapter:** We will discuss **future directions & extensions**, including advanced schedulers and post-quantum cryptography.

---

**Chapter Navigation:**
- [← Previous: Chapter 14 - Deployment & Operations](./14-deployment-and-operations.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 16 - Future Directions & Extensions](./16-future-directions-and-extensions.md)
