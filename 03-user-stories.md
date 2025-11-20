# Chapter 3: User Stories & Use Cases

## Personas

To design QuicEther effectively, we need to understand who will use it and what they need. Let's define specific personas with real constraints, motivations, and pain points.

---

## Persona 1: Sarah - Remote Software Developer

### Background
- **Age:** 32
- **Location:** Portland, Oregon (works remotely)
- **Technical Level:** High (professional developer)
- **Internet:** Home fiber (500 Mbps down / 100 Mbps up)

### Current Situation
Sarah works for a startup based in San Francisco. She needs to:
- Access company GitLab server (internal)
- Connect to development databases
- SSH into staging servers
- Collaborate on internal Slack/wikis

**Current Solution:** Company VPN (OpenVPN)

**Pain Points:**
- VPN slows down connection (SF → NY → Portland routing)
- Split-tunnel configuration is complex
- VPN drops on network changes (WiFi to Ethernet)
- Cannot work offline (needs VPN for internal resources)

### QuicEther Use Case

**What Sarah Wants:**
```
Home Network (192.168.1.0/24)
  ↓
  Sarah's Laptop (172.31.1.5)
  ↓
  QuicEther overlay
  ↓
Company Office Network (10.0.0.0/16)
  ↓
  GitLab (10.0.1.10)
  Databases (10.0.2.x)
  Staging (10.0.3.x)
```

**Setup:**
```bash
# Sarah's laptop
quicether init
quicether start --bootstrap company-dht.startup.com

# Automatically discovers company office gateway
# Routes 10.0.0.0/16 → via office-gateway
# Everything else → via local ISP (no split-tunnel config)
```

**Success Criteria:**
- ✅ Setup takes < 5 minutes
- ✅ Seamless roaming (WiFi ↔ Ethernet, home ↔ coffee shop)
- ✅ No performance degradation vs direct connection
- ✅ Works offline (local dev environment cached)

---

## Persona 2: James - Small Business Owner

### Background
- **Age:** 45
- **Business:** Accounting firm (12 employees)
- **Technical Level:** Low (hires external IT consultant)
- **Budget:** $500/month for all IT infrastructure
- **Locations:** 
  - Main office (8 people)
  - Home office (2 partners)
  - 2 remote accountants

### Current Situation

**Current Solution:** QuickBooks hosted locally, Dropbox for files, Zoom for meetings

**Pain Points:**
- QuickBooks server is in main office (slow remote access)
- Dropbox sync conflicts
- Hiring external IT costs $150/hour
- Wants to add security cameras accessible remotely

### QuicEther Use Case

**What James Needs:**
```
Main Office Network (192.168.1.0/24)
  - QuickBooks Server (192.168.1.10)
  - File Server (192.168.1.11)
  - Security Cameras (192.168.1.20-25)

Home Office 1 (192.168.2.0/24)
  - Partner A's Desktop

Home Office 2 (192.168.3.0/24)
  - Partner B's Desktop

Remote Worker 1 (laptop, dynamic)
Remote Worker 2 (laptop, dynamic)
```

**Setup (by IT consultant):**
```bash
# Main office - Gateway
quicether init --role gateway
quicether start \
  --advertise-subnet 192.168.1.0/24 \
  --bootstrap business-dht.example.com

# Home offices - Gateways
quicether init --role gateway
quicether start \
  --advertise-subnet 192.168.2.0/24 \
  --connect-to main-office-gateway

# Remote workers - Clients
quicether init
quicether start \
  --connect-to main-office-gateway
```

**Success Criteria:**
- ✅ IT consultant can setup in 2 hours (vs 8 hours for traditional VPN)
- ✅ Total cost < $50/month (vs $400/month for commercial VPN)
- ✅ Employees can access QuickBooks at native speed
- ✅ James can view security cameras from his phone

**Acceptance Test:**
James opens his phone, launches security camera app, sees live feed from main office in < 3 seconds.

---

## Persona 3: Alex - Homelab Enthusiast

### Background
- **Age:** 28
- **Occupation:** DevOps engineer (hobby: homelab)
- **Technical Level:** Very High
- **Internet:** 
  - Home: Fiber 1 Gbps
  - Colo: Friend's house with gigabit
  - Cloud: Oracle free tier ARM VM

### Current Situation

