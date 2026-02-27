# Chapter 11: Daemon & CLI Architecture

## Introduction

This chapter describes how users and operators **interact** with QuicEther:
- The daemon process (long-running service)
- The CLI tool (user-facing entrypoint)
- Internal APIs between CLI and daemon
- Configuration loading and precedence
- Extensibility for GUIs and orchestration tools

The goal is a **single binary** (`quicether`) that can act as:
- Foreground daemon (for development/simple use)
- Background service (systemd, launchd, Windows service)
- CLI client speaking to a running daemon

---

## 11.1 Design Goals

- **Single binary:** Simplify distribution and packaging
- **Scriptable:** CLI with predictable output formats for automation
- **Safe defaults:** `quicether server` or `quicether connect` should "just work" for most users
- **Clear separation:** Data plane in daemon, control via CLI/API
- **OS-friendly:** Integrate with systemd/launchd/Windows services

---

## 11.2 Process Model

### 11.2.1 Modes of Operation

`quicether` supports several subcommands (proven in httpf):

```bash
# Server mode
quicether server         # Start VPN server (foreground or background)
quicether server stop    # Stop running server

# Client mode
quicether connect        # Connect to a VPN server
quicether bridge         # Connect + bridge local LAN

# Identity & auth management
quicether identity       # Generate or show Ed25519 identity
quicether password       # Manage password authentication

# Server administration
quicether hub            # List/create/modify hubs
quicether session        # List/inspect active sessions
quicether admin          # Admin API operations
quicether audit          # Query audit logs

# Diagnostic
quicether status         # Show connection status
quicether mac-table      # Show MAC forwarding table
quicether multipath      # Show per-path stats
quicether logs           # Stream daemon logs
```

Under the hood:
- **Server mode:** `quicether server` runs long-lived process managing TAP, QUIC, Virtual Hubs, sessions, mesh, etc.
- **Client mode:** `quicether connect` establishes tunnel to server and manages TAP interface
- **CLI mode:** Administrative subcommands are short-lived clients that talk to the daemon over a local IPC channel or admin API.

### 11.2.2 IPC Channel

Options considered:
- UNIX domain sockets (Unix-like OSes)
- Named pipes (Windows)
- Local TCP (localhost-only, as fallback)

Chosen approach:
- Primary: UNIX domain socket (e.g., `/var/run/quicether.sock`)
- Windows: Named pipe (e.g., `\\.\pipe\quicether`)
- Optional: Local TCP for constrained environments

IPC protocol:
- Simple JSON-RPC-style request/response:

```json
{
  "jsonrpc": "2.0",
  "method": "GetStatus",
  "params": {},
  "id": 1
}
```

Response:

```json
{
  "jsonrpc": "2.0",
  "result": {
    "node_id": "node_abc123",
    "uptime_sec": 3600,
    "peers": 5,
    "paths": 2
  },
  "id": 1
}
```

---

## 11.3 Daemon Internals (High-Level)

### 11.3.1 Core Components

Within the daemon process:

```text
┌──────────────────────────────┐
│          Daemon              │
│                              │
│  ┌───────────────┐           │
│  │ Config Loader │           │
│  └──────┬────────┘           │
│         │                    │
│  ┌──────▼──────────┐         │
│  │ Runtime State   │         │
│  │ (Auth, Hubs,     │         │
│  │  QUIC, TAP, etc) │         │
│  └──────┬──────────┘         │
│         │                    │
│  ┌──────▼───────────┐        │
│  │ IPC / Admin API  │        │
│  │ (CLI/GUI/REST)   │        │
│  └──────────────────┘        │
└──────────────────────────────┘
```

Key responsibilities:
- Initialize networking (TAP, QUIC listener)
- Manage hubs, sessions, and authentication
- Enforce firewall and policy rules
- Switch Ethernet frames between clients and mesh peers
- Expose admin API for CLI and GUIs
- Write audit logs

### 11.3.2 Runtime State Struct

Conceptual Rust structure:

```rust
struct DaemonState {
    config: Config,
    identity: Identity,           // Ed25519 keypair + BLAKE3 NodeId
    auth: AuthService,            // Three-tier auth (Ed25519/password/service token)
    transport: QuicTransport,     // QUIC listener + connections
    tap: TapInterface,
    hubs: HubManager,             // Virtual Hubs (Ethernet segments)
    sessions: SessionManager,     // Active client sessions
    switch: FrameSwitch,          // Virtual Hub MAC-based switching
    firewall: FirewallEngine,     // Proxmox-style ACL rules
    policy: PolicyEngine,         // Per-identity L2/L3/L4 rules
    multipath: PathManager,
    mesh: MeshManager,            // Server mesh peer connections
    audit: AuditLogger,           // JSONL + syslog
    metrics: Metrics,             // Prometheus metrics
    health: HealthStatus,
}
```

