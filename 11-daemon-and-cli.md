# Chapter 11: Daemon & CLI Architecture

## Introduction

This chapter describes how users and operators **interact** with QuicEther:
- The daemon process (long‑running service)
- The CLI tool (user‑facing entrypoint)
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
- **Safe defaults:** `quicether start` should "just work" for most users
- **Clear separation:** Data plane in daemon, control via CLI/API
- **OS‑friendly:** Integrate with systemd/launchd/Windows services

---

## 11.2 Process Model

### 11.2.1 Modes of Operation

`quicether` supports several subcommands:

```bash
quicether start       # Start daemon (foreground or background)
quicether stop        # Stop running daemon
quicether status      # Show daemon status
quicether config      # Inspect or modify configuration
quicether peers       # List discovered peers
quicether routes      # Show overlay routes
quicether multipath   # Show per‑path stats
quicether logs        # Stream daemon logs
```

Under the hood:
- **Daemon mode:** `quicether start` runs long‑lived process managing TUN, QUIC, DHT, etc.
- **CLI mode:** All other subcommands are short‑lived clients that talk to the daemon over a local IPC channel.

### 11.2.2 IPC Channel

Options considered:
- UNIX domain sockets (Unix-like OSes)
- Named pipes (Windows)
- Local TCP (localhost‑only, as fallback)

Chosen approach:
- Primary: UNIX domain socket (e.g., `/var/run/quicether.sock`)
- Windows: Named pipe (e.g., `\\.\pipe\quicether`)
- Optional: Local TCP for constrained environments

IPC protocol:
- Simple JSON‑RPC‑style request/response:

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

## 11.3 Daemon Internals (High‑Level)

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
│  │ (Node, DHT,     │         │
│  │  QUIC, TUN, etc)│         │
│  └──────┬──────────┘         │
│         │                    │
│  ┌──────▼───────────┐        │
│  │ IPC Server       │        │
│  │ (CLI/API)        │        │
│  └──────────────────┘        │
└──────────────────────────────┘
```

Key responsibilities:
- Initialize networking (TUN, QUIC)
- Join DHT and maintain routing tables
- Enforce policies and forward packets
- Expose control API for CLI and GUIs

### 11.3.2 Runtime State Struct

Conceptual Rust structure:

```rust
struct DaemonState {
    config: Config,
    node_id: NodeId,
    dht: DhtNode,
    transport: QuicTransport,
    tun: TunInterface,
    router: PacketRouter,
    multipath: PathManager,
    policy: PolicyEngine,
    metrics: Metrics,
    health: HealthStatus,
}
```

All IPC requests operate by reading/modifying this state through safe interfaces.

---

## 11.4 Configuration Model

### 11.4.1 Sources & Precedence

Configuration can come from:
- Built‑in defaults
- Config file (TOML/YAML), e.g., `/etc/quicether/config.toml` or `~/.config/quicether/config.toml`
- Environment variables
- Command‑line flags

Precedence (lowest to highest):
1. Built‑in defaults
2. Config file
3. Environment variables
4. CLI arguments

Example:

```bash
# config.toml
[network]
tun_name = "quicether0"

# Override at runtime
QUICETHER_TUN_NAME=qe0 quicether start --tun-name qe1
# Effective tun_name: qe1 (flag wins)
```

### 11.4.2 Sample Config File

```toml
[node]
name = "hq-gateway"
role = "gateway"  # or "standard"

[network]
tun_name = "quicether0"
tun_addr = "10.0.0.1/16"
advertise_subnets = ["10.0.0.0/16"]

[discovery]
bootstrap_mode = true
bootstrap_nodes = [
  "bootstrap.quicether.org:9000"
]

[multipath]
enabled = true
interfaces = ["eth0", "eth1"]
mode = "aggregate"   # or "failover"

[security]
policy_file = "/etc/quicether/policy.toml"
audit_log = "/var/log/quicether/audit.json"