**Current Setup:**
- Home server rack (NAS, Plex, VMs, game servers)
- Colo box at friend's house (backup, public IP)
- Cloud VM for experimenting

**Current Solution:** WireGuard tunnels manually configured

**Pain Points:**
- WireGuard config is static (IP changes require reconfiguration)
- No automatic discovery (new devices need manual peer addition)
- Cannot aggregate home internet connections (has 2 ISPs for redundancy but only uses 1)
- Friend's internet is faster upload; wants to use it for Plex remote streaming

### QuicEther Use Case

**What Alex Wants:**
```
Home Network
  - NAS (TrueNAS)
  - Plex Server
  - Kubernetes cluster (5 VMs)
  - 2× Internet connections (ISP 1: fiber, ISP 2: cable backup)

Colo (Friend's House)
  - Backup server
  - Fast uplink (1 Gbps symmetric)

Cloud (Oracle Free Tier)
  - Development VM
  - Public IP for remote access
```

**Advanced Setup:**
```bash
# Home - Main node with multipath
quicether init --name home-server
quicether start \
  --interfaces eth0,eth1 \  # Both ISPs
  --multipath aggregate \
  --advertise-subnet 192.168.1.0/24

# Colo - High-bandwidth gateway
quicether init --name colo-gateway --role gateway
quicether start \
  --advertise-bandwidth 1Gbps \
  --forward-for home-server

# Cloud - Public bootstrap node
quicether init --name cloud-bootstrap --bootstrap-mode
quicether start \
  --public-ip 150.230.x.x \
  --advertise-as-bootstrap

# Laptop - Mobile client
quicether init
quicether start
# Auto-discovers all other nodes via cloud-gateway
```

**Advanced Use Case - Plex Streaming:**
```
Remote User wants to stream 4K movie from Alex's Plex
  ↓
Plex Server → QuicEther → Colo Gateway (fast uplink) → Remote User
  (forwards via colo's 1 Gbps uplink instead of home's limited upload)
```

**Success Criteria:**
- ✅ Zero-config discovery (new devices auto-join)
- ✅ Multi-path aggregation works (2 ISPs = 2× bandwidth)
- ✅ Can forward Plex through colo gateway for better upload
- ✅ Survives IP changes (dynamic DNS not needed)

**Acceptance Test:**
Alex adds a new Raspberry Pi to homelab, runs `quicether start`, and it immediately appears in his mesh topology with no configuration.

---

## Persona 4: Maria - Digital Nomad Content Creator

### Background
- **Age:** 35
- **Occupation:** YouTube creator / photographer
- **Technical Level:** Medium
- **Travel:** 10+ countries per year
- **Internet:** Hotel WiFi, coffee shop, mobile hotspot, coworking spaces

### Current Situation

**Workflow:**
- Shoots 4K video (100-500 GB per project)
- Needs to upload to home NAS for editing team to access
- Works from various locations with unreliable internet

**Current Solution:** Dropbox Business ($20/month) + VPN for home NAS access

**Pain Points:**
- Single internet connection often slow/unreliable
- Hotel WiFi blocks VPN ports
- Dropbox sync stalls on large files
- Cannot aggregate mobile + hotel WiFi
- Upload speed bottleneck (critical for her workflow)

### QuicEther Use Case

**What Maria Needs:**
```
Home Office (Barcelona)
  - NAS (10 TB storage)
  - Editor's workstation

Maria's Setup (traveling)
  - Laptop
  - Mobile hotspot (5G, 100 Mbps)
  - Hotel WiFi (50 Mbps, unreliable)
  - Coffee shop WiFi (30 Mbps)
```

**Multipath Aggregation Setup:**
```bash
# Home NAS - Gateway
quicether init --name home-nas
quicether start \
  --advertise-subnet 192.168.1.0/24 \
  --accept-multipath

# Maria's Laptop - Multipath client
quicether init --name maria-laptop
quicether start \
  --interfaces wlan0,wwan0 \  # WiFi + mobile
  --multipath aggregate \
  --target home-nas
```

**Real-World Scenario:**
```
Maria is in Bangkok hotel, needs to upload 200 GB footage

Single connection options:
- Hotel WiFi: 50 Mbps up = 9 hours
- Mobile hotspot: 100 Mbps up = 4.5 hours

QuicEther multipath:
- Hotel WiFi (50 Mbps) + Mobile (100 Mbps) = 150 Mbps
- Upload time: 3 hours
- Automatic failover if hotel WiFi drops
```

