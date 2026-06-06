# Architecture Diagrams: AI Mesh Network Platform

This document provides high-quality Mermaid diagrams visualizing the core architecture, flows, and protocols.

## System Architecture

```mermaid
graph TB
    subgraph Application["Application Layer"]
        AppA["Robot Control<br/>Logic"]
        AppB["Inference<br/>Coordinator"]
        AppC["Event<br/>Handler"]
    end

    subgraph Coordination["Coordination Engine"]
        DelegateTask["Task Delegation<br/>Engine"]
        IntentMatch["Intent<br/>Matcher"]
        Scheduler["Task<br/>Scheduler"]
    end

    subgraph Trust["Trust & Security"]
        TrustMgr["Trust<br/>Manager"]
        AuthMgr["Authentication<br/>Manager"]
        RevokeList["Revocation<br/>List"]
    end

    subgraph Capability["Capability & Context"]
        CapReg["Capability<br/>Registry"]
        CapMatcher["Capability<br/>Matcher"]
        ContextSync["Context<br/>Synchronization"]
    end

    subgraph Discovery["Discovery & Registration"]
        PeerReg["Peer<br/>Registry"]
        AdvertiseMgr["Advertisement<br/>Manager"]
        LivenessTracker["Liveness<br/>Tracker"]
    end

    subgraph Transport["Transport Abstraction"]
        BLETransport["BLE<br/>Transport"]
        WiFiTransport["WiFi<br/>Transport<br/>Planned"]
        ThreadTransport["802.15.4<br/>Transport<br/>Planned"]
    end

    subgraph Observability["Observability & Diagnostics"]
        Logger["Event<br/>Logger"]
        Metrics["Metrics<br/>Collector"]
        Tracer["Distributed<br/>Tracer"]
    end

    Application -->|delegateTask| Coordination
    Coordination -->|query capabilities| Capability
    Coordination -->|route task to peer| Trust
    Trust -->|get peer info| Discovery
    Discovery -->|scan/advertise| Transport
    Coordination -->|emit events| Observability
    Trust -->|emit events| Observability
    Discovery -->|emit events| Observability

    style Application fill:#e1f5ff
    style Coordination fill:#f3e5f5
    style Trust fill:#fff3e0
    style Capability fill:#e8f5e9
    style Discovery fill:#fce4ec
    style Transport fill:#f1f8e9
    style Observability fill:#eceff1
```

## Peer Discovery Flow

```mermaid
sequenceDiagram
    participant "Peer A" as PeerA
    participant "Mesh" as Mesh
    participant "Peer B" as PeerB
    participant "Registry" as Reg

    Note over PeerA,PeerB: Startup Phase
    PeerA->>PeerA: Initialize Runtime
    PeerA->>Mesh: Start advertising "peer-A"
    PeerA->>Mesh: Start scanning for peers

    Note over PeerA,PeerB: Discovery
    PeerB->>Mesh: Advertise "peer-B"<br/>(capabilities, metadata)
    Mesh->>PeerA: Receive advertisement (peer-B)
    
    PeerA->>Reg: Register peer-B
    Reg->>Reg: Store metadata<br/>(node ID, device type,<br/>capabilities, TTL)
    
    Note over PeerA,PeerB: Active Mesh
    Note over PeerA: Peer B is now<br/>available for coordination

    Note over PeerA,PeerB: Liveness Tracking
    loop Every 30s (configurable)
        PeerA->>Reg: Check peer-B TTL
        alt TTL expired
            Reg->>Reg: Mark peer-B inactive
            PeerA->>PeerA: Emit peer-left event
        else TTL valid
            PeerA->>Mesh: Re-scan (refresh discovery)
            Mesh->>PeerA: peer-B still advertising
            Reg->>Reg: Reset peer-B TTL
        end
    end
```

## Trust Handshake Protocol

