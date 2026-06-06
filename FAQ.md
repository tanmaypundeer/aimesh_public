# FAQ: AI Mesh Network Platform

Common questions and answers about the project, architecture, and use cases.

## Architecture & Design

### Why not just use cloud orchestration?

Cloud orchestration (send data to cloud, get decisions back) has fundamental limitations for edge:

1. **Latency**: ~50ms round-trip adds too much delay for safety-critical scenarios (autonomous vehicles, drones)
2. **Connectivity**: Works offline? No. Works in mines, underground, rural areas? No.
3. **Bandwidth**: Uploading raw sensor data (video, LiDAR) is expensive and impractical
4. **Privacy**: Many deployments (healthcare, military) cannot send data to external services
5. **Cost**: Per-device cloud services scale poorly for millions of edge devices
6. **Resilience**: Single point of failure – if cloud is down, devices cannot coordinate

Cloud still excels at: global state, persistent storage, complex business logic, long-term analytics.

**Answer**: Use local mesh for **time-critical, local coordination**; cloud for **global state and analytics**.

### Why Bluetooth Low Energy?

We chose BLE for Phase 1 because:

1. **Ubiquitous**: Every smartphone, laptop, and modern IoT device has BLE
2. **Standardized**: Mature specification, wide tooling support
3. **Low power**: Suitable for battery-powered devices (drones, wearables)
4. **Established**: Existing libraries, well-understood behavior

**Planned support** (Phase 7): WiFi Direct, 802.15.4 (Thread), LoRaWAN

**Hybrid deployments**: Mix BLE on mobile devices with 802.15.4 on static infrastructure.

### Why not Bluetooth Mesh?

Good question. They solve different problems:

| Aspect | Bluetooth Mesh | AI Mesh |
|--------|---|---|
| **Scope** | Network relay and flooding | Semantic task coordination |
| **Use case** | "Relay this packet through the mesh" | "Find a peer with video processing capability and execute this task" |
| **State** | Minimal (relay tables) | Rich (capabilities, trust, context) |
| **Latency** | Optimized for low-latency flooding | Optimized for semantic matching |
| **Complexity** | Medium (mesh flooding) | Medium (distributed coordination) |

**Relationship**: AI Mesh **could use** Bluetooth Mesh as a transport layer. Currently we use BLE (simpler, more compatible). We're transport-agnostic by design.

### Is this a multi-agent system?

No, not exactly. Here's the distinction:

**Multi-Agent Systems** typically include:
- Agent behavior models and decision logic
- Agent beliefs and planning
- Agent-to-agent communication protocols
- Agent resource management

**AI Mesh** provides:
- Infrastructure for agents to coordinate
- Capability-aware task delegation
- Trust management
- Context sharing

**Relationship**: Multi-agent systems **run on top of** AI Mesh. Each agent has its own AI/planning; the mesh enables them to collaborate.

**Example**: "Agent A and B are robots. Each has its own control logic. AI Mesh lets them discover each other and delegate tasks like 'carry this object together'."

### Is this a local AI platform?

No. This is **not**:
- A framework for running ML models
- An inference engine
- A training platform
- A data storage system

This **is**:
- A coordination runtime that enables systems with local AI to collaborate
- A distribution mechanism for tasks (not code, not models)

**Example**: "Each robot runs its own computer vision. AI Mesh helps them discover this fact and delegate frames to peers with better models."

### Is this open source?

Yes. Apache License 2.0. All code is open source. All documentation is public.

**Governance**: Currently maintained by the project founders. We welcome community contributions, feature proposals, and real-world deployment feedback.

### How is this different from Kubernetes for edge?

Kubernetes is for **datacenter-scale orchestration** of containerized workloads. Designed for:
- Stable, large clusters (100s-1000s of nodes)
- Persistent state and storage
- Centralized control plane
- Complex resource management

AI Mesh is for **ephemeral edge meshes**. Designed for:
- Small, dynamic groups (5-20 peers typically)
- Ephemeral state and local coordination
- Decentralized peer-to-peer operation
- Semantic task coordination

**Analogy**: Kubernetes is like air traffic control. AI Mesh is like pilots communicating directly on the tarmac.

### How is this different from RPC/gRPC?