All IPC requests operate by reading/modifying this state through safe interfaces.

---

## 11.4 Configuration Model

### 11.4.1 Sources & Precedence

Configuration can come from:
- Built-in defaults
- Config file (TOML/YAML), e.g., `/etc/quicether/config.toml` or `~/.config/quicether/config.toml`
- Environment variables
- Command-line flags

Precedence (lowest to highest):
1. Built-in defaults
2. Config file
3. Environment variables
4. CLI arguments

Example:

```bash
# config.toml
[network]
tap_name = "quicether0"

# Override at runtime
QUICETHER_TAP_NAME=qe0 quicether server --tap-name qe1
# Effective tap_name: qe1 (flag wins)
```

### 11.4.2 Sample Config Files

**Server configuration:**

```toml
[server]
listen = "0.0.0.0:4433"

[auth]
method = "password"  # or "identity" or "service_token"

[[hubs]]
name = "default"
dns = ["1.1.1.1", "8.8.8.8"]

[hubs.secure_nat]
enabled = false  # Physical router handles DHCP

[firewall]
default_action = "deny"

[[firewall.rules]]
action = "allow"
src = "*"
dst = "*"

[mesh]
enabled = false

[monitoring]
metrics_addr = "127.0.0.1:9090"
log_level = "info"

[audit]
file = "/var/log/quicether/audit.jsonl"
syslog = false
```

**Client configuration:**

```toml
[server]
host = "vpn.example.com"
port = 4433

[auth]
method = "password"
username = "sarah"

[performance]
profile = "balanced"

[network.multipath]
interfaces = ["eth0", "wlan0"]
mode = "split"
```

---

## 11.5 CLI UX Design

### 11.5.1 Principles

- **Discoverable:** `--help` is clear and concise
- **Scriptable:** Optional JSON output for machine parsing
- **Safe:** Destructive actions require confirmation or `--force`
- **Consistent:** Same verbs and nouns across commands

### 11.5.2 Example Commands

#### 11.5.2.1 Server & Client

```bash
# Start server with config (foreground)
quicether server --config /etc/quicether/server.toml

# Start server as background daemon (Unix)
quicether server --config server.toml --daemon

# Connect to server as client
quicether connect --server vpn.example.com:4433

# Connect as bridge (bridging local LAN to hub)
quicether bridge --server vpn.example.com:4433 --bridge-interface eth0

# Stop running server
quicether server stop
```

Internally:
- `server --daemon` forks and detaches, leaving daemon in background
- `server stop` sends an IPC request `Shutdown {}` to daemon

#### 11.5.2.2 Status

```bash
quicether status
# Human-readable output:
# Identity: sarah (Ed25519)
# Server: vpn.example.com:4433
# Hub: office
# MAC: a1:b2:c3:d4:e5:f6
# IP (DHCP): 192.168.1.105
# Uptime: 1h 23m
# Paths: 2 (eth0, wlan0)
# Bytes: 1.2 GB sent, 3.4 GB recv

# JSON output for scripts
quicether status --json
```

JSON example:

```json
{
  "identity": "sarah",
  "server": "vpn.example.com:4433",
  "hub": "office",
  "mac": "a1:b2:c3:d4:e5:f6",
  "ip_dhcp": "192.168.1.105",
  "uptime_sec": 4980,
  "paths": [
    {"if": "eth0", "rtt_ms": 11.2, "state": "active"},
    {"if": "wlan0", "rtt_ms": 35.7, "state": "active"}
  ],
  "bytes_sent": 1288490188,
  "bytes_recv": 3650722201
}
```

#### 11.5.2.3 Sessions (Server-Side)

```bash
quicether session list
# Identity    | Hub     | MAC               | Connected  | Bytes Sent | Bytes Recv
# sarah       | office  | a1:b2:c3:d4:e5:f6 | 2h 15m     | 1.2 GB     | 3.4 GB
# james       | office  | d7:e8:f9:0a:1b:2c | 45m        | 234 MB     | 567 MB

# Disconnect a session
quicether session disconnect sarah

# JSON
quicether session list --json
```

#### 11.5.2.4 Hubs (Server-Side)

```bash
quicether hub list
# Name    | Clients | MACs | Bridge
# office  | 5       | 12   | eth0
# dev     | 2       | 4    | none

# Create a new hub
quicether hub create --name staging
```

#### 11.5.2.5 Identity Management

