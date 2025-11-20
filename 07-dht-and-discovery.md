# Chapter 7: DHT & Discovery

## Introduction

The Distributed Hash Table (DHT) is the **nervous system** of QuicEther.

It is responsible for:
- Finding other nodes
- Locating subnets and gateways
- Bootstrapping new nodes into the overlay

This chapter goes deep into:
- DHT design (Kademlia-based)
- Data model (what we store)
- Wire protocol (how nodes talk)
- Security considerations (Sybil, poisoning)
- Operational behavior (joins, leaves, churn)

---

## 7.1 Requirements

Before designing the DHT, we restate what it must do:

1. **Node Discovery**
   - Given `NodeId`, find its current reachable addresses

2. **Subnet Discovery**
   - Given `CIDR`, find which node(s) advertise it

3. **Gateway Discovery**
   - Given a node or subnet, discover which gateway(s) can forward to it

4. **Bootstrap**
   - Allow a fresh node to join the network with minimal prior knowledge

5. **Scale & Resilience**
   - Support 10⁴–10⁶ nodes
   - Tolerate churn (nodes coming and going)

6. **Low Overhead**
   - Background traffic must not dominate bandwidth
   - Lookups should finish in ~O(log N) steps

7. **Security**
   - Make it hard to:
     - Poison records (misleading routes)
     - Impersonate nodes
     - Build trivial Sybil attacks (within reason for MVP)

---

## 7.2 Node Identity & Keyspace

### 7.2.1 NodeId Definition

Each node has an Ed25519 public key `PK`.

We define:

$$
\text{NodeId} = \text{SHA1}(PK) \in \{0,1\}^{160}
$$

**Why SHA‑1?**
- Kademlia traditionally uses a 160‑bit keyspace
- Collision resistance is *not* our security boundary (we rely on Ed25519)
- We can replace SHA‑1 with a 160‑bit truncated SHA‑256 later without changing the DHT structure

In code:

```rust
type NodeId = [u8; 20]; // 160 bits

fn node_id_from_pubkey(pubkey: &[u8]) -> NodeId {
    use sha1::{Digest, Sha1};
    let mut hasher = Sha1::new();
    hasher.update(pubkey);
    let result = hasher.finalize();
    let mut id = [0u8; 20];
    id.copy_from_slice(&result[..20]);
    id
}
```

### 7.2.2 Distance Metric

We use XOR distance as in Kademlia:

$$
\text{dist}(a, b) = a \oplus b
$$

Properties:
- Symmetric
- Defines ordering of "closeness" in keyspace

---

## 7.3 Routing Table Structure

Each node maintains a **routing table** of other nodes, organized into **buckets** based on XOR distance.

### 7.3.1 Buckets

- Keyspace: 160 bits
- Buckets indexed by prefix length (0–159)
- Bucket `i` stores nodes whose distance to us has the `i`‑th most significant bit as the first differing bit

```rust
const K: usize = 20; // Max nodes per bucket

struct RoutingTable {
    node_id: NodeId,
    buckets: Vec<Vec<NodeInfo>>, // 160 buckets
}

struct NodeInfo {
    node_id: NodeId,
    addrs: Vec<SocketAddr>,
    last_seen: Instant,
    capabilities: NodeCapabilities,
}
```

### 7.3.2 Bucket Operations

- **Insert:**
  - If bucket has < K entries → append
  - If bucket is full → ping least recently seen
    - If unresponsive → replace
    - If responsive → consider dropping new node (classic Kademlia) or keep bias for diverse IPs

- **Refresh:**
  - Periodically perform `FIND_NODE` for random IDs in each bucket range

### 7.3.3 NodeCapabilities

```rust
bitflags::bitflags! {
    struct NodeCapabilities: u32 {
        const BOOTSTRAP = 0b00000001; // Public, stable node
        const GATEWAY   = 0b00000010; // Can forward for others
        const MULTIPATH = 0b00000100; // Supports multipath
        const RELAYLESS = 0b00001000; // Explicit: no generic relay
        const ENTERPRISE= 0b00010000; // Enterprise-mode features
    }
}
```

