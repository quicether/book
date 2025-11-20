# Chapter 13: Testing & Validation

## Introduction

QuicEther is a **networking + security** system. Bugs can mean:
- Data leaks
- Outages
- Subtle performance problems

This chapter defines how we test and validate QuicEther:
- Testing layers (unit, integration, system, performance)
- Test environments (local, lab, cloud)
- Scenario coverage mapped to personas
- Automation and tooling

The goal is to catch regressions early and build confidence before production use.

---

## 13.1 Testing Layers

We use multiple layers of testing:

1. **Unit tests:**
   - Small, fast, deterministic
   - Validate pure logic (routing tables, DHT buckets, policy engine)

2. **Integration tests:**
   - Multiple components within one process
   - e.g., DHT + transport, policy + router

3. **System tests (multi‑node):**
   - Multiple QuicEther nodes, simulated or real
   - Exercise full flows: discovery → connect → route → policy

4. **Performance tests:**
   - Throughput, latency, CPU usage

5. **Chaos & resilience tests:**
   - Interface flaps, node restarts, packet loss, partitions

---

## 13.2 Unit Tests

### 13.2.1 Targets

- DHT routing table:
  - Bucket insert/eviction
  - XOR distance ordering
- Policy engine:
  - Rule matching (allow/deny)
  - Group membership resolution
- Routing:
  - Longest‑prefix match
  - Local vs Node vs Gateway precedence
- Config loader:
  - Precedence (defaults, file, env, flags)
- Metrics helpers:
  - Counters, histograms, gauges

### 13.2.2 Tooling

- Rust’s built‑in `cargo test`
- `proptest` or `quickcheck` for property‑based tests where useful

Example (routing longest‑prefix match):

```rust
#[test]
fn longest_prefix_match_prefers_more_specific_route() {
    let mut table = OverlayRoutingTable::new();
    table.add("10.0.0.0/8", RouteNextHop::Node(node_a()));
    table.add("10.0.1.0/24", RouteNextHop::Node(node_b()));

    let dest: IpAddr = "10.0.1.42".parse().unwrap();
    let nh = table.lookup(dest).unwrap();
    assert_eq!(nh, RouteNextHop::Node(node_b()));
}
```

---

## 13.3 Integration Tests

Integration tests combine multiple modules in‑process.

### 13.3.1 Examples

- **DHT + Node Identity:**
  - Generate many NodeIds
  - Insert into routing table
  - Test `FIND_NODE` correctness

- **Router + Policy:**
  - Configure routes and policies
  - Feed in synthetic packets
  - Verify allow/deny + next hop decisions

- **QUIC Transport + Encapsulation:**
  - Spin up QUIC endpoints in test
  - Send encapsulated IP packets
  - Confirm they emerge intact on the other side

### 13.3.2 Layout

- Place in `tests/` directory with descriptive names:
  - `tests/dht_integration.rs`
  - `tests/router_policy.rs`
  - `tests/quic_tunnel.rs`

---

## 13.4 System Tests (Multi‑Node)

System tests validate real‑world scenarios.

### 13.4.1 Local Multi‑Node with Docker Compose

Use `docker-compose` to simulate a small network:

```yaml
version: '3.8'
services:
  node1:
    image: quicether:test
    command: ["quicether", "start", "--config", "/config/node1.toml"]
    network_mode: host

  node2:
    image: quicether:test
    command: ["quicether", "start", "--config", "/config/node2.toml"]
    network_mode: host
```

While `network_mode: host` is one option, we can also use docker networks to simulate NAT behavior.

Tests scripted with:
- `docker-compose up -d`
- Run `quicether` CLI from host to inspect peers/routes
- Run `ping`/`iperf` between containers

### 13.4.2 Specific Scenarios

1. **Personal VPN:**
   - Node A (home), Node B (laptop)
   - Verify:
     - Discovery via DHT
     - Direct QUIC connection
     - Ping + SSH over overlay

2. **Site‑to‑Site:**
   - HQ and Factory with distinct subnets
   - Validate:
     - Subnet advertisement
     - Overlay routing
     - Failover via gateway when direct path removed

3. **Multipath:**
   - Node with multiple interfaces, separate docker networks as different ISPs
   - Validate traffic distribution and failover