```bash
# Generate new Ed25519 identity
quicether identity generate
# Identity: a1b2c3d4... (BLAKE3 hash of public key)
# Private key saved to: ~/.config/quicether/identity.key

# Show current identity
quicether identity show
```

#### 11.5.2.6 Audit Log Query

```bash
# Query recent audit events
quicether audit --last 100

# Filter by identity
quicether audit --identity sarah --last 50

# Filter by event type
quicether audit --event firewall_deny --last 20
```

# Validate config file
quicether config validate /etc/quicether/config.toml
```

---

## 11.6 Service Integration

### 11.6.1 systemd (Linux)

Example unit file:

```ini
[Unit]
Description=QuicEther VPN Daemon
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/quicether server --config /etc/quicether/server.toml
Restart=on-failure
User=quicether
Group=quicether
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW

[Install]
WantedBy=multi-user.target
```

Notes:
- Run foreground (`--daemon=false`) and let systemd handle process lifecycle
- Use `CAP_NET_ADMIN` for TAP creation and bridging

### 11.6.2 launchd (macOS)

A `.plist` file (simplified):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.quicether.daemon</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/quicether</string>
    <string>server</string>
    <string>--config</string>
    <string>/etc/quicether/server.toml</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
```

### 11.6.3 Windows Service

- Use standard Windows service framework
- `quicether.exe server --service` registers as a service

---

## 11.7 Extensibility: GUIs & Orchestration

Because the daemon exposes a stable local API, additional tools can be built:

### 11.7.1 Desktop GUI

- Cross-platform application using the same IPC as CLI
- Can display:
  - Connection status
  - Peers and MAC table
  - Per-path usage graphs
- Can modify:
  - Basic settings (on/off, interface selection)

### 11.7.2 Web UI

- Optional HTTP server bound to localhost
- Uses REST or WebSocket API to talk to daemon
- Suitable for:
  - Headless servers
  - Small business deployments

### 11.7.3 Orchestration / DevOps

- Terraform or Ansible modules that:
  - Deploy `quicether` binaries
  - Push config files
  - Use CLI/IPC to apply changes and verify status

---

## 11.8 Error Handling & Diagnostics

### 11.8.1 CLI Exit Codes

Consistent exit codes for automation:

- `0` → success
- `1` → generic error
- `2` → invalid arguments
- `3` → connection to daemon failed
- `4` → permission error (e.g., missing CAP_NET_ADMIN)

### 11.8.2 Diagnostics Command

```bash
quicether diag
# Collects:
# - Daemon status
# - Recent logs
# - Metrics snapshot
# - Config summary (redacted secrets)
# Outputs tarball for support/troubleshooting
```

### 11.8.3 Logging Levels

- `error`, `warn`, `info`, `debug`, `trace`

CLI examples:

```bash
quicether server --config server.toml --log-level debug
quicether connect --server vpn.example.com:4433 --log-level debug
quicether logs --level warn
```

---

## 11.9 Developer View: Crate Layout

A possible Rust workspace organization:

```text
quicether/
  Cargo.toml
  src/main.rs          # CLI entrypoint & subcommands
  src/commands.rs      # Command dispatch (server, connect, bridge, etc.)
  src/config.rs        # Config structs, loading, merging

  src/server/
    mod.rs             # Server mode (listener, hub, session, mesh)

  src/client/
    mod.rs             # Client mode (connect, bridge, transport)

  src/auth/
    mod.rs             # Three-tier auth (identity, password, service_token)

  src/network/
    mod.rs             # Firewall, policy, frame switching

  src/protocol/
    mod.rs             # QUIC framing, FrameBatch, handshake

  crates/ffi/
    src/lib.rs         # Mobile FFI (iOS/Android)
```

This separation keeps concerns clean while still producing a single binary.

---

## Summary

This chapter defined how QuicEther is **operated** (matching httpf's proven model):

- A single `quicether` binary acts as server, client, bridge, and admin CLI
- The server manages hubs, sessions, authentication, firewall, policy, and mesh
- The client connects to a server and manages local TAP interface
- CLI provides `session`, `hub`, `identity`, `password`, `audit`, and `admin` subcommands
- Configuration follows a clear precedence: defaults → file → env → flags
- Integration guidance is provided for systemd, launchd, and Windows services

With this, we have a coherent operations model that works for individuals, small businesses, and enterprises alike.

**Next Chapter:** We will outline the **implementation roadmap & milestones**, mapping the architecture to concrete development phases.

---

**Chapter Navigation:**
- [← Previous: Chapter 10 - Virtual Hub, TAP & Bridging](./10-virtual-hub-tap-and-bridging.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 12 - Implementation Roadmap](./12-implementation-roadmap.md)