RPC systems enable method calls across a network. AI Mesh is higher-level:

| Aspect | RPC/gRPC | AI Mesh |
|---|---|---|
| **Abstraction** | Remote functions | Semantic intents |
| **Discovery** | Explicit (registry, config) | Automatic (capability-based) |
| **Matching** | Manual routing | Automatic peer selection |
| **Semantics** | Arbitrary functions | Domain-specific intents |

**Relationship**: AI Mesh **could use** gRPC as a transport for data exchange. We're transport-agnostic.

## Use Cases & Applicability

### Is this suitable for robotics?

Yes. Strong use cases:

- **Multi-robot collaboration**: Teams of robots discovering each other and coordinating tasks
- **Robot + Infrastructure**: Robots coordinating with stationary infrastructure (camera array, computing edge server)
- **Human-Robot interaction**: Wearables coordinating with robots for safety-critical scenarios

See [robotics examples](examples/robotics/) for sample code.

### Can this handle autonomous vehicles?

Partially. Good for:
- **Local coordination**: Vehicles coordinating with nearby infrastructure and other vehicles
- **Sub-100ms decisions**: Emergency maneuvers, collision avoidance
- **Offline operation**: Coordination works without cloud connectivity

Not a fit for:
- **Global navigation**: Route planning needs global map
- **Cloud integration**: Long-term behavior, analytics need persistent state

**Typical deployment**: Hybrid – local mesh for real-time coordination, cloud for global planning and analytics.

### Is this suitable for IoT?

Yes, but it depends on the scale:

**Good fit**:
- 5-20 devices coordinating locally (e.g., smart home, warehouse sensors)
- Devices need to discover each other and coordinate
- Offline operation is acceptable

**Poor fit**:
- 1000s of devices (IoT platforms like AWS IoT are better)
- Need global state consistency
- Persistent cloud storage required

### Can I use this for edge AI inference?

Yes. Example workflow:

1. Edge device receives inference request
2. Queries mesh: "Which peers have this model?"
3. Sends frame to peer with best GPU
4. Gets result back
5. Integrates into application logic

See [inference distribution example](examples/inference-distribution/).

### Can this work offline?

Yes, by design. Core coordination works entirely locally. Optional cloud integration happens only when connection is available.

**Offline capabilities**:
- Peer discovery
- Task coordination
- Capability negotiation
- Trust management
- Context synchronization

**Cloud integration** (optional, Phase 4+):
- Analytics upload when connection restored
- Global state synchronization
- Device enrollment and management

### Can this work in intermittent connectivity scenarios?

Yes. Designed for unreliable links.

**Resilience mechanisms**:
- Tasks have timeouts; failure is handled gracefully
- Peers can retry or delegate to alternate capability
- No assumptions about link stability

### Is this suitable for real-time applications?

Partially. Depends on your definition of real-time:

| Latency Requirement | Suitable? | Notes |
|---|---|---|
| <10ms | No | Too much overhead in typical deployments |
| 10-100ms | Yes | Typical use case; achievable in local mesh |
| 100-500ms | Yes | Easy; high success rate |
| >500ms | Yes | Not a constraint |

**Example**: Autonomous vehicle emergency stopping (<10ms) – difficult. Traffic coordination (100ms) – easy.

## Technical Questions

### How many peers can one mesh support?

**Optimal**: 5-20 peers

**Current testing**: Validated up to 5 peers; performance not tested beyond 20

**Scaling limits**:
- **Peer discovery**: O(n) scan messages; discovery time ~5-10s for n peers
- **Capability matching**: O(n) registry lookup; <10ms for any size
- **Task routing**: Single-hop default; O(1) latency
- **Memory**: ~5-10 KB per peer

**For 100+ peers**: Use hierarchical clustering (Phase 7) or multiple meshes with bridges.

### What is typical task execution latency?

**Components**:
- Peer matching: <10ms
- Message transmission (BLE): 50-200ms
- Remote execution: variable (depends on task)
- Result transmission: 50-200ms
- **Total median**: 100-500ms

**P99 latency**: ~2-5 seconds (depends on radio interference, peer load)

### How much bandwidth does AI Mesh use?

**Components**:

