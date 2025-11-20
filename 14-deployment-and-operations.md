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

We’ll describe four common deployment patterns:

1. **Personal Remote Access** (Sarah / Maria)
2. **Homelab Mesh** (Alex)
3. **Small Business Site‑to‑Site** (James)
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

2. Create config `/etc/quicether/config.toml`:
   ```toml
   [node]
   name = "home-server"

   [network]
   tun_name = "quicether0"
   tun_addr = "100.64.0.1/16"
   advertise_subnets = ["192.168.1.0/24"]

   [discovery]
   bootstrap_mode = true
   public_ip = "203.0.113.5"   # or use dynamic DNS name

   [security]
   policy_file = "/etc/quicether/policy.toml"
   ```

3. Basic policy `/etc/quicether/policy.toml`:
   ```toml
   [[rules]]
   subject = "*"              # any authenticated node for now
   object = "192.168.1.0/24"
   action = "allow"
   ```

4. Start via systemd:
   ```bash
   sudo systemctl enable --now quicether
   ```

### 14.2.3 Laptop Setup (Linux/macOS)

1. Install binary.
2. User‑config `~/.config/quicether/config.toml`:
   ```toml
   [node]
   name = "sarah-laptop"

   [network]
   tun_name = "quicether0"
   tun_addr = "100.64.0.10/16"

   [discovery]
   bootstrap_nodes = ["home-server.example.com:9000"]
   ```

3. Route home subnet via QuicEther:
   ```bash
   sudo ip route add 192.168.1.0/24 dev quicether0   # Linux
   # or equivalent on macOS using `route add`
   ```

4. Start daemon in user session:
   ```bash
   quicether start
   ```

5. Validate:
   ```bash
   quicether status
   ping 192.168.1.10      # home NAS
   ssh user@192.168.1.10
   ```

---

## 14.3 Homelab Mesh

### 14.3.1 Scenario

- Multiple nodes:
  - Home server
  - Colo gateway
  - Cloud VM
  - Laptops/desktops
- Goal: Unified overlay for services (Plex, dev VMs, etc.) with optional multipath.

### 14.3.2 Topology

```text
Home LAN (192.168.1.0/24)
  └─ home-server (gateway)

Colo (1 Gbps uplink)
  └─ colo-gateway (public IP)

Cloud (control / bootstrap)
  └─ cloud-bootstrap

Remote devices
  └─ laptops, phones, etc.
```

### 14.3.3 Config Highlights

**Home Server:**
```toml
[node]
name = "home-server"
role = "gateway"

[network]
tun_addr = "100.64.0.2/16"
advertise_subnets = ["192.168.1.0/24"]

[discovery]
bootstrap_nodes = ["cloud-bootstrap.example.com:9000"]
```

**Colo Gateway:**
```toml
[node]
name = "colo-gateway"
role = "gateway"

[network]
tun_addr = "100.64.0.3/16"

[discovery]
bootstrap_mode = true
public_ip = "203.0.113.5"

[multipath]
interfaces = ["eth0"]
```

**Cloud Bootstrap:**
```toml
[node]
name = "cloud-bootstrap"

[discovery]
bootstrap_mode = true
public_ip = "198.51.100.10"
```

Operations:
- All nodes run `quicether` as a service.
- Alex uses `quicether peers` and `quicether routes` to see the mesh.

---

## 14.4 Small Business Site‑to‑Site

### 14.4.1 Scenario

- HQ and branch office
- Each with two ISPs
- Goal: Encrypted site‑to‑site with optional multipath, plus remote worker access.

### 14.4.2 HQ Gateway

1. Config `/etc/quicether/hq.toml`:
   ```toml
   [node]
   name = "hq-gateway"
   role = "gateway"

   [network]
   tun_addr = "10.0.0.1/16"
   advertise_subnets = ["10.0.0.0/16"]

   [discovery]
   bootstrap_mode = true
   public_ip = "198.51.100.5"

   [multipath]
   enabled = true
   interfaces = ["eth0", "eth1"]
   mode = "aggregate"

   [security]
   policy_file = "/etc/quicether/policy.toml"
   audit_log = "/var/log/quicether/audit.json"
   ```

2. Configure LAN routing so office devices send `10.1.0.0/16` traffic via `hq-gateway`.

### 14.4.3 Branch Gateway

```toml
[node]
name = "branch-gateway"
role = "gateway"

[network]
tun_addr = "10.1.0.1/16"
advertise_subnets = ["10.1.0.0/16"]

[discovery]
bootstrap_nodes = ["hq-gateway.example.com:9000"]

[multipath]
interfaces = ["eth0", "eth1"]
mode = "aggregate"
```

### 14.4.4 Remote Worker

- Config similar to Sarah’s laptop, but bootstrap to `hq-gateway`.
- Policy file grants limited access per role.

---

## 14.5 Enterprise Pilot

### 14.5.1 Objectives

- Validate technology in controlled subset of environment
- Integrate with existing monitoring, logging, and IAM

### 14.5.2 Pilot Scope

- 2–3 sites (HQ + 1–2 branches)
- 10–20 remote users
- Dedicated test environment or non‑critical segments

### 14.5.3 Integration Steps

1. **PKI Integration:**
   - Use enterprise CA for node certificates.
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

**Start:**
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

Or equivalent `quicether start/stop` commands for non‑systemd.

### 14.6.2 Checking Health

1. CLI:
   ```bash
   quicether status
   quicether peers
   quicether routes
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
3. Generate new keypair (tooling to be provided, e.g. `quicether keygen`).
4. Update trust stores / policy with new NodeId.
5. Start daemon and verify peers.

### 14.6.4 Updating QuicEther

1. Drain gateways where possible:
   - Use policy to temporarily disallow new sessions.
2. Update package/binary.
3. Restart daemon.
4. Monitor metrics and logs for anomalies.

For single‑user setups, update can be as simple as replacing binary and restarting.

---

## 14.7 Monitoring & Alerting

### 14.7.1 Key Metrics

- `connections_active`
- `packets_sent`, `packets_recv`
- `bytes_sent`, `bytes_recv`
- Per‑path RTT and loss
- Error counters (failed handshakes, policy denies, DHT timeouts)

### 14.7.2 Alert Examples

- **High error rate:**
  - Condition: `failed_handshakes / total_handshakes > 5%` for 5 minutes
  - Action: Alert ops; investigate cert/PKI or network issues.

- **Gateway down:**
  - Condition: Health endpoint returns `Unhealthy` or unreachable
  - Action: Trigger failover runbook.

- **Path degradation:**
  - Condition: RTT > threshold or loss > threshold on critical paths
  - Action: Alert; check ISP issues.

---

## 14.8 Troubleshooting Guide (Quick Reference)

### 14.8.1 "I Can’t Reach Remote Host"

Check list:
1. `quicether status` – is daemon running, TUN up?
2. `quicether peers` – is remote node connected?
3. `quicether routes` – is there a route for destination subnet?
4. OS routing table – is traffic being sent to `quicether0`?
5. Policy – is access allowed in policy file?
6. Logs – any `deny` or error entries?

### 14.8.2 "High Latency or Poor Throughput"

- Check per‑path metrics:
  ```bash
  quicether multipath
  curl http://127.0.0.1:9090/metrics | grep quicether_path_rtt
  ```
- Verify:
  - Is one path much slower? Consider removing it.
  - Is CPU pegged? Consider more powerful hardware or tuning.

### 14.8.3 "Nodes Can’t Discover Each Other"

- Check DHT health:
  - Logs for DHT timeouts
  - Bootstrap node reachability
- Verify clocks (large skew can break TLS).

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
