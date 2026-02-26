# QuicEther Architecture Diagrams

> **Post-httpf Revision:** These diagrams reflect the validated hub-based,
> server mesh architecture ported from HTTP Fabric to native QUIC,
> using Layer 2 (TAP/Ethernet) switching inspired by SoftEther.

## Diagram 1: System Architecture

```mermaid
graph TD
    %% Server layer
    subgraph Server["QuicEther Server"]
        QUIC["QUIC Listener (TLS 1.3)"]
        AUTH["Auth Service<br/>(Ed25519 / Password / Token)"]
        HUB["Hub Manager<br/>(Virtual Hubs / Ethernet Segments)"]
        FW["Firewall Engine<br/>(L2/L3/L4)"]
        SWITCH["Virtual Hub Switch<br/>(MAC Learning)"]
        AUDIT["Audit Logger<br/>(JSONL + Syslog)"]
        MESH["Mesh Manager<br/>(Cascade Connections)"]
        TAP_S["TAP Interface / Local Bridge"]
    end

    %% Client layer
    subgraph Client["QuicEther Client"]
        CONN["QUIC Connection"]
        MP["Multipath Manager"]
        PERF["Performance Profile<br/>(latency/balanced/throughput)"]
        DEV["Device Abstraction Layer"]
        TAP_C["Native TAP<br/>(Linux/macOS/Windows)"]
        VTAP["Virtual TAP<br/>(iOS/Android/containers)<br/>TUN + L2↔L3 translation"]
    end

    %% Physical networks
    subgraph Underlay["Physical Networks"]
        ISP1["ISP / WAN #1"]
        ISP2["ISP / WAN #2 (failover)"]
    end

    %% Client to server flow
    TAP_C -->|"Ethernet frames"| DEV
    VTAP -->|"Ethernet frames<br/>(synthesized)"| DEV
    DEV --> CONN
    CONN -->|"FrameBatch (LZ4 + ChaCha20)"| MP
    MP <--> ISP1
    MP <--> ISP2
    ISP1 & ISP2 -->|"QUIC streams"| QUIC
    QUIC --> AUTH
    AUTH --> HUB
    HUB --> FW
    FW --> SWITCH
    SWITCH <--> TAP_S
    SWITCH --> AUDIT
    HUB --> MESH

    %% Styling
    classDef server fill:#e3f2fd,stroke:#1e88e5;
    classDef client fill:#e8f5e9,stroke:#43a047;
    classDef underlay fill:#f3e5f5,stroke:#8e24aa;

    class Server,QUIC,AUTH,HUB,FW,SWITCH,AUDIT,MESH,TAP_S server;
    class Client,CONN,MP,PERF,DEV,TAP_C,VTAP client;
    class Underlay,ISP1,ISP2 underlay;
```

## Diagram 2: Server Mesh Topology

```mermaid
graph TD
    %% Three-region mesh
    subgraph US["US-East Server"]
        S1["Server 1"]
        H1["Hub: us-east"]
        C1A["Client A"]
        C1B["Client B"]
    end

    subgraph EU["EU-West Server"]
        S2["Server 2"]
        H2["Hub: eu-west"]
        C2A["Client C"]
        C2B["Client D"]
    end

    subgraph AP["AP-South Server"]
        S3["Server 3"]
        H3["Hub: ap-south"]
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

    %% Cascade connection example
    C1A -.->|"Cascade:<br/>A → S1 → S2 → C"| C2A

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
    S->>S: Assign MAC, join Virtual Hub
    S->>S: Apply firewall rules
    S-->>C: Session established (hub, MAC, TAP config)
    
    loop Data plane
        C->>Q: FrameBatch (LZ4 compressed, ChaCha20 encrypted)
        Q->>H: Switch through firewall + MAC table
        H-->>C: Response frames
    end
```

## Diagram 4: Hub Multi-Tenancy

```mermaid
graph LR
    subgraph Server["QuicEther Server"]
        subgraph HubEng["Hub: engineering"]
            E1["Alice<br/>MAC: a1:b2:c3:.."]
            E2["Bob<br/>MAC: d4:e5:f6:.."]
            FW1["Firewall:<br/>ACCEPT tcp/22,443<br/>DROP all"]
        end
        
        subgraph HubGuest["Hub: guests"]
            G1["Visitor<br/>MAC: 01:23:45:.."]
            FW2["Firewall:<br/>ACCEPT tcp/80,443<br/>DROP all"]
        end
        
        subgraph HubBridge["Hub: datacenter"]
            B1["Site-B Bridge"]
            FW3["Firewall:<br/>ACCEPT all<br/>(trusted)"]
            LAN["Local Bridge → eth0<br/>(192.168.1.0/24)"]
        end
    end
    
    %% Isolation
    HubEng ~~~ HubGuest
    HubGuest ~~~ HubBridge
    
    %% Notes
    B1 --> LAN
```

## Diagram 5: Device Abstraction Layer (Native TAP vs Virtual TAP)

```mermaid
graph TD
    subgraph NativeMode["Native TAP Mode (Linux/macOS/Windows)"]
        APP_N["Application"]
        KERNEL_N["Kernel Network Stack"]
        TAP_N["TAP Device (quicether0)<br/>Raw Ethernet frames"]
        QE_N["QuicEther Client"]
    end

    subgraph VirtualMode["Virtual TAP Mode (iOS/Android/Containers)"]
        APP_V["Application"]
        KERNEL_V["Kernel Network Stack"]
        TUN_V["TUN Device<br/>Raw IP packets only"]
        VTAP_V["Virtual TAP Engine"]
        ARP_V["ARP Proxy / Cache<br/>MAC↔IP learning"]
        QE_V["QuicEther Client"]
    end

    subgraph Overlay["QUIC Overlay (Always L2)"]
        TUNNEL["QUIC Tunnel<br/>Ethernet Frames"]
        HUB["Virtual Hub<br/>(MAC Switch)"]
    end

    %% Native TAP flow
    APP_N --> KERNEL_N
    KERNEL_N -->|"ARP + Ethernet"| TAP_N
    TAP_N -->|"Ethernet frame"| QE_N
    QE_N -->|"Ethernet frame"| TUNNEL

    %% Virtual TAP flow
    APP_V --> KERNEL_V
    KERNEL_V -->|"IP packet"| TUN_V
    TUN_V -->|"IP packet"| VTAP_V
    ARP_V <-->|"MAC lookup"| VTAP_V
    VTAP_V -->|"+ Ethernet header<br/>(14 bytes)"| QE_V
    QE_V -->|"Ethernet frame"| TUNNEL

    TUNNEL --> HUB

    %% Styling
    classDef native fill:#e8f5e9,stroke:#43a047;
    classDef virtual fill:#fff3e0,stroke:#fb8c00;
    classDef overlay fill:#e3f2fd,stroke:#1e88e5;

    class NativeMode,APP_N,KERNEL_N,TAP_N,QE_N native;
    class VirtualMode,APP_V,KERNEL_V,TUN_V,VTAP_V,ARP_V,QE_V virtual;
    class Overlay,TUNNEL,HUB overlay;
```