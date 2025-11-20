# Chapter 15: Configuration Reference

## Introduction

This chapter is a **reference** for QuicEther configuration.

It documents:
- Top‑level config structure
- All major sections and keys
- Types, defaults, and examples

This is not a tutorial; see previous chapters for narrative examples. This is the place you consult when you wonder: *“What does `multipath.mode` do?”*

---

## 15.1 Config Files & Locations

### 15.1.1 Default Search Paths

QuicEther looks for a config file in the following order:

1. Path specified via CLI flag:
   ```bash
   quicether start --config /path/to/config.toml
   ```
2. Environment variable:
   - `QUICETHER_CONFIG=/path/to/config.toml`
3. OS‑specific defaults:
   - Linux: `/etc/quicether/config.toml`, then `~/.config/quicether/config.toml`
   - macOS: `/usr/local/etc/quicether/config.toml`, then `~/Library/Application Support/QuicEther/config.toml`
   - Windows: `%ProgramData%\QuicEther\config.toml`, then `%APPDATA%\QuicEther\config.toml`

### 15.1.2 Format

- TOML is the primary format for configuration files.
- All examples below use TOML.

---

## 15.2 Top‑Level Structure

```toml
[node]
# Identity and role

[network]
# TUN interface, overlay addresses, subnets

[discovery]
# DHT and bootstrap settings

[multipath]
# Multi‑interface and scheduling

[security]
# Policy, keys, audit logging

[monitoring]
# Metrics and health endpoints

[overlay]
# Overlay routing defaults (optional)

[performance]
# Advanced tuning (optional)
```

---

## 15.3 `[node]` Section

Controls node identity and high‑level behavior.

```toml
[node]
name = "hq-gateway"       # Human‑friendly name (string)
role = "standard"          # "standard" | "gateway" | "enterprise"
key_file = "/var/lib/quicether/node.key"  # Path to private key (string, optional)
```

- `name` (string, optional):
  - Used in logs and CLI outputs.
  - Does **not** affect NodeId (which is derived from key).

- `role` (string, optional):
  - `"standard"` (default): Regular node.
  - `"gateway"`: Node may forward for others; enables extra config in `[network]` / `[overlay]`.
  - `"enterprise"`: Same as gateway + enables enterprise defaults (stricter policy, audit on).

- `key_file` (string, optional):
  - Path to Ed25519 private key.
  - If omitted, QuicEther may generate and store one in a default location on first start.

---

## 15.4 `[network]` Section

Defines virtual interface and advertised subnets.

```toml
[network]
tun_name = "quicether0"           # Name of TUN device (string)
tun_addr = "100.64.0.10/16"       # CIDR address on overlay (string)
advertise_subnets = [              # Subnets this node owns (array of CIDR strings)
  "192.168.1.0/24"
]
mtu = 1420                         # MTU for TUN device (integer, optional)
```

- `tun_name` (string):
  - Default: `"quicether0"`.

- `tun_addr` (CIDR string):
  - Required.
  - Example: `"100.64.0.10/16"`.

- `advertise_subnets` (array of CIDR strings, optional):
  - Subnets reachable *behind* this node.
  - Advertised via DHT for other nodes to route to.

- `mtu` (integer, optional):
  - Default: implementation‑dependent (commonly 1420).
  - Lower if underlying network has small MTU.

---

## 15.5 `[discovery]` Section

Controls DHT and bootstrap behavior.

```toml
[discovery]
bootstrap_mode = false           # If true, node acts as bootstrap
bootstrap_nodes = [              # Addresses of known bootstrap nodes
  "bootstrap.quicether.org:9000",
  "198.51.100.10:9000"
]
public_ip = "198.51.100.10"     # Public IP/hostname (for bootstrap/gateway)
dht_port = 9000                  # UDP port for DHT/QUIC (integer)
```

- `bootstrap_mode` (bool):
  - If `true`, node expects inbound joins from others and advertises itself as bootstrap.

- `bootstrap_nodes` (array of strings):
  - Each string is `"host:port"`.
  - Used to join the DHT at startup.

- `public_ip` (string, optional):
  - IP or hostname others should use to reach this node.
  - Particularly relevant for gateways and bootstrap nodes.

- `dht_port` (integer):
  - Default: 9000.
  - UDP port for DHT and QUIC endpoint.

---

## 15.6 `[multipath]` Section

Configures use of multiple interfaces and scheduling.

```toml
[multipath]
enabled = true                    # Enable multipath (bool)
mode = "aggregate"                # "aggregate" | "failover"
interfaces = ["eth0", "eth1"]     # Network interfaces to use
scheduler = "weighted"            # "round_robin" | "weighted" | "latency_aware"

[multipath.weights]
eth0 = 1.0
eth1 = 2.0

[multipath.policies]
interactive_max_rtt_ms = 50       # Prefer low‑RTT paths for small flows
bulk_min_bandwidth_mbps = 5       # Use all paths for bulk transfers
```

- `enabled` (bool):
  - Default: `false` (MVP may start single‑path by default).

- `mode` (string):
  - `"aggregate"`: Use all active paths simultaneously.
  - `"failover"`: Use best path; others as standby.

- `interfaces` (array of strings):
  - Names of OS network interfaces to use.

