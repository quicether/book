# QuicEther Design Book (v2.0 — Post-httpf Revision)

## What This Is

This is the **design and architecture document** for QuicEther — a Layer 3 VPN built on native QUIC transport.

> **Post-httpf Revision (Feb 2026):** This book has been comprehensively revised
> to reflect lessons learned from building HTTP Fabric (httpf), a working L3 VPN
> that tunnels IP packets over HTTP/WebSocket. QuicEther ports httpf's validated
> architecture to native QUIC (RFC 9000), replacing HTTP/WebSocket with QUIC
> streams for better performance, native multipath, and connection migration.

It answers:
- **Why** are we building this? (And what httpf proved)
- **What** problems does it solve?
- **How** should it work? (Grounded in httpf's validated architecture)
- **Who** is it for?

## Reading Order

### Phase 1: Understanding (Chapters 1-3)
Start here to understand the problem space:

1. **[The Problem We're Solving](./01-the-problem.md)**
   - What are the fundamental issues?
   - Why do they matter?
   - What does success look like?

2. **[Existing Solutions & Their Limitations](./02-existing-solutions.md)**
   - What exists today?
   - Why isn't it good enough?
   - What can we learn from it?

3. **[User Stories & Use Cases](./03-user-stories.md)**
   - Who will use QuicEther?
   - What will they do with it?
   - What are the real-world scenarios?

### Phase 2: Solution Vision (Chapters 4-6)
Once you understand the problem, explore the solution:

4. **[Core Principles & Philosophy](./04-core-principles.md)**
   - What are our guiding principles?
   - What trade-offs do we make?
   - What do we explicitly NOT do?

5. **[High-Level Architecture](./05-architecture.md)**
   - What are the major components?
   - How do they fit together?
   - What are the interfaces between them?

6. **[Technology Choices & Trade-offs](./06-technology-choices.md)**
   - Why Rust? Why QUIC? Why server mesh?
   - What did we reject and why?
   - What httpf validated?

### Phase 3: Technical Deep-Dive (Chapters 7-11)
For implementers who need to understand the details:

7. **[Server Mesh & Discovery](./07-server-mesh-and-discovery.md)**
8. **[QUIC Transport & Multipath](./08-quic-and-multipath.md)**
9. **[Security Model & Zero-Trust](./09-security-and-zero-trust.md)**
10. **[VPN Interface & Routing](./10-vpn-interface-and-routing.md)**
11. **[Daemon & CLI Architecture](./11-daemon-and-cli.md)**

### Phase 4: Implementation Strategy (Chapters 12-15)
How to actually build this:

12. **[Implementation Roadmap](./12-implementation-roadmap.md)**
13. **[Testing & Validation Strategy](./13-testing-and-validation.md)**
14. **[Deployment & Operations](./14-deployment-and-operations.md)**
15. **[Configuration Reference](./15-configuration-reference.md)**

### Phase 5: Future Directions (Chapter 16)
Longer-term thinking and extensions:

16. **[Future Directions & Extensions](./16-future-directions-and-extensions.md)**

## Current Status

✅ **Chapter 0:** Introduction (Complete)  
✅ **Chapter 1:** The Problem We're Solving (Complete)  
✅ **Chapter 2:** Existing Solutions & Their Limitations (Complete)  
✅ **Chapter 3:** User Stories & Use Cases (Complete)  
✅ **Chapter 4:** Core Principles & Philosophy (Complete)  
✅ **Chapter 5:** High-Level Architecture (Complete)  
✅ **Chapter 6:** Technology Choices & Trade-offs (Complete)  
✅ **Chapter 7:** Server Mesh & Discovery (Complete)  
✅ **Chapter 8:** QUIC Transport & Multipath (Complete)  
✅ **Chapter 9:** Security Model & Zero-Trust (Complete)  
✅ **Chapter 10:** VPN Interface & Routing (Complete)  
✅ **Chapter 11:** Daemon & CLI Architecture (Complete)  
✅ **Chapter 12:** Implementation Roadmap (Complete)  
✅ **Chapter 13:** Testing & Validation Strategy (Complete)  
✅ **Chapter 14:** Deployment & Operations (Complete)  
✅ **Chapter 15:** Configuration Reference (Complete)  
✅ **Chapter 16:** Future Directions & Extensions (Complete)  

� **[Architecture Diagrams](./diagrams/architecture.md)** — Mermaid diagrams of system architecture

*All chapters revised to reflect httpf validation (Feb 2026).*

## How to Use This Book

### For Decision Makers
- Read Chapters 1-2 to understand the problem
- Read Chapter 4 for core principles
- Skim Chapter 5 for architecture overview
- Review Chapter 14 for deployment models

### For Architects
- Read everything in Phase 1-2 carefully
- Deep dive into Phase 3 for technical details
- Reference Phase 4 for implementation strategy

### For Developers
- Understand Phase 1-2 first (the "why")
- Study Phase 3 in detail (the "what")
- Follow Phase 4 for building (the "how")
- Keep Phase 5 handy for operational concerns

### For Operators/SREs
- Skim Phase 1-2 for context
- Read Chapter 5 for system understanding
- Focus on Phase 5 for operational details

## Philosophy

This book follows these principles:

1. **First Principles Thinking**
   - Start from fundamental truths
   - Question all assumptions
   - Don't cargo-cult solutions

2. **Explicit Trade-offs**
   - Every decision has costs and benefits
   - Document what we rejected and why
   - Admit what we don't know

3. **User-Centric Design**
   - Real user needs drive features
   - Simplicity over feature creep
   - "Good enough" over "perfect"

4. **Pragmatic Engineering**
   - Leverage existing technology where appropriate
   - Build new only when necessary
   - Ship working software, iterate based on feedback

## Contributing

As development progresses, this book should evolve:

### Adding New Chapters
When designing a new major component:
1. Write the chapter FIRST (before code)
2. Get review and consensus
3. Update chapter as you learn during implementation

### Updating Existing Chapters
When you discover something that invalidates prior thinking:
1. Document the new learning
2. Update the affected chapter
3. Keep a "Decision Log" section showing what changed and why

### Decision Logs
At the end of key chapters, maintain a decision log:

```markdown
## Decision Log

### 2025-11-20: Chose QUIC over WireGuard
**Reason:** QUIC has native multipath support (draft standard)
**Trade-off:** Less mature than WireGuard
**Risk:** Multipath spec may change before finalization

### 2025-11-25: Rust over Go
**Reason:** Memory safety + performance critical for data plane
**Trade-off:** Slower development initially
**Risk:** Smaller ecosystem than Go for networking
```

## Living Document

This book is **not set in stone**. As we:
- Prototype features
- Talk to users
- Discover technical challenges
- Learn from the community

We should update this book to reflect our evolving understanding.

**Rule:** Major architectural decisions should be documented here BEFORE implementing in code.

## Next Steps

With the core chapters (0–16) written, the next steps are:
- Refine chapters based on early prototyping and feedback
- Add decision logs as major choices are validated or revised
- Potentially add new chapters for advanced topics as they become concrete

## Questions or Feedback?

This book is meant to be discussed and refined. If you find:
- Unclear explanations
- Missing context
- Questionable decisions
- Better alternatives

Please open an issue or PR to discuss!

---

**Remember:** Code is temporary, architecture decisions are permanent (or at least very expensive to change). Invest time here to save months of rework later.