---

## 7.4 Data Model

The DHT stores different record types. All records are **signed** by the publishing node.

### 7.4.1 Node Metadata Record

Key:
- `NODE:<NodeId>` → maps to `NodeRecord`

Value (conceptual):

```json
{
  "node_id": "<hex>",
  "pubkey": "<base64>",
  "addrs": [
    "203.0.113.5:9000",
    "[2001:db8::1]:9000"
  ],
  "capabilities": ["BOOTSTRAP", "GATEWAY"],
  "timestamp": 1732065600,
  "signature": "<sig over above fields>"
}
```

Semantics:
- Publisher: node itself
- TTL: e.g., 1 hour (must be refreshed)

### 7.4.2 Subnet Advertisement Record

Key:
- `SUBNET:<CIDR>` (e.g., `SUBNET:10.0.0.0/16`)

Value:

```json
{
  "owner_node_id": "<hex>",
  "subnet": "10.0.0.0/16",
  "timestamp": 1732065600,
  "ttl": 3600,
  "signature": "<sig by owner_node_id>"
}
```

Semantics:
- Node asserts ownership of subnet
- Used to populate routing tables

### 7.4.3 Gateway Capability Record

Key:
- `GW:<TargetNodeId>` or `GW_SUBNET:<CIDR>`

Value:

```json
{
  "gateway_node_id": "<hex>",
  "target": {
    "type": "node",
    "id": "<TargetNodeId>"
  },
  "timestamp": 1732065600,
  "ttl": 600,
  "signature": "<sig by gateway_node_id>"
}
```

Semantics:
- Node declares it is willing to forward for a specific node or subnet
- Enables gateway discovery for NAT‑broken paths

---

## 7.5 Wire Protocol Overview

We use a simple Kademlia‑inspired RPC set over QUIC streams (or UDP framed packets during bootstrap).

### 7.5.1 Message Types

Core RPCs:
- `PING` / `PONG`
- `FIND_NODE`
- `FIND_VALUE`
- `STORE`

All messages encoded as length‑prefixed protobuf or CBOR. Conceptually:

```rust
enum DhtMessage {
    Ping { from: NodeId },
    Pong { from: NodeId },
    FindNode { from: NodeId, target: NodeId },
    FindValue { from: NodeId, key: Vec<u8> },
    Store { from: NodeId, key: Vec<u8>, value: Vec<u8>, sig: Vec<u8> },
    Nodes { from: NodeId, nodes: Vec<NodeInfo> },
    Value { from: NodeId, key: Vec<u8>, value: Vec<u8> },
}
```

Transport mapping:
- During **bootstrap**: fire‑and‑forget over UDP (QUIC not yet established)
- After connection: use a dedicated QUIC stream per peer for DHT RPCs

### 7.5.2 Basic RPC Flows

**PING/PONG:**
- Health check
- Validates reachability and latency

**FIND_NODE(target):**
- Request nearest known nodes to `target` from peer

**FIND_VALUE(key):**
- Request value for `key` if stored, else nearest nodes to `key`

**STORE(key, value, sig):**
- Request peer to store signed `value` under `key`

---

## 7.6 Lookup Algorithm

We use standard **iterative Kademlia lookup** with small adaptations.

### 7.6.1 FIND_NODE(NodeId)

Pseudocode:

