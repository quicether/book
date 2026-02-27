# Chapter 14: Deployment & Operations

## Introduction

This chapter turns QuicEther from a design into something you can **run in the real world**.

We cover:
- Deployment patterns (personal, homelab, small business, enterprise pilot)
- OS-specific setup notes
- Operational runbooks (start, stop, rotate keys, update)
- Monitoring, alerting, and troubleshooting

The goal is to provide concrete playbooks aligned with personas and architecture.

---

## 14.1 Deployment Archetypes

We'll describe four common deployment patterns:

1. **Personal Remote Access** (Sarah / Maria)
2. **Homelab Server** (Alex)
3. **Small Business Multi-Site** (James)
4. **Enterprise Pilot** (David)

Each builds on the same core binary and concepts.

---

## 14.2 Personal Remote Access

### 14.2.1 Scenario

- One or more **home servers**
- One or more **laptops/PCs**
- Goal: Secure access to home services from anywhere

### 14.2.2 Home Server Setup (Linux)

1. Install binary:
   ```bash
   # Example
   curl -L https://get.quicether.org/install.sh | sudo sh
   ```

2. Create config `/etc/quicether/server.toml`:
   ```toml
   [server]
   listen = "0.0.0.0:4433"

   [auth]
   method = "password"

   [[hubs]]
   name = "home"
   dns = ["1.1.1.1"]

   [hubs.local_bridge]
   interface = "eth0"

   [firewall]
   default_action = "deny"

   [[firewall.rules]]
   action = "allow"
   src = "*"
   dst = "*"
   comment = "Allow all traffic within hub"
   ```

3. Set password for clients:
   ```bash
   quicether password set sarah
   ```

4. Start via systemd:
   ```bash
   sudo systemctl enable --now quicether
   ```

### 14.2.3 Laptop Setup (Linux/macOS)

1. Install binary.
2. Connect to home server:
   ```bash
   quicether connect --server home.example.com:4433
   # Prompted for username and password
   ```

3. Or use config file `~/.config/quicether/client.toml`:
   ```toml
   [client]
   server = "home.example.com:4433"

   [auth]
   method = "password"
   username = "sarah"
   ```

4. Start:
   ```bash
   quicether connect --config ~/.config/quicether/client.toml
   ```

5. Validate:
   ```bash
   quicether status
   ping 192.168.1.10      # home NAS
   ssh user@192.168.1.10
   ```

---

## 14.3 Homelab Server

### 14.3.1 Scenario

- One server with multiple hubs
- Several devices (laptops, desktops, phones)
- Optional Local Bridge for exposing home LAN
- Goal: unified access to services (Plex, dev VMs, etc.)

### 14.3.2 Topology

```text
Cloud VPS (public IP)
  └─ quicether server (hub: "home", hub: "dev")

Home LAN (192.168.1.0/24)
  └─ bridge client (Local Bridge to home LAN)

Remote devices
  └─ laptops, phones (connect clients)
```

### 14.3.3 Config Highlights

**Server (Cloud VPS):**
```toml
[server]
listen = "0.0.0.0:4433"

[auth]
method = "password"

[[hubs]]
name = "home"

[[hubs]]
name = "dev"

[firewall]
default_action = "deny"

[[firewall.rules]]
action = "allow"
src = "*"
dst = "*"
comment = "Allow all hub traffic"
```

**Bridge Client (Home LAN):**
```bash
quicether bridge \
  --server vps.example.com:4433 \
  --bridge-interface eth0
```

Operations:
- All devices connect to the VPS server.
- Alex uses `quicether session list` and `quicether hub list` to monitor.

---

## 14.4 Small Business Multi-Site

### 14.4.1 Scenario

- HQ and branch office
- Servers connected via mesh
- Goal: Encrypted site-to-site with optional multipath, plus remote worker access.

### 14.4.2 HQ Server

1. Config `/etc/quicether/server.toml`:
   ```toml
   [server]
   listen = "0.0.0.0:4433"

   [auth]
   method = "password"

   [[hubs]]
   name = "hq"

   [hubs.local_bridge]
   interface = "eth0"

   [mesh]
   enabled = true

   [[mesh.peers]]
   url = "quic://branch.example.com:4433"
   token = "mesh-token-abc"

   [network.multipath]
   interfaces = ["eth0", "eth1"]
   mode = "split"

   [firewall]
   default_action = "deny"

   [[firewall.rules]]
   action = "allow"
   src = "*"
   dst = "*"
   comment = "Allow all hub traffic"

   [audit]
   file = "/var/log/quicether/audit.jsonl"
   syslog = true
   ```

### 14.4.3 Branch Server

```toml
[server]
listen = "0.0.0.0:4433"

[auth]
method = "password"

[[hubs]]
name = "branch"

[hubs.local_bridge]
interface = "eth0"

[mesh]
enabled = true

[[mesh.peers]]
url = "quic://hq.example.com:4433"
token = "mesh-token-abc"

[network.multipath]
interfaces = ["eth0", "eth1"]
mode = "split"
```

### 14.4.4 Remote Worker

