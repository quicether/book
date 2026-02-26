# QuicEther Architecture Diagrams

> **Post-httpf Revision:** These diagrams reflect the validated hub-based,
> server mesh architecture ported from HTTP Fabric to native QUIC.

## Diagram 1: System Architecture

```mermaid
graph TD
    %% Server layer
    subgraph Server["QuicEther Server"]
        QUIC["QUIC Listener (TLS 1.3)"]
        AUTH["Auth Service<br/>(Ed25519 / Password / Token)"]
        HUB["Hub Manager<br/>(Multi-tenant IP Pools)"]
        FW["Firewall Engine<br/>(Proxmox-style)"]
        ROUTER["Router + Virtual NAT"]
        AUDIT["Audit Logger<br/>(JSONL + Syslog)"]
        MESH["Mesh Manager<br/>(Cascade Routing)"]
        TUN_S["TUN Interface"]
    end

    %% Client layer
    subgraph Client["QuicEther Client"]
        CONN["QUIC Connection"]
        MP["Multipath Manager"]
        PERF["Performance Profile<br/>(latency/balanced/throughput)"]
        TUN_C["TUN Interface"]
    end

    %% Physical networks
    subgraph Underlay["Physical Networks"]
        ISP1["ISP / WAN #1"]
        ISP2["ISP / WAN #2 (failover)"]
    end

    %% Client to server flow
    TUN_C -->|"IP packets"| CONN
    CONN -->|"PacketBatch (LZ4 + ChaCha20)"| MP
    MP <--> ISP1
    MP <--> ISP2
    ISP1 & ISP2 -->|"QUIC streams"| QUIC
    QUIC --> AUTH
    AUTH --> HUB
    HUB --> FW
    FW --> ROUTER
    ROUTER <--> TUN_S
    ROUTER --> AUDIT
    HUB --> MESH

    %% Styling
    classDef server fill:#e3f2fd,stroke:#1e88e5;
    classDef client fill:#e8f5e9,stroke:#43a047;
    classDef underlay fill:#f3e5f5,stroke:#8e24aa;

    class Server,QUIC,AUTH,HUB,FW,ROUTER,AUDIT,MESH,TUN_S server;
    class Client,CONN,MP,PERF,TUN_C client;
    class Underlay,ISP1,ISP2 underlay;
```

## Diagram 2: Server Mesh Topology

```mermaid
graph TD
    %% Three-region mesh
    subgraph US["US-East Server"]
        S1["Server 1"]
        H1["Hub: 10.20.1.0/24"]
        C1A["Client A"]
        C1B["Client B"]
    end

    subgraph EU["EU-West Server"]
        S2["Server 2"]
        H2["Hub: 10.20.2.0/24"]
        C2A["Client C"]
        C2B["Client D"]
    end

    subgraph AP["AP-South Server"]
        S3["Server 3"]
        H3["Hub: 10.20.3.0/24"]
        C3A["Client E"]
    end

    %% Client connections
    C1A & C1B -->|"QUIC"| S1
    C2A & C2B -->|"QUIC"| S2
    C3A -->|"QUIC"| S3

    %% Mesh links
    S1 <-->|"Mesh QUIC<br/>(shared token)"| S2
    S2 <-->|"Mesh QUIC<br/>(shared token)"| S3
    S1 <-->|"Mesh QUIC<br/>(shared token)"| S3

    %% Cascade routing example
    C1A -.->|"Cascade route:<br/>A → S1 → S2 → C"| C2A

    %% Styling
    classDef server fill:#e3f2fd,stroke:#1e88e5;
    classDef client fill:#e8f5e9,stroke:#43a047;

    class S1,S2,S3,H1,H2,H3 server;
    class C1A,C1B,C2A,C2B,C3A client;
```

## Diagram 3: Authentication & Session Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant Q as QUIC/TLS 1.3
    participant A as Auth Service
    participant H as Hub Manager
    participant S as Session Manager

    C->>Q: QUIC handshake (0-RTT if resuming)
    Q->>A: Auth frame (method + credentials)
    
    alt Identity Auth
        A->>A: Verify Ed25519 signature
        A->>A: Derive BLAKE3 identity ID
    else Password Auth
        A->>A: Verify Argon2id hash
        A->>A: Check lockout status
    else Service Token
        A->>A: Verify SHA-256 token hash
        A->>A: Check expiry + max sessions
    end
    
    A->>H: Request hub access
    H->>H: Check access control (identities/groups)
    H->>S: Create session
    S->>S: Allocate IP from CIDR pool
    S->>S: Apply firewall rules
    S-->>C: Session established (assigned IP, routes, DNS)
    
    loop Data plane
        C->>Q: PacketBatch (LZ4 compressed, ChaCha20 encrypted)
        Q->>H: Route through firewall + NAT
        H-->>C: Response packets
    end
```

## Diagram 4: Hub Multi-Tenancy

```mermaid
graph LR
    subgraph Server["QuicEther Server"]
        subgraph HubEng["Hub: engineering<br/>10.20.1.0/24"]
            E1["Alice<br/>10.20.1.2"]
            E2["Bob<br/>10.20.1.3"]
            FW1["Firewall:<br/>ACCEPT tcp/22,443<br/>DROP all"]
        end
        
        subgraph HubGuest["Hub: guests<br/>10.20.2.0/24"]
            G1["Visitor<br/>10.20.2.2"]
            FW2["Firewall:<br/>ACCEPT tcp/80,443<br/>DROP all"]
        end
        
        subgraph HubBridge["Hub: datacenter<br/>10.20.3.0/24"]
            B1["Site-B Bridge<br/>10.20.3.2"]
            FW3["Firewall:<br/>ACCEPT all<br/>(trusted)"]
            LAN["→ 192.168.1.0/24"]
        end
    end
    
    %% Isolation
    HubEng ~~~ HubGuest
    HubGuest ~~~ HubBridge
    
    %% Notes
    B1 --> LAN
```