```rust
async fn lookup_node(target: NodeId) -> Vec<NodeInfo> {
    const ALPHA: usize = 3; // Parallelism
    let mut shortlist = routing_table.closest_nodes(target, K);
    let mut contacted = HashSet::new();

    loop {
        // Select up to ALPHA uncontacted closest nodes
        let to_query: Vec<NodeInfo> = shortlist
            .iter()
            .filter(|n| !contacted.contains(&n.node_id))
            .take(ALPHA)
            .cloned()
            .collect();

        if to_query.is_empty() {
            break;
        }

        // Mark as contacted
        for n in &to_query {
            contacted.insert(n.node_id);
        }

        // Send FIND_NODE in parallel
        let responses = send_find_node_parallel(&to_query, target).await;

        // Merge results
        let mut improved = false;
        for resp in responses {
            for node in resp.nodes {
                if !shortlist.iter().any(|n| n.node_id == node.node_id) {
                    shortlist.push(node);
                    improved = true;
                }
            }
        }

        // Sort by distance
        shortlist.sort_by_key(|n| xor_distance(n.node_id, target));
        shortlist.truncate(K);

        if !improved {
            break;
        }
    }

    shortlist
}
```

### 7.6.2 FIND_VALUE(key)

Similar to `FIND_NODE`, but stops early if a value is found.

```rust
async fn lookup_value(key: &[u8]) -> Result<Option<Vec<u8>>> {
    // 1. Try local storage
    if let Some(v) = local_store.get(key) {
        return Ok(Some(v));
    }

    // 2. Kademlia iterative lookup
    //   a) Ask closest nodes
    //   b) If any returns Value → done
    //   c) Else continue like FIND_NODE
}
```

### 7.6.3 Usage in QuicEther

- **Node discovery:** `lookup_node(target_node_id)`
- **Subnet discovery:** `lookup_value(SUBNET:cidr)`
- **Gateway discovery:** `lookup_value(GW:<target>)`

---

## 7.7 Bootstrap Process

New node sequence:

1. **Generate Identity**
   ```bash
   quicether keygen   # or on first start
   ```

2. **Load Config**
   - Read list of `bootstrap_nodes` (IP/port or DNS names)

3. **Contact Bootstrap Nodes**
   - Send `PING` or `FIND_NODE(self_id)` to each bootstrap

4. **Populate Routing Table**
   - Use responses to fill buckets

5. **Announce Presence**
   - `STORE NODE:<self_id>` with own metadata
   - Optionally `STORE SUBNET:<cidr>` and `STORE GW:*` if acting as gateway

6. **Ready for Lookups**

Bootstrapping from multiple sources:
- Public QuicEther bootstrap nodes (default)
- User‑specified private bootstrap nodes
- Cached peers from previous runs

---

## 7.8 Security Considerations

The DHT is a shared, partially trusted structure. We must limit how much damage a malicious node can cause.

### 7.8.1 Record Authenticity

All DHT records that assert identity or authority are **signed**:

- Node metadata → signed by the node’s own key
- Subnet advertisements → signed by advertising node
- Gateway records → signed by gateway node

Verification:
- Reject records with invalid signatures
- Reject stale records (timestamp/TTL check)

### 7.8.2 Sybil Attacks (High‑Level)

Full Sybil resistance on a public Internet‑scale DHT requires:
- Some form of scarce identity (PKI, blockchain, Web of Trust, etc.)

For MVP:
- Focus on **private and semi‑public meshes** where:
  - Bootstrap nodes are controlled by the operator
  - Gateways and key nodes are known and monitored

Future work (Chapter 16+):
- Optional PoA blockchain for public registry
- Rate‑limit new identities per IP / per subnet

### 7.8.3 DHT Poisoning

Threats:
- Malicious node returns incorrect peer lists
- Malicious node stores bogus subnet/gateway mappings

Mitigations:
- Verify signatures and timestamps
- Prefer records confirmed by multiple distinct peers (quorum)
- Limit trust in single peers; use multiple lookup paths

### 7.8.4 Privacy

DHT leaks some metadata:
- Existence of nodes
- Advertised subnets

Mitigations (design choices):
- QuicEther is not an anonymity system (see non‑goals)
- For sensitive deployments:
  - Use private DHT (no public bootstrap)
  - Keep advertisements minimal (e.g., only gateway nodes advertise)