[monitoring]
metrics_addr = "127.0.0.1:9090"
health_addr = "127.0.0.1:9091"
log_level = "info"
```

---

## 11.5 CLI UX Design

### 11.5.1 Principles

- **Discoverable:** `--help` is clear and concise
- **Scriptable:** Optional JSON output for machine parsing
- **Safe:** Destructive actions require confirmation or `--force`
- **Consistent:** Same verbs and nouns across commands

### 11.5.2 Example Commands

#### 11.5.2.1 Start & Stop

```bash
# Start with defaults (foreground)
quicether start

# Start as background daemon (Unix)
quicether start --daemon

# Stop running daemon
quicether stop
```

Internally:
- `start --daemon` forks and detaches, leaving daemon in background
- `stop` sends an IPC request `Shutdown {}` to daemon

#### 11.5.2.2 Status

```bash
quicether status
# Human‑readable output:
# Node ID: node_abc123
# Uptime: 1h 23m
# Peers: 7
# Paths: 2 (eth0, wlan0)
# TUN: quicether0 (100.64.0.10/16)

# JSON output for scripts
quicether status --json
```

JSON example:

```json
{
  "node_id": "node_abc123",
  "uptime_sec": 4980,
  "peers": 7,
  "paths": [
    {"if": "eth0", "rtt_ms": 11.2, "state": "active"},
    {"if": "wlan0", "rtt_ms": 35.7, "state": "active"}
  ],
  "tun": {
    "name": "quicether0",
    "addr": "100.64.0.10/16"
  }
}
```

#### 11.5.2.3 Peers

```bash
quicether peers
# NodeID          | Address           | RTT   | Status
# node_home       | 203.0.113.5:9000  | 15ms  | Connected
# node_hq_gateway | 198.51.100.10:9000| 45ms  | Connected (gateway)

# JSON
quicether peers --json
```

#### 11.5.2.4 Routes

```bash
quicether routes
# Destination      | Next Hop        | Type     |
# 192.168.1.0/24   | node_home       | Node     |
# 10.0.0.0/16      | node_hq_gateway | Gateway  |
# 0.0.0.0/0        | node_dc_gateway | Default  |
```

#### 11.5.2.5 Config Management

```bash
# Show effective config (after merging file/env/flags)
quicether config show

# Edit config with $EDITOR
quicether config edit

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
ExecStart=/usr/bin/quicether start --daemon=false
Restart=on-failure
User=quicether
Group=quicether
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW

[Install]
WantedBy=multi-user.target
```

Notes:
- Run foreground (`--daemon=false`) and let systemd handle process lifecycle
- Use `CAP_NET_ADMIN` for TUN creation and routing

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
    <string>start</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
```

### 11.6.3 Windows Service

- Use standard Windows service framework
- `quicether.exe start --service` registers as a service

---

## 11.7 Extensibility: GUIs & Orchestration

Because the daemon exposes a stable local API, additional tools can be built:

### 11.7.1 Desktop GUI

- Cross‑platform application using the same IPC as CLI
- Can display:
  - Connection status
  - Peers and routes
  - Per‑path usage graphs
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
quicether start --log-level debug
quicether logs --level warn
```

---

## 11.9 Developer View: Crate Layout

A possible Rust workspace organization:

```text
quicether/
  Cargo.toml
  src/main.rs          # CLI entrypoint & subcommands

  daemon/
    src/lib.rs         # DaemonState, run_loop

  cli/
    src/lib.rs         # CLI parsing, IPC client

  ipc/
    src/lib.rs         # JSON-RPC types, socket handling

  config/
    src/lib.rs         # Config structs, loading, merging
```

This separation keeps concerns clean while still producing a single binary.

---

## Summary

This chapter defined how QuicEther is **operated**:

- A single `quicether` binary acts as both daemon and CLI
- The daemon manages all networking, discovery, routing, and policy
- The CLI and potential GUIs communicate via a stable local IPC API
- Configuration follows a clear precedence: defaults → file → env → flags
- Integration guidance is provided for systemd, launchd, and Windows services

With this, we have a coherent operations model that works for individuals, small businesses, and enterprises alike.

**Next Chapter:** We will outline the **implementation roadmap & milestones**, mapping the architecture to concrete development phases.

---

**Chapter Navigation:**
- [← Previous: Chapter 10 - VPN Interface & Routing](./10-vpn-interface-and-routing.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 12 - Implementation Roadmap](./12-implementation-roadmap.md)