- `scheduler` (string):
  - `"round_robin"`: Simple distribution.
  - `"weighted"`: Based on `[multipath.weights]`.
  - `"latency_aware"`: Prefer lower RTT where appropriate.

- `[multipath.weights]` (table):
  - Interface → relative weight (float).

- `[multipath.policies]` (table, optional):
  - Advanced hints for scheduler; semantics may evolve.

---

## 15.7 `[security]` Section

Controls policy engine, audit logging, and potentially key/Pki settings.

```toml
[security]
policy_file = "/etc/quicether/policy.toml"   # Path to policy rules
audit_log = "/var/log/quicether/audit.json" # Where to write audit events
require_2fa = false                           # Reserved for future (enterprise)
```

- `policy_file` (string):
  - If omitted, default allow/deny policy may apply (implementation‑defined, likely conservative).

- `audit_log` (string):
  - Path to audit log file (JSON lines).

- `require_2fa` (bool):
  - Placeholder for future support of MFA/2FA flows.

Policy file format is described in Chapter 9; briefly:

```toml
[[rules]]
subject = "node_sarah_laptop"   # NodeId or alias
object = "10.0.0.0/16"           # CIDR
action = "allow"                 # "allow" | "deny"
```

---

## 15.8 `[monitoring]` Section

Controls metrics and health endpoints.

```toml
[monitoring]
metrics_addr = "127.0.0.1:9090"     # Prometheus metrics (http)
health_addr = "127.0.0.1:9091"      # Health endpoint (http)
log_level = "info"                  # "error" | "warn" | "info" | "debug" | "trace"
```

- `metrics_addr` (string):
  - If unset, metrics endpoint may be disabled.

- `health_addr` (string):
  - If unset, health endpoint may be disabled.

- `log_level` (string):
  - Global minimum log level.

---

## 15.9 `[overlay]` Section (Optional)

Provides overlay routing defaults and gateway behavior.

```toml
[overlay]
default_gateway = "node_datacenter_gateway"  # NodeId or alias
```

- `default_gateway` (string, optional):
  - Used when no specific overlay route matches.
  - Common in full‑tunnel setups.

Additional keys may include:
- Static overlay routes
- Per‑subnet preferences

---

## 15.10 `[performance]` Section (Advanced)

Tuning knobs for advanced deployments.

```toml
[performance]
io_backend = "auto"           # "auto" | "io_uring" | "standard"
cpu_affinity = [0, 1, 2, 3]    # Pin I/O threads to specific CPUs
max_connections = 10000        # Safety cap
```

- `io_backend` (string):
  - `"auto"` (default): Choose best available.
  - `"io_uring"`: Use Linux io_uring explicitly.
  - `"standard"`: Use portable async I/O.

- `cpu_affinity` (array of integers, optional):
  - Pin critical threads to specific cores.

- `max_connections` (integer):
  - Hard cap on concurrent connections to avoid OOM.

---

## 15.11 Environment Variables

Environment variables override config for quick changes or secrets.

Examples:

- `QUICETHER_CONFIG=/path/to/config.toml`
- `QUICETHER_LOG_LEVEL=debug`
- `QUICETHER_METRICS_ADDR=0.0.0.0:9090`

Mapping rules (typical):
- `QUICETHER_SECTION_KEY` → `[section].key`
  - e.g., `QUICETHER_NETWORK_TUN_NAME=qe0` → `[network].tun_name = "qe0"`

Exact mapping will be documented in CLI help once implemented.

---

## 15.12 CLI Flags

CLI flags override both config and environment vars.

Selected examples:

```bash
quicether start \
  --config /etc/quicether/hq.toml \
  --log-level debug \
  --tun-name qe0 \
  --metrics-addr 0.0.0.0:9090
```

Conceptual mapping:
- `--config` → config file path
- `--log-level` → `[monitoring].log_level`
- `--tun-name` → `[network].tun_name`
- `--metrics-addr` → `[monitoring].metrics_addr`

Full flag list will be kept in `quicether --help` output.

---

## 15.13 Minimal Config Examples

### 15.13.1 Minimal Personal Node

```toml
[node]
name = "laptop"

[network]
tun_addr = "100.64.0.10/16"

[discovery]
bootstrap_nodes = ["home-server.example.com:9000"]
```

### 15.13.2 Minimal Gateway

```toml
[node]
name = "home-gateway"
role = "gateway"

[network]
tun_addr = "100.64.0.1/16"
advertise_subnets = ["192.168.1.0/24"]

[discovery]
bootstrap_mode = true
public_ip = "203.0.113.5"
```

---

## Summary

This chapter provided a **reference** for QuicEther configuration:
- Sections: `[node]`, `[network]`, `[discovery]`, `[multipath]`, `[security]`, `[monitoring]`, `[overlay]`, `[performance]`
- Key options, types, and typical defaults
- How config files, environment variables, and CLI flags interact

For practical examples, see earlier chapters on deployment, operations, and personas.

**Next Chapter:** We will discuss **future directions & extensions**, including optional blockchain integration and advanced schedulers.

---

**Chapter Navigation:**
- [← Previous: Chapter 14 - Deployment & Operations](./14-deployment-and-operations.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 16 - Future Directions & Extensions](./16-future-directions.md)