```bash
quicether connect --server hq.example.com:4433
# Policy grants limited access per identity/role
```

---

## 14.5 Enterprise Pilot

### 14.5.1 Objectives

- Validate technology in controlled subset of environment
- Integrate with existing monitoring, logging, and IAM

### 14.5.2 Pilot Scope

- 2–3 sites (HQ + 1–2 branches)
- 10–20 remote users
- Dedicated test environment or non-critical segments

### 14.5.3 Integration Steps

1. **PKI Integration:**
   - Use enterprise CA for Ed25519 certificates.
   - Or use password auth with centralized credential management.
   - Automate enrollment with existing tooling.

2. **Monitoring:**
   - Scrape `/metrics` endpoint from Prometheus.
   - Dashboards in Grafana for latency, throughput, peer counts.

3. **Logging:**
   - Ship audit logs via Fluent Bit / Vector to SIEM.

4. **Change Management:**
   - Treat QuicEther config as code (GitOps).
   - PRs for policy changes.

5. **Security Review:**
   - Internal security team review of configs and topology.

---

## 14.6 Operational Runbooks

### 14.6.1 Start/Stop/Restart

**Start server:**
```bash
sudo systemctl start quicether
```

**Stop:**
```bash
sudo systemctl stop quicether
```

**Restart:**
```bash
sudo systemctl restart quicether
```

Or equivalent `quicether server`/`quicether server stop` commands for non-systemd.

### 14.6.2 Checking Health

1. CLI:
   ```bash
   quicether status
   quicether session list   # server-side
   quicether hub list       # server-side
   quicether mac-table
   ```

2. HTTP health endpoint:
   ```bash
   curl http://127.0.0.1:9091/health
   ```

3. Metrics:
   ```bash
   curl http://127.0.0.1:9090/metrics
   ```

### 14.6.3 Key Rotation (Manual)

1. Stop daemon:
   ```bash
   sudo systemctl stop quicether
   ```

2. Backup old key and config.
3. Generate new keypair:
   ```bash
   quicether identity generate
   ```
4. Update server configuration with new identity.
5. Start daemon and verify clients can connect.

### 14.6.4 Updating QuicEther

1. Drain servers where possible:
   - Use admin API to temporarily reject new sessions.
2. Update package/binary.
3. Restart daemon.
4. Monitor metrics and logs for anomalies.

For single-user setups, update can be as simple as replacing binary and restarting.

---

## 14.7 Monitoring & Alerting

### 14.7.1 Key Metrics

- `sessions_active`, `sessions_total`
- `frames_sent`, `frames_recv`
- `bytes_sent`, `bytes_recv`
- Per-path RTT and loss
- Per-hub client counts
- Error counters (failed_auth, firewall_deny, mesh_disconnect)

### 14.7.2 Alert Examples

- **High error rate:**
  - Condition: `failed_handshakes / total_handshakes > 5%` for 5 minutes
  - Action: Alert ops; investigate cert/PKI or network issues.

- **Server down:**
  - Condition: Health endpoint returns `Unhealthy` or unreachable
  - Action: Trigger failover runbook; clients reconnect via mesh peer if available.

- **Path degradation:**
  - Condition: RTT > threshold or loss > threshold on critical paths
  - Action: Alert; check ISP issues.

---

## 14.8 Troubleshooting Guide (Quick Reference)

### 14.8.1 "I Can't Reach Remote Host"

Check list:
1. `quicether status` – is client/server running, TAP up?
2. `quicether session list` – is client session active? (server-side)
3. `quicether mac-table` – is destination MAC learned?
4. OS network – is TAP interface `quicether0` up and receiving DHCP?
5. Firewall rules – is access allowed?
6. Audit logs – any `firewall_deny` or error entries?

### 14.8.2 "High Latency or Poor Throughput"

- Check per-path metrics:
  ```bash
  quicether multipath
  curl http://127.0.0.1:9090/metrics | grep quicether_path_rtt
  ```
- Verify:
  - Is one path much slower? Consider removing it.
  - Is CPU pegged? Consider more powerful hardware or tuning.

### 14.8.3 "Clients Can't Connect to Server"

- Check server is listening:
  - `quicether status` on server
  - Firewall allows UDP port 4433
- Verify auth:
  - Client using correct auth method (password/identity/token)
  - Server has matching credentials configured
- Verify clocks (large skew can break TLS).
- Check audit logs for failed auth events.

---

## Summary

This chapter provided concrete **deployment and operations** guidance:
- Playbooks for personal, homelab, small business, and enterprise pilot scenarios
- OS integration via systemd/launchd and common CLI operations
- Runbooks for health checks, key rotation, and updates
- Monitoring, alerting, and troubleshooting patterns

With these operational practices, QuicEther can be deployed safely and maintained over time.

**Next Chapter:** We will cover **configuration reference and examples**, detailing all key options and recommended defaults.

---

**Chapter Navigation:**
- [← Previous: Chapter 13 - Testing & Validation](./13-testing-and-validation.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 15 - Configuration Reference](./15-configuration-reference.md)