4. **Policy Enforcement:**
   - Define allow/deny rules
   - Confirm unauthorized attempts are blocked and logged

---

## 13.5 Performance Testing

### 13.5.1 Metrics

We care about:
- Throughput (Gbps) per node
- Latency overhead vs raw IP
- CPU usage per core
- Memory footprint per connection

### 13.5.2 Tools

- `iperf3` (inside overlay) for throughput
- `ping` for latency
- Custom load generator if needed for many concurrent connections

### 13.5.3 Scenarios

1. **Single Path Baseline:**
   - One interface, direct connection
   - Measure baseline overhead vs plain TCP/UDP

2. **Multi‑Path Aggregation:**
   - 2–4 paths with controlled bandwidth limits
   - Confirm effective throughput approaches target (≥80% of sum)

3. **High Connection Counts:**
   - Many concurrent connections (100s–1000s)
   - Monitor CPU, memory, and latency

Results feed back into performance targets from Chapter 5.

---

## 13.6 Resilience & Chaos Testing

### 13.6.1 Interface Failures

Use tools like `tc` (Linux) to simulate:
- Interface down/up cycles
- Packet loss, delay, and reordering

Examples:

```bash
# Add 100ms delay and 1% loss on eth0
tc qdisc add dev eth0 root netem delay 100ms loss 1%

# Remove impairment
tc qdisc del dev eth0 root
```

Validation:
- Connections should remain established
- Multipath should reroute or reweight paths

### 13.6.2 Node Crashes

- Kill `quicether` process on one node abruptly
- Observe behavior on peers:
  - Time to detect failure
  - How quickly routes are updated

### 13.6.3 DHT Partitions

- Use firewall rules or docker networks to partition subsets of nodes
- Validate:
  - Existing connections continue when possible
  - New lookups eventually succeed when partition heals

---

## 13.7 Security Testing

### 13.7.1 Static Analysis

- Enable `cargo clippy` with strict lints
- Use `cargo audit` for dependency vulnerabilities

### 13.7.2 Fuzzing

Targets:
- DHT message parsing
- Config file parsing
- Policy parser

Tools:
- `cargo-fuzz` with AFL/libFuzzer integration

### 13.7.3 Penetration Testing

- External security review once v1.0 candidate is ready
- Focus areas:
  - TLS usage and key handling
  - DHT poisoning and injection protections
  - Policy bypass attempts

---

## 13.8 CI/CD Strategy

### 13.8.1 Continuous Integration

Every push/PR should trigger:
- `cargo fmt --check`
- `cargo clippy -- -D warnings`
- `cargo test` (unit + small integration)

Nightly or scheduled jobs:
- Extended integration tests (multi‑node via docker)
- Fuzzing runs (time‑bounded)

### 13.8.2 Release Validation

Before tagging v0.x / v1.x releases:
- Run full system test suite
- Run performance benchmarks on reference hardware
- Run basic chaos tests (interface flap, node restart)

Artifacts:
- Binary builds for supported platforms
- Checksums and signatures

---

## 13.9 Mapping Tests to Personas

To ensure we don’t miss real‑world workflows:

- **Sarah (Developer):**
  - Remote access tests
  - Roaming tests (WiFi ↔ wired)

- **James (Small Business):**
  - Site‑to‑site tests
  - Policy tests (departmental segmentation)

- **Alex (Homelab):**
  - Multi‑node mesh tests
  - Multipath experiments

- **Maria (Nomad):**
  - Multipath + captive portal edge cases

- **Priya (Rural):**
  - Multi‑ISP aggregation under varying path quality

- **David (Enterprise):**
  - Audit log completeness
  - Integration with external monitoring

---

## Summary

This chapter defined how we will validate QuicEther:
- Layered testing (unit → integration → system → performance → chaos)
- Concrete test scenarios tied to architecture and personas
- CI/CD practices to keep quality high as the codebase grows

Testing is not a one‑time phase; it is an ongoing process built into every milestone of the implementation roadmap.

**Next Chapter:** We will describe **deployment and operations playbooks**—how to actually run QuicEther in different environments.

---

**Chapter Navigation:**
- [← Previous: Chapter 12 - Implementation Roadmap](./12-implementation-roadmap.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 14 - Deployment & Operations](./14-deployment-and-operations.md)