---

## 7.9 Operational Behavior

### 7.9.1 Join & Leave

- **Join:** as in bootstrap process
- **Leave:**
  - Graceful: send `STORE` with TTL=0 (delete) for own records
  - Ungraceful: records expire via TTL

### 7.9.2 Churn Handling

Kademlia is designed for churn:
- Buckets refresh periodically
- Unresponsive nodes pruned
- New nodes discovered via ongoing lookups

QuicEther tuning:
- Bucket refresh interval: e.g., 15 minutes
- Node timeout: e.g., 5 failed pings

### 7.9.3 Load

Target overhead:
- DHT traffic should be **well under 1%** of data traffic in typical usage

Techniques:
- Adaptive refresh (less often on stable networks)
- Coalesce announcements
- Cache successful lookups aggressively with TTL

---

## 7.10 Example Flows

### 7.10.1 Discover Peer by NodeId

```
Goal: Sarah’s laptop wants to connect to home server (NodeId H).

1. Sarah's node S runs lookup_node(H)
2. Uses routing table to contact closest known nodes
3. Iteratively refines shortlist
4. Receives NodeInfo for H: { addrs: [203.0.113.5:9000], capabilities: [...] }
5. Initiates QUIC connection to 203.0.113.5:9000
6. TLS 1.3 mutual auth confirms identity (NodeId H)
```

### 7.10.2 Discover Subnet Owner

```
Goal: Determine who owns 10.1.0.0/16.

1. Node N computes key = "SUBNET:10.1.0.0/16"
2. Runs lookup_value(key)
3. Receives SubnetRecord: { owner_node_id: F, ... }
4. Adds route: 10.1.0.0/16 → NodeId F
5. Uses QUIC tunnel to F for packets to that subnet
```

### 7.10.3 Find Gateway for NAT‑Broken Peer

```
Goal: Maria can't reach Alex's home node directly.

1. Node M attempts direct QUIC to NodeId A → fails (hole punching)
2. M queries DHT for key = "GW:<NodeId A>"
3. Receives GatewayRecord: { gateway_node_id: G, ... }
4. M connects to G
5. Marks routing entry: NodeId A → via gateway G
6. Data path: M → G → A
```

---

## 7.11 Implementation Notes

### 7.11.1 Crate Layout (Rust)

```text
quicether-dht/
  src/
    lib.rs
    node.rs        // DHTNode
    routing.rs     // RoutingTable, NodeInfo
    rpc.rs         // DhtMessage, handlers
    store.rs       // Storage backend
    bootstrap.rs   // Bootstrap logic
    records.rs     // NodeRecord, SubnetRecord, GatewayRecord
```

### 7.11.2 Storage Backend

For MVP:
- In‑memory map with periodic serialization to disk

Later:
- Pluggable backends (e.g., sled, LMDB) for faster restart on large networks

### 7.11.3 Testing Strategy

- Unit tests for:
  - Bucket behavior (insert/evict)
  - XOR distance ordering
  - Signature verification
- Integration tests for:
  - Lookup convergence in simulated networks
  - Churn resilience under random joins/leaves

---

## Summary

In this chapter we:

- Defined the DHT requirements for QuicEther
- Chose a Kademlia‑style DHT with 160‑bit keyspace
- Designed record types for nodes, subnets, and gateways
- Specified wire protocol, lookups, and bootstrap behavior
- Addressed security, churn, and operational constraints

The DHT is the **discovery backbone** enabling:
- Direct node‑to‑node connections
- Subnet routing
- Gateway fallback for hard NAT cases

**Next Chapter:** We will dive into the QUIC transport and multipath implementation details.

---

**Chapter Navigation:**
- [← Previous: Chapter 6 - Technology Choices](./06-technology-choices.md)
- [↑ Table of Contents](./README.md)
- [→ Next: Chapter 8 - QUIC Transport & Multipath](./08-quic-and-multipath.md)
