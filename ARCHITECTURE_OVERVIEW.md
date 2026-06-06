# Architecture Overview: AI Mesh Network Platform

## System Architecture

The platform is organized into **seven core layers**:

```
┌─────────────────────────────────────────────────────────────┐
│  Application Layer                                           │
│  (User code, agents, coordination logic)                     │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│  Coordination Engine                                         │
│  ├─ Task Delegation                                          │
│  ├─ Intent Matching                                          │
│  ├─ Workflow Orchestration                                   │
│  └─ State Management                                         │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│  Trust & Security Layer                                      │
│  ├─ Peer Authentication                                      │
│  ├─ Trust Establishment                                      │
│  ├─ Access Control                                           │
│  └─ Reputation Management                                    │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│  Capability & Context Layer                                  │
│  ├─ Capability Registry                                      │
│  ├─ Capability Matching                                      │
│  ├─ Context Synchronization                                  │
│  └─ State Exchange                                           │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│  Discovery & Registration Layer                              │
│  ├─ Peer Discovery                                           │
│  ├─ Advertisement                                            │
│  ├─ Registry Management                                      │
│  └─ Peer Metadata Sync                                       │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│  Transport Abstraction Layer                                 │
│  ├─ BLE (Implemented)                                        │
│  ├─ WiFi Direct (Planned)                                    │
│  ├─ 802.15.4 (Planned)                                       │
│  └─ LoRaWAN (Planned)                                        │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│  Observability & Diagnostics Layer                           │
│  ├─ Event Logging                                            │
│  ├─ Metrics Collection                                       │
│  ├─ Telemetry                                                │
│  └─ Debug Interfaces                                         │
└─────────────────────────────────────────────────────────────┘
```

## Component Architecture

### 1. Transport Layer

**Responsibility**: Abstract underlying mesh protocols and provide a unified interface for sending/receiving messages.

**Current Implementation**: BLE (Bluetooth Low Energy)
- GAP (Generic Access Profile) for discovery
- GATT (Generic Attribute Profile) for data exchange
- Advertisement flags for peer metadata

**Planned**: WiFi Direct, 802.15.4, LoRaWAN

**Key Classes/Interfaces**:
```typescript
interface Transport {
  advertise(data: AdvertisementPayload): Promise<void>;
  scan(filter?: ScanFilter): Observable<PeerAdvertisement>;
  send(peerId: string, message: Message): Promise<void>;
  on(event: string, handler: EventHandler): void;
}

class BLETransport implements Transport {
  // BLE-specific implementation
}
```

### 2. Discovery & Registration Layer

**Responsibility**: Maintain an up-to-date registry of nearby peers and their metadata.

**Key Functions**:
- Detect new peers via transport-level advertisements
- Track peer liveness (active/inactive/stale)
- Maintain peer metadata (node ID, device type, capabilities)
- Handle peer departure and cleanup
- Provide peer querying interface

**Key Classes**:
```typescript
class LocalPeerRegistry {
  registerPeer(peer: PeerInfo): void;
  getPeer(peerId: string): PeerInfo | null;
  getActivePeers(): PeerInfo[];
  getPeersByCapability(name: string): PeerInfo[];
  on(event: 'peer-joined' | 'peer-left'): void;
}

interface PeerInfo {
  peerId: string;
  deviceType: string;
  status: 'active' | 'inactive' | 'stale';
  capabilities: CapabilityInfo[];
  lastSeen: Date;
  metadata: Record<string, any>;
}
```

### 3. Capability & Context Layer

**Responsibility**: Manage capability advertisement, discovery, and matching; synchronize relevant state.

**Key Functions**:
- Register local capabilities
- Advertise capabilities to peers
- Query peer capabilities
- Match tasks to capabilities
- Exchange context state efficiently
- Handle capability version management

**Key Classes**:
```typescript
class CapabilityRegistry {
  register(capability: CapabilityDefinition): void;
  getLocal(): CapabilityDefinition[];
  getRemote(peerId: string): CapabilityDefinition[];
  query(filter: CapabilityFilter): CapabilityMatch[];
  matchIntent(intent: Intent): PeerMatch[];
}

interface CapabilityDefinition {
  name: string;
  version: string;
  parameters: ParameterSchema[];
  constraints: CapabilityConstraint[];
  metadata: Record<string, any>;
}

interface Intent {
  action: string;
  parameters: Record<string, any>;
  requiredCapability?: string;
  constraints?: IntentConstraint[];
  priority: 'low' | 'normal' | 'high';
}
```

### 4. Trust & Security Layer

**Responsibility**: Establish trust between peers, manage authentication, and enforce access control.

**Key Functions**:
- Challenge-response authentication
- Trust relationship establishment
- Trust state management (trusted/untrusted/unknown)
- Credential exchange
- Access control for sensitive operations