```mermaid
sequenceDiagram
    participant "Peer A" as PeerA
    participant "Trust Manager A" as TrustA
    participant "Transport" as Transport
    participant "Trust Manager B" as TrustB
    participant "Peer B" as PeerB

    Note over PeerA,PeerB: Trust Establishment
    
    PeerA->>TrustA: initiateTrust(peer-B)
    TrustA->>TrustA: Set state: VERIFYING
    TrustA->>Transport: Send TrustInitiation
    Transport->>TrustB: Receive TrustInitiation

    TrustB->>TrustB: Generate challenge<br/>(random bytes)
    TrustB->>Transport: Send Challenge
    Transport->>TrustA: Receive Challenge

    TrustA->>TrustA: Solve challenge<br/>(compute response)
    TrustA->>Transport: Send ChallengeResponse
    Transport->>TrustB: Receive ChallengeResponse

    TrustB->>TrustB: Verify response
    alt Verification succeeds
        TrustB->>TrustB: Set state: TRUSTED
        TrustB->>Transport: Send TrustEstablished
        Transport->>TrustA: Receive TrustEstablished
        TrustA->>TrustA: Set state: TRUSTED
        Note over PeerA,PeerB: Trust established<br/>Both peers are TRUSTED
    else Verification fails
        TrustB->>Transport: Send TrustFailed
        Transport->>TrustA: Receive TrustFailed
        TrustA->>TrustA: Set state: UNTRUSTED
        Note over PeerA,PeerB: Trust failed<br/>Cannot execute tasks
    end
```

## Task Delegation Flow

```mermaid
sequenceDiagram
    participant "App" as App
    participant "Coordination" as Coord
    participant "CapabilityRegistry" as CapReg
    participant "PeerRegistry" as PeerReg
    participant "RemotePeer" as RemotePeer

    Note over App,RemotePeer: Task Initiation
    App->>Coord: delegateTask(intent)
    Coord->>Coord: Create task ID

    Note over App,RemotePeer: Capability Matching
    Coord->>CapReg: query peers with capability
    CapReg->>CapReg: Match intent action<br/>to capabilities
    CapReg->>Coord: Return: [peer-B, peer-C]

    Note over App,RemotePeer: Peer Selection
    Coord->>PeerReg: Get peer details
    PeerReg->>Coord: Return: status, load,<br/>trust state
    Coord->>Coord: Select best peer<br/>(peer-B: trusted, light load)

    Note over App,RemotePeer: Task Transmission
    Coord->>RemotePeer: Send TaskMessage<br/>(task ID, intent, timeout)
    RemotePeer->>RemotePeer: Receive task
    RemotePeer->>RemotePeer: Execute intent
    RemotePeer->>RemotePeer: Compute result

    Note over App,RemotePeer: Result Collection
    RemotePeer->>Coord: Send TaskResult<br/>(task ID, result/error)
    Coord->>Coord: Receive result<br/>Update metrics
    Coord->>App: Return TaskResult

    Note over App,RemotePeer: Completion
    App->>App: Process result
```

## Capability Negotiation Flow

```mermaid
sequenceDiagram
    participant "Peer A" as A
    participant "Transport" as T
    participant "Peer B" as B
    participant "Cap Registry B" as RegB

    Note over A,B: Initial State
    Note over A: Has no info<br/>about B
    
    Note over A,B: Capability Query
    A->>T: Send CapabilityQuery<br/>to peer-B
    T->>B: Deliver query
    B->>RegB: Get local capabilities
    RegB->>RegB: Gather all capabilities<br/>(motion, vision, inference)

    Note over A,B: Capability Response
    B->>T: Send CapabilityAdvertisement<br/>[motion-v2.1, vision-v1.0]
    T->>A: Deliver advertisement

    Note over A,B: Registry Update
    A->>A: Store capabilities<br/>peer-B: [motion, vision]
    A->>A: Emit capability-updated<br/>event

    Note over A,B: Matching
    loop For each new task
        A->>A: Need action "move"
        A->>A: Query registry<br/>for "motion" capability
        A->>A: Find peer-B matches
        A->>A: Execute task on peer-B
    end
```

## Complete Task Execution Flow (End-to-End)