**Success Criteria:**
- ✅ Works through restrictive hotel/public WiFi
- ✅ Aggregates multiple connections (WiFi + mobile)
- ✅ Survives connection changes (moving between locations)
- ✅ Mobile app available (iOS/Android)

**Acceptance Test:**
Maria arrives at hotel, connects laptop to hotel WiFi, enables mobile hotspot, and QuicEther automatically uses both for uploading footage at combined speed.

---

## Persona 5: Priya - Rural Internet User

### Background
- **Age:** 40
- **Location:** Rural Montana
- **Occupation:** Freelance graphic designer
- **Technical Level:** Medium
- **Internet Challenge:** No single high-speed ISP available

### Current Situation

**Available Internet Options:**
- DSL: 25 Mbps down / 5 Mbps up ($40/month)
- Cable: 50 Mbps down / 10 Mbps up ($60/month)
- 4G LTE: 50 Mbps down / 20 Mbps up ($50/month)
- Starlink: 100 Mbps down / 20 Mbps up ($110/month)

**Current Solution:** Uses Starlink only (most expensive)

**Pain Points:**
- Expensive ($110/month)
- Starlink has high latency (satellite)
- Starlink drops during storms
- Other connections sit idle (waste of potential bandwidth)

### QuicEther Use Case

**What Priya Wants:**
```
Use ALL connections simultaneously:
- DSL (25/5)
- Cable (50/10)
- 4G LTE (50/20)
- Starlink (100/20)

Total Available:
- Download: 225 Mbps
- Upload: 55 Mbps

Cost: $260/month (vs $110 for Starlink alone)
But 2.25× download, 2.75× upload
```

**Setup:**
```bash
# Priya's desktop with USB Ethernet adapters
quicether init
quicether start \
  --interfaces eth0,eth1,wwan0,sat0 \
  --multipath aggregate \
  --scheduler weighted \
  --target datacenter-node

# Cheap cloud VM as gateway node
# (Hetzner: $5/month)
quicether init --name datacenter-gateway --role gateway
quicether start --accept-multipath
```

**Path Quality Handling:**
```
QuicEther automatically handles:
- DSL: Stable, low latency → prefer for interactive traffic
- Cable: Medium quality → use for bulk transfer
- 4G LTE: Variable latency → use when others saturated
- Starlink: High latency → use only for large downloads, avoid for video calls
```

**Success Criteria:**
- ✅ Aggregates all 4 connections for bulk transfers
- ✅ Routes video calls through DSL (low latency)
- ✅ Automatic failover when Starlink drops (storm)
- ✅ Cost justified by improved reliability + speed

**Acceptance Test:**
Priya downloads a 10 GB software update. Instead of taking 13 minutes (Starlink alone), it takes 6 minutes (all connections combined). During download, a storm causes Starlink to drop; download continues at reduced speed without interruption.

---

## Persona 6: David - Enterprise IT Manager

### Background
- **Age:** 50
- **Company:** Manufacturing company (200 employees)
- **Technical Level:** High (CCNA certified)
- **Current VPN:** Cisco AnyConnect ($50,000/year license + support)

### Current Situation

**Locations:**
- HQ (New York): 100 employees
- Factory (Ohio): 80 employees
- Warehouse (Texas): 15 employees
- Remote sales team: 5 people

**Current Solution:** Cisco AnyConnect + Meraki SD-WAN

**Pain Points:**
- High cost ($50k/year VPN + $30k/year SD-WAN)
- Vendor lock-in (difficult to migrate)
- Limited throughput (1 Gbps per site cap)
- Requires expensive hardware refresh every 5 years

### QuicEther Use Case

**What David Evaluates:**

**Evaluation Criteria:**
```
Requirements:
1. Site-to-site connectivity (HQ ↔ Factory ↔ Warehouse)
2. Remote access for sales team
3. Multi-path WAN aggregation at each site
4. Central management and monitoring
5. Compliance logging (SOC 2)
6. 99.9% uptime SLA
```