| Activity | Bandwidth | Frequency |
|---|---|---|
| Discovery (advertisement) | ~100 bytes | Every 5s (configurable) |
| Peer discovery (scan) | ~10 bytes | Every 5s |
| Task delegation | 100-1000 bytes | Per task |
| Result return | 100-1000 bytes | Per task |
| Context sync | 10-100 bytes | Per update |
| **Total typical** | 1-10 KB/s | Depends on activity |

**Low-bandwidth mode**: Increase discovery interval to 30s, disable context sync → 100 bytes/s

### What are the memory requirements?

**Per-node baseline**: ~5 MB (runtime)

**Per-peer overhead**: 5-10 KB (registry, trust state, capabilities)

**With 20 peers**: ~5.2 MB

**Total task queue**: Depends on application; typically <1 MB

**Recommendation**: Works on devices with 64+ MB RAM. Suitable for most phones, edge servers, and IoT devices.

### Can tasks be executed in parallel?

Yes. Each peer can execute multiple tasks concurrently.

**Coordination Engine**: Spawns task in separate async context

**Parallelism limits**: Depends on device hardware and application logic

**Example**: Robot can execute multiple delegated tasks concurrently while running its own control loop

### Can I chain tasks across multiple peers?

**Phase 1-3**: No (single-hop only)

**Phase 4+**: Yes (multi-hop routing with relay support planned)

**Example (Phase 4)**: Task: "Process frame → filter → classify"
- Peer A: Processing capability (GPU available)
- Peer B: Filtering capability (algorithm optimized)
- Peer C: Classification capability (trained model)
- Mesh: Routes task A → B → C → result

### How is data serialized?

**Format**: Custom binary codec optimized for bandwidth efficiency

**Details**:
- Variable-length encoding for integers
- LEB128 encoding
- String compression (for repeated capability names)
- Capability hash instead of full definition (reduces size)

**JSON support**: Optional (Phase 2), for debugging/compatibility

### Is there encryption?

**Phase 1-3**: No encryption. Challenge-response trust only.

**Phase 6**: End-to-end encryption planned. All messages will be encrypted.

**Until then**: Deploy in trusted networks or use VPN/secure transport.

## Operational Questions

### How do I deploy this?

**Development**:
```bash
npm install aimesh-platform
npm start
```

**Production** (Phase 10): Deployment guides, Docker containers, Kubernetes charts planned.

### How do I debug issues?

**Observability hooks**:
- Event logging: All major operations logged
- Metrics: Task latency, peer count, success rates
- Debug interface: Inspect peer registry, trust state, capabilities

**CLI tool** (Phase 9): Inspect mesh state, debug peer connectivity, test capabilities

### How do I monitor health?

**Metrics available** (Phase 2+):
- Peer count (active, inactive, stale)
- Task success rate (success/failure/timeout)
- Task latency (p50, p99)
- Discovery time
- Message loss rate (estimated)

**Alerting** (Phase 10): Integration with standard monitoring tools (Prometheus, DataDog)

### Can I update code in a running mesh?

No. All devices must be restarted with new code.

**Planned** (Phase 9+): Rolling updates with configuration management

### What happens when a peer crashes?

**Graceful degradation**:
1. Peer stops advertising
2. Registry marks it inactive after ~30s
3. Tasks delegated to it timeout (~30s)
4. Peer is removed from registry after ~60s total
5. Other peers handle tasks

**Result**: Temporary performance degradation; no mesh-wide failure

### Can I run multiple meshes simultaneously?

Yes. Each mesh is independent.

**Configuration**: Different transport channels or frequency bands

**Interoperability** (Phase 7+): Meshes can communicate via bridges

## Support & Community

### Where do I report bugs?

[GitHub Issues](https://github.com/aimesh/platform/issues)

**Please include**:
- OS and device type
- Mesh size (number of peers)
- Steps to reproduce
- Error messages and logs

### Where can I ask questions?

[GitHub Discussions](https://github.com/aimesh/platform/discussions)

### How do I contribute?

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Is there a community forum?

Currently: GitHub Discussions

**Planned**: Community Slack or Discord (Phase 9)

### Who maintains this?

Currently: Founding team + early community contributors

**Long-term**: Open governance model planned (Phase 9)

---

**Last Updated**: June 2026 | **Version**: 0.1.0-alpha