**Key Classes**:
```typescript
class TrustManager {
  establishTrust(peerId: string, options?: TrustOptions): Promise<TrustResult>;
  getTrustStatus(peerId: string): TrustStatus;
  revokeTrust(peerId: string): void;
  canExecuteTask(peerId: string, task: Intent): boolean;
}

enum TrustStatus {
  UNKNOWN = 'unknown',
  VERIFYING = 'verifying',
  TRUSTED = 'trusted',
  UNTRUSTED = 'untrusted',
  REVOKED = 'revoked'
}
```

**Trust Handshake Flow**:

```
Peer A                                    Peer B
  │
  ├─ initiates trust                      │
  │─────────────────────────────────→    │
  │     (TrustInitiation message)         │
  │                                       │ generates challenge
  │     ←─────────────────────────────    │
  │     (Challenge message)               │
  │                                       │
  │ solves challenge                      │
  │─────────────────────────────────→    │
  │     (ChallengeResponse message)       │
  │                                       │ verifies response
  │                                       │
  │     ←─────────────────────────────    │
  │     (TrustEstablished message)        │
  │                                       │
Peer A                              Peer B
(TRUSTED)                           (TRUSTED)
```

### 5. Coordination Engine

**Responsibility**: Execute task delegation, intent matching, and distributed coordination.

**Key Functions**:
- Accept intents from application
- Match intents to capable peers
- Delegate tasks with timeout/retry
- Collect results and handle failures
- Support complex workflows (Phase 4+)

**Key Classes**:
```typescript
class CoordinationEngine {
  delegateTask(intent: Intent, options?: DelegationOptions): Promise<TaskResult>;
  matchPeers(intent: Intent): PeerMatch[];
  executeWorkflow(workflow: Workflow): Promise<WorkflowResult>;
  on(event: 'task-completed' | 'task-failed'): void;
}

interface TaskResult {
  taskId: string;
  status: 'success' | 'failed' | 'timeout';
  result?: any;
  error?: Error;
  executedAt: string;
  executionTime: number;
}
```

**Task Delegation Flow**:

```
Local App                          Coordination Engine              Peer Registry
    │                                    │                              │
    ├─ delegateTask(intent)             │                              │
    │────────────────────────────→       │                              │
    │                                    ├─ matchIntent()              │
    │                                    ├─────────────────────────→  │
    │                                    │      (query capabilities)    │
    │                                    │ ←─────────────────────────  │
    │                                    │      (peer matches)          │
    │                                    │                              │
    │                                    ├─ selectBestPeer()           │
    │                                    │                              │
    │                                    ├─ send task to peer          │
    │                                    ├─ wait for result or timeout │
    │                                    │                              │
    │ ←─ TaskResult                      │                              │
    │────────────────────────────       │                              │
```

### 6. Observability & Diagnostics Layer

**Responsibility**: Provide visibility into mesh health, coordination decisions, and system state.

**Key Functions**:
- Event logging for all major operations
- Metric collection (peer count, task latency, success rates)
- Telemetry export
- Debug interfaces
- Performance profiling hooks

**Key Metrics**:
- Peer discovery latency
- Task delegation latency
- Peer match success rate
- Task completion success rate
- Mesh connectivity ratio
- Capability match hit rate

## Runtime Flow

### Startup Sequence

```
1. Initialize Transport (BLE, WiFi, etc.)
   ↓
2. Start Discovery (scan for nearby peers)
   ↓
3. Register Local Capabilities
   ↓
4. Begin Advertisement (tell peers about ourselves)
   ↓
5. Wait for Peers (discovery interval: typically 5-10s)
   ↓
6. Runtime Ready (applications can delegate tasks)
```

### Task Execution Flow (Detailed)

```
Step 1: Application creates Intent
  {
    action: "process-image",
    parameters: { imageBuffer: [...] },
    requiredCapability: "image-processing",
    priority: "high"
  }

Step 2: CoordinationEngine.delegateTask(intent)

Step 3: Query Capability Registry
  - Which peers have "image-processing" capability?
  - Are they trusted?
  - What is their current load?

Step 4: Match Peers
  - Peer A: image-processing v1.2, trusted, light load
  - Peer B: image-processing v2.0, untrusted, heavy load
  - Select Peer A (best match)

Step 5: Send Task Message
  - Serialized intent + task metadata
  - Via established trust relationship

Step 6: Wait for Result
  - Set timeout (default 30s)
  - Listen for task completion message

Step 7: Receive Result
  - Deserialize result
  - Update task metrics
  - Return to application

Step 8: Application receives result
  {
    taskId: "task-abc123",
    status: "success",
    result: { processed: true, ... },
    executionTime: 245
  }
```