**Proof of Concept Setup:**
```bash
# HQ - Main gateway + DHT bootstrap
quicether init --name hq-gateway --enterprise
quicether start \
  --advertise-subnet 10.0.0.0/16 \
  --interfaces eth0,eth1 \  # Dual ISPs
  --multipath failover \
  --bootstrap-mode \
  --audit-log /var/log/quicether/audit.log

# Factory - Gateway with multipath
quicether init --name factory-gateway --enterprise
quicether start \
  --advertise-subnet 10.1.0.0/16 \
  --interfaces eth0,eth1,eth2 \  # 3 ISPs
  --multipath aggregate

# Warehouse - Simple gateway
quicether init --name warehouse-gateway --enterprise
quicether start \
  --advertise-subnet 10.2.0.0/16

# Sales team - Mobile clients
quicether init
quicether start --require-2fa
```

**Cost Comparison:**

| Item | Cisco Solution | QuicEther Solution |
|------|---------------|-------------------|
| VPN License | $50,000/year | $0 (open source) |
| SD-WAN License | $30,000/year | $0 (built-in) |
| Hardware | $20,000 (appliances) | $5,000 (commodity servers) |
| Professional Services | $10,000 (setup) | $5,000 (consultant) |
| Support | $15,000/year | $10,000/year (optional) |
| **Total (5 years)** | **$545,000** | **$60,000** |

**Savings: $485,000 over 5 years (89% reduction)**

**Success Criteria:**
- ✅ Passes security audit (SOC 2 compliant logging)
- ✅ Monitoring integration (Prometheus/Grafana)
- ✅ Policy enforcement (zero-trust access control)
- ✅ Performance meets or exceeds Cisco
- ✅ Support available (community or paid)

**Acceptance Test:**
David runs QuicEther for 90 days alongside Cisco solution. After 90 days, performance metrics show QuicEther has:
- 15% lower latency (direct site-to-site)
- 2.5× higher throughput (multi-path aggregation)
- Zero downtime incidents
- Passes security audit

David gets board approval to migrate fully to QuicEther, saving $485k over 5 years.

---

## Use Case Matrix

| Persona | Primary Use Case | Key Feature Needed | Success Metric |
|---------|-----------------|-------------------|----------------|
| Sarah (Developer) | Remote work | Auto-discovery, roaming | Setup < 5 min |
| James (Business) | Site-to-site | Easy setup, low cost | Cost < $50/mo |
| Alex (Homelab) | Mesh network | Zero-config, multipath | Zero manual config |
| Maria (Nomad) | Mobile + upload | Multipath aggregation | 2× upload speed |
| Priya (Rural) | Bandwidth aggregation | Multi-ISP bonding | Use all connections |
| David (Enterprise) | Replace VPN/SD-WAN | Policy, audit, scale | 80%+ cost reduction |

---

## Common Patterns Across Use Cases

### Pattern 1: Direct Connection (Preferred)
- Sarah, James, Maria (when both endpoints are reachable)
- Direct node-to-node, lowest latency, highest throughput

### Pattern 2: Gateway Forwarding (Fallback)
- Alex (home behind NAT, uses colo gateway)
- David (branch offices behind strict NAT)
- Gateway node forwards traffic when direct connection fails
- Gateway is user-deployed, not vendor infrastructure

### Pattern 3: Multipath Aggregation
- Priya (3 ISPs), James (2 ISPs), David (2 ISPs per office)
- Combine multiple network interfaces for higher throughput
- Automatic failover if one ISP goes down

---

## Anti-Personas (Who QuicEther Is NOT For)

### Anti-Persona 1: Grandma Who Wants to Watch Netflix

**Why NOT:**
- Needs one-click consumer VPN
- Doesn't want to understand networking
- Just wants to "unblock content"

**Better Solution:** NordVPN, ExpressVPN (consumer VPN services)

### Anti-Persona 2: Activist in Oppressive Regime

**Why NOT:**
- Needs anonymity (hiding who they are)
- Requires onion routing / mixing networks
- Threat model includes nation-state surveillance

**Better Solution:** Tor, I2P (anonymity networks)

### Anti-Persona 3: Enterprise with 10,000+ Employees

**Why NOT:**
- Requires enterprise support contracts
- Needs certified compliance (FedRAMP, etc.)
- Has budget for commercial solutions

**Better Solution:** Cisco, Palo Alto, Zscaler (enterprise vendors with support orgs)

---

## Success Scenarios (Acceptance Tests)

