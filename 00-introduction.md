# QuicEther Design Book

## Introduction

This book documents the design, architecture, and implementation strategy for QuicEther — a **Layer 2 VPN built on native QUIC transport**, inspired by the SoftEther architecture.

QuicEther evolves from the lessons learned building **HTTP Fabric (httpf)**, an experimental VPN that tunneled IP packets over HTTP/WebSocket. httpf validated the core architecture — hub-based multi-tenancy, session management, multi-connection aggregation, firewall and policy engines, server mesh networking, cascade routing, mobile FFI, and the full authentication model. QuicEther takes everything that worked in httpf and rebuilds it on **QUIC** (RFC 9000), but makes a fundamental architectural shift: from **Layer 3 (IP routing)** to **Layer 2 (Ethernet switching)**.

### Why Layer 2?

The name "QuicEther" literally means "QUIC + Ethernet." An L2 VPN operates at the Ethernet frame level, creating a **virtual Ethernet segment** that spans physical networks. This means:

- **TAP devices** instead of TUN — the VPN interface is a virtual Ethernet adapter
- **Virtual Hub** acts as an Ethernet switch with MAC address learning, not an IP router
- **Physical routers handle DHCP, ARP, DNS** — the VPN doesn't need to reinvent these
- **Full L2 transparency** — broadcast, multicast, ARP, DHCP all pass through naturally
- **Bridge mode** connects the Virtual Hub to a physical LAN, extending it across the internet

This is the same foundational approach that made SoftEther VPN one of the most versatile VPN solutions ever built. Combined with QUIC's native multipath, connection migration, and TLS 1.3, QuicEther aims to be a modern, high-performance successor.

---

## What This Book Contains

### Part I: Problem Space
- **Chapter 1:** The Problem We're Solving
- **Chapter 2:** Existing Solutions & Their Limitations
- **Chapter 3:** User Stories & Use Cases

### Part II: Solution Vision
- **Chapter 4:** Core Principles & Philosophy
- **Chapter 5:** High-Level Architecture
- **Chapter 6:** Technology Choices & Trade-offs

### Part III: Technical Deep-Dive
- **Chapter 7:** Server Mesh & Discovery
- **Chapter 8:** QUIC Transport & Multipath
- **Chapter 9:** Security & Zero-Trust
- **Chapter 10:** Virtual Hub, TAP & Bridging
- **Chapter 11:** Daemon & CLI Architecture

### Part IV: Implementation Strategy
- **Chapter 12:** Implementation Roadmap
- **Chapter 13:** Testing & Validation Strategy
- **Chapter 14:** Deployment & Operations
- **Chapter 15:** Configuration Reference

### Part V: Future Directions
- **Chapter 16:** Future Directions & Extensions

---

## How to Read This Book

**For Decision Makers:**
- Read Part I (Problem Space) and Part II (Solution Vision)
- Skim Part III for technical feasibility
- Review Chapter 14 for deployment patterns

**For Architects:**
- Focus on Part II (Solution Vision) and Part III (Technical Deep-Dive)
- Study Chapter 6 (Technology Choices) carefully
- Reference Part IV for implementation strategy

**For Developers:**
- Read everything, but especially Part III and Part IV
- Use this as the "source of truth" for design decisions
- Refer back when making implementation choices

**For Operators:**
- Skim Part I-II for context
- Read Chapter 5 for system understanding
- Focus on Chapters 14-15 for deployment and configuration

---

## Document Status

**Version:** 2.0 (Post-httpf Revision)  
**Date:** February 27, 2026  
**Status:** Living Document — revised based on httpf implementation experience  
**Previous Version:** v1.0 (Pre-Development, November 2025)

---

## What Changed Since v1.0

The original book was written before any code existed, envisioning a pure peer-to-peer system with Kademlia DHT as the foundation. After building httpf end-to-end, major revisions reflect proven reality:

| Aspect | v1.0 (Theory) | v2.0 (Post-httpf) |
|--------|---------------|-------------------|
| **Topology** | Pure P2P, every node equal | Client-server with Virtual Hubs; server mesh for distribution |
| **Network Layer** | L3 (IP/TUN) | L2 (Ethernet/TAP) — SoftEther-inspired virtual switching |
| **Discovery** | Kademlia DHT from day 1 | Server-based; DHT deferred to future versions |
| **Protocol** | Conceptual QUIC encapsulation | Proven FrameBatch framing with LZ4 + ChaCha20 |
| **Auth** | TLS 1.3 mutual auth only | Three-tier: Ed25519 identity + password + service tokens |
| **Hashing** | SHA-1 for NodeId | BLAKE3 for all hashing |
| **Server features** | Minimal routing | Virtual Hub (L2 switch), SecureNAT (optional), firewall, policy, audit |
| **DHCP/ARP** | Server provides virtual NAT | Physical router handles DHCP/ARP; SecureNAT as fallback |
| **Networking** | Direct P2P only | Server mesh + cascade connections between hubs |
| **Mobile** | Not mentioned | FFI crate with iOS/Android support |
| **Blockchain** | Optional future feature | Removed entirely |

---

## Key Principles

This book is written with these principles:

1. **Implementation-Informed Design**
   - Every architectural decision has been validated in httpf
   - Theory is grounded in working code
   - Proven patterns are preserved; unproven ideas are deferred

2. **User-Centric Design**
   - Every decision serves a real user need
   - No features "for the sake of it"
   - Simplicity over complexity

3. **Pragmatic Engineering**
   - Acknowledge trade-offs explicitly
   - Choose "good enough" over "perfect"
   - Ship working software, iterate based on feedback

4. **Transparent Documentation**
   - Explain the "why" behind every decision
   - Document what we rejected and why
   - Include lessons learned from httpf

---

## Let's Begin

The journey to building QuicEther starts with understanding the problem deeply. Turn to Chapter 1 to begin.