```mermaid
graph TD
    Start["App: delegateTask<br/>(action: 'process-image'<br/>params: {...})"] 
    
    Start --> CreateIntent["1. Create Intent<br/>action: 'process-image'<br/>requiredCapability: 'image-processor'"]
    
    CreateIntent --> QueryCap["2. Query Registry<br/>Which peers have<br/>'image-processor'?"]
    
    QueryCap --> GetPeers["3. Get Peer Details<br/>Peer A: v1.0, trusted, CPU 20%<br/>Peer B: v2.0, untrusted, CPU 80%"]
    
    GetPeers --> Select["4. Select Best Peer<br/>Choose Peer A<br/>(better version, trusted,<br/>lower load)"]
    
    Select --> SerializeIntent["5. Serialize Intent<br/>Binary format<br/>+ metadata"]
    
    SerializeIntent --> SendTask["6. Send Task Message<br/>via BLE<br/>include: task ID,<br/>timeout (30s)"]
    
    SendTask --> RemoteReceive["7. Remote Execution<br/>Peer A receives<br/>Deserializes intent<br/>Executes task"]
    
    RemoteReceive --> Compute["8. Computation<br/>Process image<br/>Generate result"]
    
    Compute --> SendResult["9. Send Result<br/>task ID, result data<br/>execution time"]
    
    SendResult --> ReceiveResult["10. Result Reception<br/>Coordination engine<br/>collects result"]
    
    ReceiveResult --> UpdateMetrics["11. Update Metrics<br/>task_latency: 245ms<br/>task_success: 1"]
    
    UpdateMetrics --> ReturnToApp["12. Return to App<br/>TaskResult object<br/>status, result, time"]
    
    ReturnToApp --> End["App processes<br/>result"]
    
    style Start fill:#e3f2fd
    style End fill:#c8e6c9
    style Select fill:#fff9c4
    style Compute fill:#ffe0b2
    style UpdateMetrics fill:#f0f4c3
```

## Context Synchronization Flow

```mermaid
sequenceDiagram
    participant "Peer A" as A
    participant "ContextManager" as CtxMgr
    participant "Transport" as T
    participant "Peer B" as B
    participant "LocalStateB" as StateB

    Note over A,B: Initial State
    A->>A: State S = {location: (10,20),<br/>battery: 85%, status: idle}

    Note over A,B: State Change
    A->>A: Update: battery: 75%
    A->>CtxMgr: onChange() event

    Note over A,B: Context Exchange
    CtxMgr->>CtxMgr: Create delta<br/>{battery: 75%}
    CtxMgr->>CtxMgr: Compress delta<br/>(bandwidth efficient)
    CtxMgr->>T: Send ContextUpdate
    
    Note over A,B: Remote Application
    T->>B: Deliver ContextUpdate
    B->>StateB: Merge delta<br/>State = {...,<br/>battery: 75%}
    B->>B: Emit state-changed event
    B->>B: Notify listeners

    Note over A,B: Eventual Consistency
    Note over A,B: All peers converge<br/>on same state<br/>time: <100ms
```

## Discovery and Registration State Machine

```mermaid
stateDiagram-v2
    [*] --> UNKNOWN

    UNKNOWN --> DISCOVERING: First advertisement<br/>from peer

    DISCOVERING --> ACTIVE: Full metadata<br/>received

    ACTIVE --> ACTIVE: Re-advertisement<br/>received, TTL reset

    ACTIVE --> INACTIVE: TTL expired<br/>No advertisement<br/>for 30 seconds

    INACTIVE --> ACTIVE: Peer re-advertises<br/>TTL reset

    INACTIVE --> STALE: TTL expired<br/>No advertisement<br/>for 60 seconds

    STALE --> [*]: Cleaned up from<br/>registry

    ACTIVE --> [*]: Explicit removal
    INACTIVE --> [*]: Explicit removal
```

## Trust State Machine

```mermaid
stateDiagram-v2
    [*] --> UNKNOWN

    UNKNOWN --> VERIFYING: initiateTrust()<br/>called

    VERIFYING --> TRUSTED: Challenge<br/>verification<br/>succeeds

    VERIFYING --> UNTRUSTED: Challenge<br/>verification<br/>fails

    TRUSTED --> UNTRUSTED: Peer fails<br/>verification

    UNTRUSTED --> TRUSTED: Retry trust<br/>succeeds

    TRUSTED --> REVOKED: revokeTrust()<br/>called

    UNTRUSTED --> REVOKED: revokeTrust()<br/>called

    REVOKED --> [*]: Cleanup
```