### Scenario 1: Sarah's First Day
```
Monday 9 AM: Sarah gets hired
Monday 9:30 AM: IT sends her laptop with QuicEther pre-installed
Monday 9:45 AM: Sarah opens laptop at home
Monday 9:46 AM: QuicEther auto-connects to company network
Monday 9:50 AM: Sarah accesses GitLab, starts working

Total time to productivity: 5 minutes
```

### Scenario 2: James's Camera Access
```
Saturday 10 PM: James is at home, hears alarm from main office
Saturday 10:01 PM: Opens phone, launches camera app
Saturday 10:02 PM: Sees live security camera feed
Saturday 10:03 PM: Confirms it's false alarm (cat triggered motion sensor)

Total time to verify: 3 minutes
```

### Scenario 3: Alex's New Device
```
Tuesday afternoon: Alex receives new Raspberry Pi
Tuesday 15:30: Flashes microSD with OS
Tuesday 15:35: Boots Pi, runs: curl -sSL install.quicether.net | sh
Tuesday 15:36: Pi appears in Alex's mesh topology
Tuesday 15:37: Alex SSHs into Pi from laptop

Total time to mesh: 2 minutes
```

### Scenario 4: Maria's Upload
```
Wednesday: Maria in Tokyo hotel with 200 GB footage
Wednesday 11 PM: Starts upload via hotel WiFi (50 Mbps)
Thursday 12 AM: Enables mobile hotspot (100 Mbps)
Thursday 12:01 AM: QuicEther detects second path, aggregates automatically
Thursday 3 AM: Upload completes (3 hours vs 4.5 hours single path)

Time saved: 1.5 hours (33% faster)
```

### Scenario 5: Priya's Storm
```
Friday afternoon: Working on client project, downloading textures (5 GB)
Friday 15:30: Download starts using all 4 connections (225 Mbps)
Friday 15:35: Storm rolls in, Starlink drops
Friday 15:35:30: QuicEther auto-rebalances to remaining 3 connections
Friday 15:36: Download continues at reduced speed (125 Mbps)
Friday 15:40: Download completes

User impact: Slight slowdown, no interruption, no manual intervention
```

---

## Requirements Derived from Personas

### Functional Requirements

1. **Auto-Discovery (All personas)**
   - Must work without manual IP configuration
   - Must discover peers within 30 seconds
   - Must work through NAT/firewall

2. **Connection Migration (Sarah, Maria)**
   - Must survive network changes
   - Must reconnect within 5 seconds
   - Must not drop active transfers

3. **Multi-Path Aggregation (Alex, Maria, Priya, David)**
   - Must bond 2-10 connections
   - Must achieve 80%+ of theoretical aggregate
   - Must handle heterogeneous paths (different ISPs, latencies)

4. **Policy Engine (James, David)**
   - Must support subnet advertisement
   - Must enforce access control
   - Must provide audit logging

5. **Mobile Support (Sarah, Maria)**
   - Must work on iOS/Android
   - Must be battery efficient
   - Must handle frequent network changes

### Non-Functional Requirements

1. **Performance**
   - Latency overhead: < 5ms
   - Throughput: Limited by hardware, not software
   - CPU usage: < 10% for 1 Gbps traffic

2. **Reliability**
   - Uptime: 99.9% (excluding network failures)
   - MTTR (Mean Time To Repair): < 1 minute
   - No data loss on path failure

3. **Usability**
   - Initial setup: < 10 minutes for technical users
   - Zero-config for most use cases
   - Web UI for monitoring

4. **Cost**
   - Per-user cost: $0 (no licensing)
   - Infrastructure cost: User-controlled
   - Target: 80%+ savings vs commercial VPN

5. **Security**
   - End-to-end encryption (TLS 1.3)
   - Zero-trust by default
   - Audit logging for compliance

---

## Summary

These personas and use cases drive our design:

**Core Users:**
1. **Remote workers** (Sarah) → Need auto-discovery, roaming
2. **Small businesses** (James) → Need low cost, easy setup
3. **Hobbyists** (Alex) → Need flexibility, zero-config
4. **Mobile users** (Maria) → Need multipath, resilience
5. **Rural users** (Priya) → Need ISP aggregation
6. **Enterprises** (David) → Need policy, scale, cost savings

**Key Insights:**
- Auto-discovery is non-negotiable
- Multi-path is the killer feature
- User-owned infrastructure enables cost savings
- Simple setup is critical for adoption

Next chapter: We'll define the core principles and philosophy that guide QuicEther's design based on these user needs.