## Capability Negotiation Flow

```
Peer A wants to know what Peer B can do:

1. Peer A: "Hey B, advertise your capabilities"
2. Peer B: Sends CapabilityAdvertisement
   {
     capabilities: [
       { name: "vision", version: "1.0", ... },
       { name: "motion", version: "2.1", ... }
     ]
   }
3. Peer A: Stores in CapabilityRegistry
4. Later: Peer A discovers intent needs "vision"
5. Peer A: Queries registry, finds Peer B
6. Peer A: "Hey B, execute this vision task"
```

## Context Synchronization Flow

```
Scenario: Multiple peers need to coordinate around shared state

Step 1: Coordinator has state S
  S = { location: (10, 20), battery: 85%, ... }

Step 2: Context message broadcast
  (via direct messaging to each peer or via flood)
  
Step 3: Peer receives context update
  - Merges with local state
  - Notifies listeners
  - Applies state consistency rules

Step 4: Peer executes task with updated context
  - Uses fresh state for decision-making
  - May broadcast its own state updates

Step 5: Eventual consistency achieved
  - All peers converge on consistent state
  - Time: typically <100ms in small meshes
```

## Data Structures

### Peer Advertisement (BLE Payload)

```
┌────────────────────────────────────┐
│ Node ID (8 bytes)                  │
├────────────────────────────────────┤
│ Device Type (2 bytes)              │
├────────────────────────────────────┤
│ Status Flags (1 byte)              │
├────────────────────────────────────┤
│ Capability Count (1 byte)          │
├────────────────────────────────────┤
│ Capability Hashes (N × 1 byte)     │
├────────────────────────────────────┤
│ Metadata Version (1 byte)          │
├────────────────────────────────────┤
│ Reserved (4 bytes)                 │
└────────────────────────────────────┘
```

### Task Message

```json
{
  "messageType": "TASK_DELEGATION",
  "taskId": "task-abc123",
  "sourceId": "peer-a",
  "targetId": "peer-b",
  "intent": {
    "action": "process-frame",
    "parameters": { "frame": "base64-encoded..." },
    "requiredCapability": "frame-processor"
  },
  "timestamp": 1717939200000,
  "timeout": 30000
}
```

## State Management

### Peer State Lifecycle

```
          ┌──────────────┐
          │   UNKNOWN    │
          └──────────────┘
                 │
                 │ peer discovered
                 ↓
          ┌──────────────┐
    ┌────→│   ACTIVE     │←────┐
    │     └──────────────┘     │
    │            │             │
    │            │ no activity  │
    │            ↓              │
    │     ┌──────────────┐      │
    │     │   INACTIVE   │      │
    │     └──────────────┘      │
    │            │              │
    │            │ reappears    │
    └────────────┴──────────────┘

    Or: INACTIVE → STALE → cleaned up
```

### Trust State Lifecycle

```
       ┌──────────────┐
       │   UNKNOWN    │
       └──────────────┘
              │
              │ initiate trust
              ↓
       ┌──────────────┐
       │  VERIFYING   │
       └──────────────┘
         ↙            ↘
        ✓              ✗
        ↓              ↓
  ┌──────────┐   ┌──────────────┐
  │ TRUSTED  │   │  UNTRUSTED   │
  └──────────┘   └──────────────┘
        │              │
        │ explicit     │ implicit (unused)
        │ revoke       │
        ↓              ↓
  ┌──────────┐   
  │ REVOKED  │   
  └──────────┘   
```

## Performance Characteristics

### Discovery Latency
- **First peer detection**: 0-2.5s (depends on scan window)
- **Typical mesh convergence**: 5-15s (all peers aware of each other)
- **Peer departure detection**: 30-60s (configurable inactivity timeout)

### Task Execution Latency
- **Peer matching**: <10ms (registry lookup)
- **Network transmission**: 50-200ms (BLE + transport overhead)
- **Median task round-trip**: 100-500ms (including execution)

### Scalability Characteristics
- **Peer count**: Optimal 5-20 peers per mesh
- **Tasks/sec**: ~100 tasks/sec per node (depends on task complexity)
- **Memory per peer**: ~5-10 KB (minimal state per node)
- **Bandwidth**: ~1-10 KB/sec (depends on discovery interval and context sync)

## Security Model (Phase 1)

### Current Implementation
- Challenge-response peer authentication
- Trust state tracking
- No encryption (added in Phase 6)
- Basic access control (trusted/untrusted)

### Planned (Phase 6)
- End-to-end encryption for all messages
- Certificate-based authentication
- Fine-grained access control
- Audit logging
- Revocation lists

---

**Last Updated**: June 2026 | **Version**: 0.1.0-alpha