## Multi-Transport Architecture (Phase 7 Vision)

```mermaid
graph TB
    subgraph Application["Application Layer"]
        App["Coordination<br/>Engine"]
    end

    subgraph Abstraction["Transport Abstraction"]
        Selector["Transport<br/>Selector"]
        Fallback["Fallback<br/>Manager"]
    end

    subgraph Transports["Transport Implementations"]
        BLE["BLE<br/>⚠ ~100m range<br/>✓ Low power<br/>✓ Phone compatible"]
        WiFi["WiFi Direct<br/>⚠ ~150m range<br/>✓ High bandwidth<br/>✓ Laptop compatible"]
        Thread["802.15.4<br/>⚠ ~100m range<br/>✓ Low power<br/>✓ Standard mesh"]
        LoRa["LoRaWAN<br/>⚠ ~10km range<br/>⚠ Low bandwidth<br/>✓ Long distance"]
    end

    subgraph Protocol["Bridging & Relay"]
        Bridge["Multi-Transport<br/>Bridge"]
        Relay["Capability<br/>Relay"]
    end

    App --> Selector
    Selector --> Fallback
    
    Fallback --> BLE
    Fallback --> WiFi
    Fallback --> Thread
    Fallback --> LoRa
    
    BLE --> Bridge
    WiFi --> Bridge
    Thread --> Bridge
    LoRa --> Bridge
    
    Bridge --> Relay

    style BLE fill:#e3f2fd
    style WiFi fill:#e3f2fd
    style Thread fill:#f3e5f5
    style LoRa fill:#fce4ec
```

## Performance Characteristics Timeline

```mermaid
graph LR
    T0["T=0s<br/>Start<br/>Runtime"] 
    T1["T=2.5s<br/>First Peer<br/>Detected"]
    T2["T=10s<br/>Mesh<br/>Converged"]
    T3["T=10.1s<br/>Task 1<br/>Delegated"]
    T4["T=10.3s<br/>Task 1<br/>Completed"]
    T5["T=15s<br/>Peer<br/>Leaves"]
    T6["T=60s<br/>Peer<br/>Timeout"]

    T0 -->|Discovery| T1
    T1 -->|Network| T2
    T2 -->|Query| T3
    T3 -->|Execution| T4
    T4 -->|Running| T5
    T5 -->|Liveness<br/>tracking| T6

    style T0 fill:#e1f5fe
    style T1 fill:#b3e5fc
    style T2 fill:#81d4fa
    style T3 fill:#4fc3f7
    style T4 fill:#29b6f6
    style T5 fill:#fff9c4
    style T6 fill:#ffecb3
```

## Message Flow for Multi-Hop Task (Phase 4+)

```mermaid
sequenceDiagram
    participant "Local" as Local
    participant "Peer A" as A
    participant "Peer B" as B
    participant "Peer C" as C

    Note over Local,C: Complex Task Requires Relay
    
    Local->>A: Task: "process<br/>then filter"
    A->>A: Process phase complete<br/>Generate intermediate result
    
    A->>B: Forward task + result
    B->>B: Filter phase<br/>using A's result
    B->>B: Generate new intermediate
    
    B->>C: Forward task + result
    C->>C: Final transformation
    C->>C: Generate final result
    
    C->>A: Send result back
    A->>Local: Collect final result
    
    Local->>Local: All phases<br/>complete
    
    Note over Local,C: Total time: ~500ms<br/>Latency: acceptable
```

---

## Rendering Notes

These Mermaid diagrams are designed to render in:
- GitHub markdown
- GitLab markdown
- Notion
- Any Mermaid-compatible viewer

For local development, use [mermaid.live](https://mermaid.live) to view and edit.

---

**Last Updated**: June 2026 | **Version**: 0.1.0-alpha
