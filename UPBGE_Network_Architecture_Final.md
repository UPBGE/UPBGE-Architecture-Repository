# UPBGE Network Module

**Architecture & Integration Reference**

`source/gameengine/Network/` · 2026

---

## 0. Scope & Architectural Paradigm

### 0.1 Target

This architecture targets:

- **2–16 players** (cooperative, competitive action, small-scale multiplayer)
- **50–200 replicated objects** (primary target), functional up to ~500
- **LAN and Internet** with reasonable latency (< 200ms RTT)

This is not designed for: MMO (1000+ players), large-scale RTS (1000+ dynamic entities), competitive esports requiring sub-frame prediction, or distributed simulation. These require fundamentally different architectures.

### 0.2 Replication Model

This architecture uses **snapshot-based replication**:

- The server is the sole source of truth for all replicated state
- Clients receive periodic state snapshots and interpolate between them
- The same model used by Source Engine, Overwatch, and Rocket League

This architecture does **not** support:

- **Lockstep determinism** (required for RTS/fighting games) — lockstep requires bit-identical simulation across all peers, which is infeasible with floating-point physics (Bullet), Python game logic, and cross-platform builds
- **Peer-to-peer authority** — all authority is server-side

Client-side prediction is available as an optional module (§3.8) that can be enabled per-object when needed.

### 0.3 Validation Boundary

The network module guarantees **data integrity**: well-formed packets, sequence numbers, size limits, replay protection. **Gameplay validation** — whether a movement is physically possible, a fire rate is legal, or a damage value is valid — is the responsibility of the game logic layer, implemented via server-side RPC handlers or Logic Brick logic. See §9 for guidance and examples.

---

## 1. Design Principles

- **Performance first:** `Tick()` must cost < 0.3 ms at 60 FPS. Zero heap allocations in hot path. All pools pre-allocated at initialization.
- **Send rate ≠ frame rate:** Snapshots at 20 Hz (configurable), interpolation at render rate.
- **No allocations in hot path:** Pre-allocated pools, static scratch buffers, arena allocators for variable-size data.
- **Interchangeable provider:** ENet default. Nakama prepared behind compile flag.
- **No-code first:** Full configuration from Blender UI. Python for advanced cases.
- **Logic Brick parity:** Every core network operation available via Python is also available via Logic Bricks. No-code users are first-class citizens.
- **Explicit lifecycle:** No singletons. NetworkManager lifetime owned by `KX_KetsjiEngine`. Deterministic init/shutdown order.
- **Server-authoritative:** Server is the source of truth for all replicated state. Clients are predictive but server corrects.
- **Budget-bounded:** Every per-frame operation has a configurable time/count budget. No unbounded loops in the game loop.
- **Fail-safe degradation:** Network degradation reduces quality (lower send rate, larger interpolation window) but never blocks the game loop.
- **Optional complexity:** Advanced features (prediction, encryption) are opt-in and disabled by default. Default configuration is simple and performant.

---

## 2. File Structure — `source/gameengine/Network/`

All files prefixed `NET_`. No subdirectories.

| File | Layer | Responsibility |
|---|---|---|
| `NET_NetworkManager.h/.cpp` | Core | Explicit-lifetime manager. One `Tick()` per frame. Packet routing, send-rate accumulator, budget enforcement, adaptive send rate, slow consumer detection. Not a singleton. |
| `NET_NetworkTypes.h` | Core | Shared types: `PeerID`, `NetObjectHandle` (ID + generation), `RPCID`, `MessageType`, `NetChannel`, `RPCTarget`, `QuantizationProfile`. Compile-time constants. |
| `NET_NetworkStats.h` | Core | Lock-free stats struct (atomic writes from net tick, relaxed reads from UI). Ping, bandwidth, loss, jitter, arena overflows, queue depth, effective send rate. |
| `NET_NetworkConfig.h` | Core | Immutable config struct built from DNA at init. Holds all tunable parameters. Passed by const-ref, never mutated at runtime. |
| `NET_ConnectionState.h` | Core | Enum FSM: `Disconnected`, `Connecting`, `Connected`, `Disconnecting`. Timeout values, retry logic, transition callbacks. |
| `NET_PeerHealth.h` | Core | Per-peer health monitoring: RTT tracking, queue depth, auto-disconnect thresholds. |
| `NET_NetworkProvider.h` | Provider | Abstract interface. Methods: `StartServer`, `Connect`, `Service`, `PollEvent`, `Send`, `Broadcast`, `GetRTT`, `GetPacketLoss`, `GetPeerRTT`, `GetPeerQueueDepth`. Virtual `Encrypt()`/`Decrypt()` hooks. |
| `NET_ENetProvider.h/.cpp` | Transport | ENet UDP implementation. Circular event pool (fixed 64), deferred packet destroy, jitter estimation. |
| `NET_NakamaProvider.h` | Provider | Future stub. Compile-gated with `-DWITH_UPBGE_NAKAMA=ON`. |
| `NET_ReplicationManager.h/.cpp` | Replication | Dirty-check via FNV-1a + epoch. Priority queue with bandwidth budget. Interpolation with configurable jitter buffer. Robin Hood hash map for O(1) lookup. Adaptive send rate. |
| `NET_Snapshot.h` | Replication | Compact snapshot format. Delta encoding with bitfield masks. Per-object quantization profiles. |
| `NET_PriorityAccumulator.h` | Replication | Per-object priority accumulator with starvation-free guarantees. `minDistantObjectsPerTick` reserve. |
| `NET_RPCDispatcher.h/.cpp` | RPC | 16-bit RPC ID on wire. Direct array index lookup. GIL batching for Python handlers. Validates payload size before dispatch. |
| `NET_PacketValidator.h` | Security | Validates incoming packets: size limits, sequence numbers (replay protection), message type whitelist per connection state. |
| `NET_ArenaAllocator.h` | Memory | Frame-scoped bump allocator for variable-size serialization. Reset every tick. Zero fragmentation, zero syscalls. Overflow counter with graceful degradation. |
| `NET_ClientPrediction.h/.cpp` | Prediction | Optional client-side prediction + reconciliation. Disabled by default. Enabled per-object via `OB_NET_PREDICT` flag. |
| `NET_EncryptionProvider.h` | Security | Abstract encryption interface. Default: no-op pass-through (zero cost). Optional XChaCha20-Poly1305 behind compile flag `-DWITH_UPBGE_NET_ENCRYPTION=ON`. |
| `NET_PythonNetwork.cpp` | Python | Module `bge.network`. Complete API surface. |

**Logic Brick files** (in `source/gameengine/GameLogic/`):

| File | Responsibility |
|---|---|
| `SCA_NetworkEventSensor.h/.cpp` | Network event detection: connection state, peer events, RPC received, replication events. |
| `SCA_NetworkActuator.h/.cpp` | Network actions: connect, disconnect, replicate, send RPC, set ownership. |

---

## 3. Architecture Detail

### 3.1 Lifecycle Management

NetworkManager is an explicit-lifetime object owned by `KX_KetsjiEngine`. Created in `ConvertScene()`, destroyed in `EndGame()`. Passed by pointer/reference to systems that need it. Zero static state.

```cpp
class NET_NetworkManager {
public:
    explicit NET_NetworkManager(const NET_NetworkConfig& config);
    ~NET_NetworkManager(); // Deterministic cleanup
    NET_NetworkManager(const NET_NetworkManager&) = delete;
    NET_NetworkManager& operator=(const NET_NetworkManager&) = delete;

    void Tick(float dt);
    void Shutdown();
};

// In KX_KetsjiEngine:
class KX_KetsjiEngine {
    std::unique_ptr<NET_NetworkManager> m_networkManager;
    // Created in ConvertScene(), null-checked in NextFrame()
};
```

**Performance impact:** Zero. Pointer dereference vs static-local guard. Actually faster on some compilers since we avoid the hidden mutex in static initialization.

---

### 3.2 Connection State Machine

Explicit FSM with 4 states and deterministic transitions. Timeout-driven. All network operations check current state before executing.

```cpp
enum class ConnectionState : uint8_t {
    Disconnected  = 0, // No active connection
    Connecting    = 1, // Handshake in progress, timeout armed
    Connected     = 2, // Fully operational
    Disconnecting = 3  // Graceful shutdown in progress
};

struct NET_ConnectionFSM {
    ConnectionState state       = ConnectionState::Disconnected;
    float           stateTimer  = 0.0f;
    float           connectTimeoutSec = 5.0f;
    uint8_t         retryCount  = 0;
    uint8_t         maxRetries  = 3;

    bool TransitionTo(ConnectionState newState);
    void Update(float dt);
};
```

**State transitions:**

```
Disconnected → (Connect called)        → Connecting
Connecting   → (Handshake OK)          → Connected
Connected    → (Disconnect called)     → Disconnecting
Disconnecting→ (ACK or timeout)        → Disconnected
Connecting   → (Timeout, retries done) → Disconnected
Connected    → (Timeout, no keepalive) → Disconnected
```

**Performance impact:** Negligible. One enum compare per `Tick()`. Early-out when not in `Connected` state prevents wasted work.

---

### 3.3 Budget-Capped Packet Processing with Burst Mode

Configurable `maxEventsPerTick` (default 64). Overflow events remain in the provider's queue and are processed next frame. Priority: disconnect events first (cleanup), then data, then connects (heaviest).

During initial sync (first 2 seconds after connection), a **burst budget** allows processing up to 512 events per tick. If queue depth exceeds 4× the normal budget for consecutive ticks, the budget is temporarily elevated to prevent unbounded queue growth.

```cpp
void NET_NetworkManager::ProcessIncomingPackets() {
    // Determine effective budget
    int budget = m_config.maxEventsPerTick;  // Default: 64

    // Burst mode: elevated budget during initial sync window
    if (m_timeSinceConnect < m_config.initialSyncWindow) {  // Default: 2.0s
        budget = m_config.maxEventsInitialSync;              // Default: 512
    }
    // Queue pressure: prevent unbounded growth
    else if (m_queueDepth > budget * 4) {
        budget = std::min(budget * 4, m_config.maxEventsInitialSync);
        m_stats.queuePressureEvents++;
    }

    int processed = 0;
    NET_ProviderEvent event;

    while (processed < budget && m_provider->PollEvent(event)) {
        switch (event.type) {
            case EventType::Disconnect:
                HandleDisconnect(event); break;
            case EventType::Data:
                if (m_validator.Validate(event.packet)) {
                    RoutePacket(event);
                }
                break;
            case EventType::Connect:
                HandleConnect(event); break;
        }
        ++processed;
    }
    m_queueDepth = m_provider->GetPendingEventCount();
}
```

**Configuration:**

```cpp
int   maxEventsPerTick     = 64;    // Normal operation
int   maxEventsInitialSync = 512;   // Burst mode (first 2s + queue pressure)
float initialSyncWindow    = 2.0f;  // Seconds of elevated budget after connect
```

**Performance impact:** Zero cost in normal operation (one extra comparison). Burst mode is a one-time cost per connection, not per frame. At 512 events × ~2µs = ~1ms, which is acceptable during the initial loading/sync period.

---

### 3.4 Bandwidth-Budgeted Replication with Priority Queue

Priority accumulator with per-tick bandwidth budget. Objects sorted by accumulated priority. Budget consumed per-object. Excess deferred to next send tick with accumulated priority preserved (starvation-free).

A `minDistantObjectsPerTick` parameter (default: 2) reserves bandwidth slots for the highest-priority distant objects, preventing total starvation under extreme nearby load.

```cpp
struct ObjectPriority {
    NetObjectHandle handle;
    float accumulator;  // Grows each tick not sent
    float basePriority; // 1.0 normal, higher = more important
};

void NET_ReplicationManager::FlushOutgoing(float budgetBytes) {
    // 1. Dirty-check all registered objects (FNV-1a + epoch)
    UpdateDirtyFlags();

    // 2. Accumulate priority for all dirty objects
    for (auto& obj : m_dirtyObjects) {
        obj.accumulator += obj.basePriority * DistanceWeight(obj);
    }

    // 3. Partial sort: only need top-N, not full sort
    //    std::nth_element is O(n), not O(n log n)
    float bytesSent = 0;
    std::nth_element(begin, begin + estimatedFit, end, PrioCmp);

    // 4. Send until budget exhausted
    for (auto& obj : m_dirtyObjects) {
        float packetSize = EstimateSnapshotSize(obj);
        if (bytesSent + packetSize > budgetBytes) break;
        SendDeltaSnapshot(obj);
        bytesSent += packetSize;
        obj.accumulator = 0;
    }
    // Objects that didn't send keep their accumulated priority
    // — guaranteed starvation-free

    // 5. Reserve slots for distant objects
    int distantSent = 0;
    for (auto& obj : m_distantDirtyObjects) {
        if (distantSent >= m_config.minDistantObjectsPerTick) break;
        if (bytesSent + EstimateSnapshotSize(obj) > budgetBytes) break;
        SendDeltaSnapshot(obj);
        bytesSent += EstimateSnapshotSize(obj);
        obj.accumulator = 0;
        ++distantSent;
    }
}
```

**Performance impact:** `nth_element` is O(n). Net cost is similar to iterate-all but with guaranteed bandwidth ceiling. The accumulator guarantees all objects eventually sync regardless of distance.

---

### 3.5 Compact Snapshot Format with Configurable Quantization

Explicit compact delta format with bitfield masks and per-object quantization profiles selectable in the Blender UI.

**Quantization Profiles:**

| Profile | Enum | Range | Precision | Bytes/axis | Use case |
|---------|------|-------|-----------|------------|----------|
| `QUANT_STANDARD` | 0 | ±327m | 1cm | 2 | Default, most objects |
| `QUANT_HIGH` | 1 | ±327m | 0.1mm | 4 (float32) | Precision physics, headshots |
| `QUANT_WORLD` | 2 | ±32,767m | 1m | 2 | Large maps, distant objects |
| `QUANT_NONE` | 3 | Full float | Full float | 4 (float32) | Special cases, debugging |

**Sector Origin:** Objects with `QUANT_STANDARD` or `QUANT_HIGH` quantize relative to a configurable sector origin, allowing arbitrary map sizes with standard precision. The sector origin is an object property (`network_sector_origin`) defaulting to (0,0,0).

**Snapshot header (6 bytes):**

```cpp
enum class QuantizationProfile : uint8_t {
    Standard = 0,
    High     = 1,
    World    = 2,
    None     = 3
};

struct NET_SnapshotHeader {
    uint16_t objectId;      // Object slot
    uint8_t  generation;    // ID generation counter (§3.13)
    uint8_t  dirtyMask;     // Bitfield: pos|rot|scale|custom
    uint8_t  quantProfile;  // QuantizationProfile enum
    uint8_t  sequence;      // Per-object sequence number
};
```

**Snapshot body sizes:**

| Field | Full Size | Standard Quantized | Method |
|---|---|---|---|
| Position (vec3) | 12 bytes | 6 bytes | 16-bit per axis relative to sector origin. |
| Rotation (quat) | 16 bytes | 4 bytes | Smallest-three: drop largest component (2-bit index), 10-bit per remaining axis. ~0.06° precision. |
| Scale (vec3) | 12 bytes | 3 bytes | 8-bit per axis, [0.0, 8.0] range at ~0.03 resolution. |
| **TOTAL (pos+rot)** | **32 bytes** | **14 bytes** | **56% bandwidth reduction for the common case** |

**Blender UI:** Per-object dropdown in the Network panel: "Quantization: Standard / High Precision / World Scale / None". Exposed via RNA as `object.network_quant_profile`.

**Performance impact:** One extra byte in header per snapshot. Quantization cost is the same — the profile selects the encoding function at serialization time via a function pointer table (no branching in hot path).

---

### 3.6 Hash Collision Safety

Dirty-check relies on FNV-1a hash. Two-layer defense against collision:

1. **Epoch counter** incremented on every mutation ensures hash+epoch never matches by accident
2. **Periodic full snapshot heartbeat** (every 100 send ticks = 5 seconds at 20Hz) guarantees eventual consistency regardless of any hash collision

The FNV-1a hash is computed **server-side only**. Clients never send hashes. A malicious client cannot force hash collisions in the server-authoritative model.

```cpp
struct NET_ObjectState {
    uint64_t hash;
    uint32_t epoch;              // Incremented on every SetTransform()
    uint16_t ticksSinceFullSync; // Counts up to fullSyncInterval
};

bool NET_ReplicationManager::IsDirty(NET_ObjectState& state,
                                     const void* data, size_t len)
{
    uint64_t newHash = FNV1a_64(data, len);

    if (newHash != state.hash || state.epoch != state.lastSentEpoch) {
        state.hash = newHash;
        return true;
    }

    if (++state.ticksSinceFullSync >= m_config.fullSyncInterval) {
        state.ticksSinceFullSync = 0;
        return true; // Force-send even if hash matches
    }

    return false;
}
```

**Performance impact:** One extra `uint32` compare per object per dirty-check (~0 additional cost). Full sync every 5 seconds adds negligible average bandwidth.

---

### 3.7 Packet Validation

Lightweight validation layer that runs before any packet processing. Fixed overhead, no heap allocation, no string operations.

```cpp
struct NET_PacketValidator {
    uint16_t expectedSeq[MAX_PEERS];

    bool Validate(PeerID peer, const uint8_t* data, size_t len) {
        // 1. Minimum size (header must be present)
        if (len < NET_PACKET_HEADER_SIZE) return false;

        // 2. Maximum size (prevent buffer overrun)
        if (len > NET_MAX_PACKET_SIZE) return false;

        // 3. Message type must be valid enum value
        uint8_t msgType = data[0];
        if (msgType >= static_cast<uint8_t>(MessageType::COUNT))
            return false;

        // 4. Sequence number (wrapping uint16 comparison)
        uint16_t seq  = ReadU16(data + 1);
        int16_t  diff = (int16_t)(seq - expectedSeq[peer]);
        if (diff <= 0 && diff > -1000) return false;

        expectedSeq[peer] = seq + 1;
        return true;
    }
};
```

Packet integrity at the transport level is handled by ENet's built-in CRC32. An additional application-layer checksum is not necessary.

**Performance impact:** ~50ns per packet (3 comparisons + 1 uint16 read).

---

### 3.8 Client-Side Prediction (Optional, Disabled by Default)

Client-side prediction is extremely game-specific. An FPS needs sub-frame prediction; an RTS does not; a puzzle coop does not. Therefore, prediction is an **optional module**, disabled by default, enabled per-object via the `OB_NET_PREDICT` flag in the Blender UI.

**When enabled:**

- The owning client applies inputs locally immediately (no waiting for server confirmation)
- The client stores a history of inputs and predicted states
- When the server snapshot arrives, the client compares its predicted state with the server state
- If discrepancy exceeds a configurable threshold, the client corrects smoothly (exponential blend, not snap)

```cpp
struct PredictionEntry {
    uint32_t inputSequence;
    float    predictedPos[3];
    float    predictedRot[4];
    float    timestamp;
};

class NET_ClientPrediction {
    static constexpr int HISTORY_SIZE = 64;  // ~3.2s at 20Hz
    PredictionEntry m_history[HISTORY_SIZE];
    int m_head = 0;
    float m_correctionBlend = 0.1f;

public:
    // Called when local input is applied (client-side, immediate)
    void RecordPrediction(uint32_t inputSeq,
                          const float pos[3], const float rot[4]);

    // Called when server snapshot arrives
    // Returns correction delta if prediction was wrong
    bool Reconcile(uint32_t serverAckInputSeq,
                   const float serverPos[3], const float serverRot[4],
                   float outCorrectionPos[3], float outCorrectionRot[4]);
};
```

**Blender UI:** Per-object checkbox: "Enable Client Prediction". Only meaningful for the local player's controlled objects. Greyed out unless `OB_NET_REPLICATE` is enabled.

**Limitations:** This is a simple predict-reconcile system for position/rotation. It does **not** include physics rollback (would require re-simulating Bullet, infeasible), weapon fire prediction (game-specific), or lag compensation (hit registration). These are game-layer concerns.

**Performance impact:** ~5µs per predicted object per tick. For 1 player object, negligible. The correction blend adds ~2µs in `ApplyInterpolation` only when a correction is active. **Zero cost when disabled.**

---

### 3.9 Encryption (Optional, Disabled by Default)

Encryption adds ~50–100µs per packet, which is significant for a system with a 300µs budget. Most indie games don't need it. It is therefore opt-in.

```cpp
class NET_IEncryption {
public:
    virtual ~NET_IEncryption() = default;
    virtual size_t Encrypt(const uint8_t* plaintext, size_t len,
                           uint8_t* ciphertext, size_t maxLen) = 0;
    virtual size_t Decrypt(const uint8_t* ciphertext, size_t len,
                           uint8_t* plaintext, size_t maxLen) = 0;
};

// Default: no-op pass-through (zero cost)
class NET_NoEncryption : public NET_IEncryption {
    size_t Encrypt(const uint8_t* in, size_t len,
                   uint8_t* out, size_t maxLen) override {
        memcpy(out, in, len);
        return len;
    }
    size_t Decrypt(const uint8_t* in, size_t len,
                   uint8_t* out, size_t maxLen) override {
        memcpy(out, in, len);
        return len;
    }
};
```

`NET_NetworkProvider::Send()` calls `m_encryption->Encrypt()` before passing to ENet. `NET_NetworkProvider::OnReceive()` calls `m_encryption->Decrypt()` before passing to `ProcessIncomingPackets()`.

**Blender UI:** Checkbox in the Network Advanced panel: "Enable Packet Encryption". Disabled by default.

**Compile flag:** `-DWITH_UPBGE_NET_ENCRYPTION=ON` enables a concrete XChaCha20-Poly1305 implementation using libsodium. Without the flag, only `NET_NoEncryption` is available.

---

### 3.10 Frame-Scoped Arena Allocator

Arena allocator that resets every tick. Supports variable-size allocations without fragmentation. Pre-allocated at init, zero syscalls during gameplay.

On overflow: serialization stops gracefully, a per-tick warning is logged once, the overflow counter is exposed in `NET_NetworkStats`, and remaining objects keep their accumulated priority for next tick.

```cpp
class NET_ArenaAllocator {
    alignas(64) uint8_t m_buffer[NET_ARENA_SIZE]; // Configurable, default 64KB
    size_t   m_offset = 0;
    uint32_t m_overflowCount = 0;
    bool     m_overflowThisTick = false;

public:
    void* Allocate(size_t size) {
        size_t aligned = (m_offset + 7) & ~7;
        if (aligned + size > NET_ARENA_SIZE) {
            m_overflowCount++;
            m_overflowThisTick = true;
            return nullptr;
        }
        void* ptr = m_buffer + aligned;
        m_offset = aligned + size;
        return ptr;
    }

    void Reset() {
        m_offset = 0;
        m_overflowThisTick = false;
    }

    bool     OverflowedThisTick() const { return m_overflowThisTick; }
    uint32_t TotalOverflows()     const { return m_overflowCount; }
    size_t   Used()               const { return m_offset; }
    size_t   Capacity()           const { return NET_ARENA_SIZE; }
};
```

In `FlushOutgoing()`:

```cpp
void* mem = m_arena.Allocate(snapshotSize);
if (!mem) {
    // Graceful degradation: stop serializing this tick
    if (!m_loggedOverflowThisTick) {
        CM_Warning("NET: Arena overflow, deferring "
                   << (dirtyCount - sentCount) << " objects");
        m_loggedOverflowThisTick = true;
    }
    break;  // Objects that weren't sent keep accumulated priority
}
```

`NET_ARENA_SIZE` is configurable in `NET_NetworkConfig` (default 64KB, max 256KB). 64KB is sufficient for ~4600 objects with quantized snapshots at 14 bytes each.

**Performance impact:** Allocation is a pointer bump + alignment (~1ns). Reset is a single store.

---

### 3.11 Interpolation with Configurable Jitter Buffer

Jitter buffer holds 2–4 snapshots and renders with a configurable delay. This absorbs jitter at the cost of slightly increased visual latency. Extrapolation kicks in only when the buffer runs dry.

```cpp
struct InterpolationBuffer {
    static constexpr int BUFFER_SIZE = 4;

    struct Entry {
        NET_Snapshot snapshot;
        float        serverTime;
    };

    Entry entries[BUFFER_SIZE]; // Circular
    int   head  = 0;
    int   count = 0;
    float interpDelay;  // Configurable, default 100ms
};

void NET_ReplicationManager::ApplyInterpolation(float renderTime) {
    for (auto& obj : m_replicatedObjects) {
        auto& buf = obj.interpBuffer;
        float t   = renderTime - buf.interpDelay;

        auto [a, b] = FindBracket(buf, t);

        if (a && b) {
            // Normal interpolation between two snapshots
            float alpha = (t - a->serverTime) /
                          (b->serverTime - a->serverTime);
            alpha = Clamp(alpha, 0.0f, 1.0f);
            LerpTransform(obj, a->snapshot, b->snapshot, alpha);
        }
        else if (a) {
            // Buffer underrun: extrapolate from last known velocity
            ExtrapolateTransform(obj, *a, t - a->serverTime);
        }
    }
}
```

**Configuration:**

```cpp
float interpDelay = 0.1f;  // Default: 100ms (2 snapshots at 20Hz)
                            // Minimum: 0.05f (50ms, 1 snapshot)
                            // Maximum: 0.5f (500ms)
```

**Blender UI:** Slider in the Network Advanced panel: "Interpolation Delay (ms)". Range 50–500, default 100.

**Python API:** `bge.network.set_interp_delay(0.05)` — sets to 50ms for lower latency at the cost of more visual stutter under jitter.

**Performance impact:** ~3µs per object. Jitter buffer adds 4 snapshot entries per object (~56 bytes). For 200 objects = ~11KB total.

---

### 3.12 GIL Batching for Python RPCs

Single GIL acquisition per tick for all pending Python RPCs, instead of per-RPC.

```cpp
void NET_RPCDispatcher::DispatchPendingRPCs() {
    if (m_pendingPythonRPCs.empty()) return;

    // Acquire GIL once for all Python RPCs this tick
    PyGILState_STATE gstate = PyGILState_Ensure();

    for (auto& rpc : m_pendingPythonRPCs) {
        InvokePythonHandler(rpc.handlerIndex, rpc.args, rpc.argsLen);
    }

    PyGILState_Release(gstate);
    m_pendingPythonRPCs.clear();

    // C++ RPCs dispatched separately without GIL
}
```

**Performance impact:** 50 Python RPCs: 5µs (1 GIL acquire) + 50 × 2µs (dispatch) = ~105µs. Without batching this would be 50 × 20µs = 1ms. C++ handlers recommended for high-frequency RPCs regardless.

---

### 3.13 Adaptive Send Rate

Send rate adapts based on ENet RTT and packet loss. Prevents congestion death spiral under degraded conditions.

```cpp
void NET_ReplicationManager::AdaptSendRate() {
    float loss = m_provider->GetPacketLoss();   // 0.0–1.0
    float rtt  = m_provider->GetRTT();          // ms

    if (loss > 0.10f || rtt > 200.0f) {
        m_effectiveSendRate = m_config.sendRate * 0.5f;   // Half rate
    } else if (loss > 0.05f || rtt > 100.0f) {
        m_effectiveSendRate = m_config.sendRate * 0.75f;  // 3/4 rate
    } else {
        m_effectiveSendRate = m_config.sendRate;           // Full rate
    }

    m_effectiveSendInterval = 1.0f / m_effectiveSendRate;
}
```

**Performance impact:** ~10ns per send tick (two float comparisons).

**Stats:** `bge.network.get_stats()["effective_send_rate"]` reports the current adapted rate.

---

### 3.14 Slow Consumer Protection

Per-peer health monitoring with configurable auto-disconnect thresholds. Prevents one bad peer from degrading the server for all players.

```cpp
struct NET_PeerHealth {
    float avgRTT            = 0.0f;
    int   queueDepth        = 0;
    float timeSinceResponse = 0.0f;
};

void NET_NetworkManager::CheckPeerHealth(float dt) {
    for (auto& [peerId, health] : m_peerHealth) {
        health.avgRTT = m_provider->GetPeerRTT(peerId);
        health.queueDepth = m_provider->GetPeerQueueDepth(peerId);
        health.timeSinceResponse += dt;

        if (health.avgRTT > m_config.maxPeerRTT ||              // Default: 500ms
            health.queueDepth > m_config.maxPeerQueueDepth ||    // Default: 256
            health.timeSinceResponse > m_config.peerTimeout)     // Default: 10s
        {
            DisconnectPeer(peerId, DisconnectReason::SlowConsumer);
            CM_Warning("NET: Disconnected peer " << peerId
                       << " (RTT=" << health.avgRTT
                       << "ms, queue=" << health.queueDepth << ")");
        }
    }
}
```

**Configuration:**

```cpp
float maxPeerRTT         = 500.0f;  // ms
int   maxPeerQueueDepth  = 256;     // packets
float peerTimeout        = 10.0f;   // seconds without response
```

**Performance impact:** ~1µs for 16 peers per tick.

---

### 3.15 NetObjectID with Generation Counter

8-bit generation counter alongside the 16-bit object ID. Prevents late packets from affecting reused IDs.

```cpp
struct NetObjectHandle {
    uint16_t id;          // Object slot (0–65535)
    uint8_t  generation;  // Wraps at 256
};

NetObjectHandle NET_ReplicationManager::Register(KX_GameObject* obj, ...) {
    uint16_t id = AllocateSlot();
    m_generations[id]++;
    return { id, m_generations[id] };
}

bool NET_ReplicationManager::ValidateHandle(NetObjectHandle handle) {
    return (handle.generation == m_generations[handle.id]);
}
```

**Wire cost:** 1 extra byte per snapshot header (included in the 6-byte header defined in §3.5). A packet would need to arrive 256 object lifetimes late to cause a conflict — effectively impossible.

---

## 4. Network Logic Bricks

### 4.1 Design Philosophy

UPBGE's core promise is "no-code first." The network module provides Logic Bricks that cover the most common network operations without writing a single line of Python.

**Why only 2 logic bricks (not 5–6)?** Performance. Every logic brick type adds to the per-frame sensor scan cost. By using mode enums within a single sensor and a single actuator, we keep the logic brick count minimal while covering all use cases. This is the same pattern as the existing Motion Actuator (which has Simple, Servo, and Character modes).

**Relationship to existing Message system:** UPBGE already has `SCA_NetworkMessageSensor` and `SCA_NetworkMessageActuator` for **local** inter-object messaging (within the same engine instance). The network Logic Bricks are for **remote** messaging (across the network to other peers). They are separate systems with distinct names.

### 4.2 Network Event Sensor — `SCA_NetworkEventSensor`

**DNA type name:** `SENS_NETWORK_EVENT`

Detects network events and produces positive/negative pulses to connected controllers.

**Modes:**

| Mode | Pulse Positive When | Readable Properties |
|---|---|---|
| `CONNECTION_STATE` | Connection state changes (connected/disconnected) | `connectionState` (string) |
| `PEER_EVENT` | A peer connects or disconnects | `peerID` (int), `isConnect` (bool) |
| `RPC_RECEIVED` | An RPC with matching subject is received | `subject` (string), `body` (string), `senderPeerID` (int) |
| `REPLICATION_EVENT` | Object replication starts/stops/ownership changes | `eventType` (string), `netObjectID` (int) |

**Blender UI:**

```
+--[ Network Event ]----------------------------------+
| Mode: [Connection State ▼]                          |
|                                                      |
| (mode=RPC_RECEIVED only:)                           |
| Subject Filter: [________________]  (blank = all)    |
+-----------------------------------------------------+
```

**Implementation:**

```cpp
class SCA_NetworkEventSensor : public SCA_ISensor {
    enum Mode : uint8_t {
        CONNECTION_STATE  = 0,
        PEER_EVENT        = 1,
        RPC_RECEIVED      = 2,
        REPLICATION_EVENT = 3,
    };

    Mode        m_mode;
    std::string m_subjectFilter;  // For RPC_RECEIVED mode

    // Cached event data (written by NetworkManager, read by controllers)
    bool        m_triggered = false;
    int         m_peerID    = -1;
    std::string m_rpcSubject;
    std::string m_rpcBody;
    std::string m_connectionState;

public:
    bool Evaluate() override;
    void Init() override;
    bool IsPositiveTrigger() override { return m_triggered; }

    // Python properties for reading event data in controllers
    KX_PYMETHOD_DOC_NOARGS(SCA_NetworkEventSensor, GetPeerID);
    KX_PYMETHOD_DOC_NOARGS(SCA_NetworkEventSensor, GetRPCSubject);
    KX_PYMETHOD_DOC_NOARGS(SCA_NetworkEventSensor, GetRPCBody);
    KX_PYMETHOD_DOC_NOARGS(SCA_NetworkEventSensor, GetConnectionState);
};
```

**`Evaluate()` implementation:**

```cpp
bool SCA_NetworkEventSensor::Evaluate() {
    NET_NetworkManager* net = GetNetworkManager();
    if (!net) {
        m_triggered = false;
        return false;
    }

    bool prev = m_triggered;

    switch (m_mode) {
        case CONNECTION_STATE: {
            auto state = net->GetConnectionState();
            m_triggered = (state != m_lastState);
            m_lastState = state;
            m_connectionState = ConnectionStateToString(state);
            break;
        }
        case PEER_EVENT: {
            auto event = net->PopPeerEvent();
            m_triggered = event.has_value();
            if (m_triggered) {
                m_peerID = event->peerId;
                m_isConnect = event->isConnect;
            }
            break;
        }
        case RPC_RECEIVED: {
            auto rpc = net->PopRPCForSensor(m_subjectFilter);
            m_triggered = rpc.has_value();
            if (m_triggered) {
                m_rpcSubject = rpc->subject;
                m_rpcBody = rpc->body;
                m_peerID = rpc->senderPeerID;
            }
            break;
        }
        case REPLICATION_EVENT: {
            auto evt = net->PopReplicationEvent(GetParent());
            m_triggered = evt.has_value();
            break;
        }
    }

    return (m_triggered != prev) || m_triggered;
}
```

**Performance impact:** `Evaluate()` is ~50–100ns (one switch, one queue check). The game engine only evaluates sensors linked to active controllers, so unlinked sensors cost nothing.

### 4.3 Network Actuator — `SCA_NetworkActuator`

**DNA type name:** `ACT_NETWORK`

Performs network operations when activated by a controller.

**Modes:**

| Mode | Action on Positive Pulse | Parameters |
|---|---|---|
| `CONNECT` | Connect to server / Start server | `address` (string), `port` (int), `asServer` (bool) |
| `DISCONNECT` | Disconnect from current session | (none) |
| `REPLICATE` | Register/unregister this object for replication | `enable` (bool), `quantProfile` (enum), `priority` (float) |
| `SEND_RPC` | Send an RPC to remote peers | `subject` (string), `body` (string/property), `target` (enum: Server/All/Owner/Others), `reliable` (bool) |
| `SET_OWNER` | Claim or release ownership of this object | `claim` (bool) |

**Blender UI:**

```
+--[ Network ]----------------------------------------+
| Mode: [Send RPC ▼]                                  |
|                                                      |
| Subject: [player_action______]                       |
| Body:    [Property ▼] [health____]                   |
| Target:  [All Clients ▼]                             |
| ☑ Reliable                                           |
+-----------------------------------------------------+
```

The "Body" field can be either a literal string or a reference to a game property. This allows sending dynamic game data (health, score, position) without Python.

**Implementation:**

```cpp
class SCA_NetworkActuator : public SCA_IActuator {
    enum Mode : uint8_t {
        CONNECT     = 0,
        DISCONNECT  = 1,
        REPLICATE   = 2,
        SEND_RPC    = 3,
        SET_OWNER   = 4,
    };

    Mode        m_mode;

    // CONNECT params
    std::string m_address;
    int         m_port        = 7777;
    bool        m_asServer    = false;

    // REPLICATE params
    bool        m_enable      = true;
    uint8_t     m_quantProfile = 0;
    float       m_priority    = 1.0f;

    // SEND_RPC params
    std::string m_rpcSubject;
    std::string m_rpcBody;
    std::string m_rpcBodyProperty; // Property name to read value from
    uint8_t     m_rpcTarget   = 0;
    bool        m_rpcReliable = true;

    // SET_OWNER params
    bool        m_claimOwnership = true;

public:
    bool Update(double curtime) override;
};
```

**`Update()` implementation:**

```cpp
bool SCA_NetworkActuator::Update(double curtime) {
    bool bPositivePulse = IsPositivePulse();
    RemoveAllEvents();

    if (!bPositivePulse) return false;

    NET_NetworkManager* net = GetNetworkManager();
    if (!net) return false;

    switch (m_mode) {
        case CONNECT:
            if (m_asServer)
                net->StartServer(m_port);
            else
                net->ConnectToServer(m_address.c_str(), m_port);
            break;

        case DISCONNECT:
            net->Disconnect();
            break;

        case REPLICATE: {
            KX_GameObject* obj = static_cast<KX_GameObject*>(GetParent());
            if (m_enable)
                net->GetReplication().Register(obj, m_quantProfile, m_priority);
            else
                net->GetReplication().Unregister(obj);
            break;
        }

        case SEND_RPC: {
            std::string body = m_rpcBody;
            if (!m_rpcBodyProperty.empty()) {
                KX_GameObject* obj = static_cast<KX_GameObject*>(GetParent());
                EXP_Value* prop = obj->GetProperty(m_rpcBodyProperty);
                if (prop) body = prop->GetText();
            }

            net->GetRPCDispatcher().EnqueueRPC(
                m_rpcSubject, body,
                static_cast<RPCTarget>(m_rpcTarget),
                m_rpcReliable);
            break;
        }

        case SET_OWNER: {
            KX_GameObject* obj = static_cast<KX_GameObject*>(GetParent());
            NetObjectHandle handle = net->GetReplication().GetHandle(obj);
            if (m_claimOwnership)
                net->GetReplication().ClaimOwnership(handle);
            else
                net->GetReplication().ReleaseOwnership(handle);
            break;
        }
    }

    return true;
}
```

**Performance impact:** `Update()` is called only on positive pulse from a controller. Cost per invocation: ~200–500ns. Zero cost when not triggered.

### 4.4 Logic Brick Use Case Examples

**Example 1: Connect on game start, replicate player**

```
Object: Player

Sensors:                Controllers:        Actuators:
[Always (pulse once)]---[AND]-------------- [Network: CONNECT]
                                               Address: "192.168.1.10"
                                               Port: 7777
                                               As Server: ☐

[Always (pulse once)]---[AND]-------------- [Network: REPLICATE]
                                               Enable: ☑
                                               Quantization: Standard
                                               Priority: 1.0
```

**Example 2: Send health update when damage received**

```
Object: Player

Sensors:                Controllers:        Actuators:
[Property "health"  ]---[AND]-------------- [Network: SEND_RPC]
  (Changed)                                    Subject: "health_update"
                                               Body: Property "health"
                                               Target: All Clients
                                               Reliable: ☑
```

**Example 3: React to RPC from server (receive damage)**

```
Object: Player

Sensors:                Controllers:        Actuators:
[Network Event      ]---[Python]----------- [Property: "health"]
  Mode: RPC_RECEIVED                           (set via Python controller)
  Subject: "take_damage"

# Python controller reads: sensor.body → damage amount
# Then sets property and activates actuator
```

**Example 4: Detect peer disconnect, show UI message**

```
Object: UIManager

Sensors:                Controllers:        Actuators:
[Network Event      ]---[AND]-------------- [Visibility: Show "Disconnected" overlay]
  Mode: PEER_EVENT
```

### 4.5 DNA Structs for Logic Bricks

**`source/blender/makesdna/DNA_actuator_types.h`:**

```c
typedef struct bNetworkActuator {
    short mode;          // ACT_NET_CONNECT, ACT_NET_DISCONNECT, etc.
    short port;
    short rpcTarget;     // RPCTarget enum
    short quantProfile;  // QuantizationProfile enum
    float priority;
    short rpcReliable;
    short pad;
    char address[64];
    char rpcSubject[64];
    char rpcBody[256];
    char rpcBodyProperty[64];
} bNetworkActuator;
```

**`source/blender/makesdna/DNA_sensor_types.h`:**

```c
typedef struct bNetworkEventSensor {
    short mode;           // SENS_NET_CONNECTION, SENS_NET_PEER, etc.
    short pad;
    char subjectFilter[64];
} bNetworkEventSensor;
```

### 4.6 Logic Brick Performance Budget

| Operation | Cost | When |
|---|---|---|
| Sensor `Evaluate()` | ~50–100ns | Every logic tick (only if linked to active controller) |
| Actuator `Update()` | ~200–500ns | Only on positive pulse |
| Queue check (PopPeerEvent, PopRPC) | ~10ns | Inside Evaluate() |

For a typical game with 5 network sensors and 5 network actuators across all objects: worst case = 5 × 100ns + 5 × 500ns = ~3µs per logic tick. Negligible.

---

## 5. Game Loop Integration

### 5.1 Converter (DNA Read at Game Start)

NetworkManager is created as a `unique_ptr` owned by `KX_KetsjiEngine`. Configuration is read from DNA once at game start and stored as an immutable struct.

```cpp
std::unique_ptr<NET_NetworkManager>
BL_ConvertNetwork(Scene* blenderScene, KX_Scene* kxScene)
{
    const GameNetworkData& dna = blenderScene->gm.net;
    if (dna.role == GAME_NET_ROLE_NONE) return nullptr; // Zero cost

    NET_NetworkConfig config;
    config.role                = static_cast<NetRole>(dna.role);
    config.port                = dna.port;
    config.maxClients          = dna.maxClients;
    config.tickRate            = dna.tickRate;
    config.sendRate            = dna.sendRate;
    config.maxEventsPerTick    = dna.maxEventsPerTick;
    config.maxEventsInitialSync = dna.maxEventsInitialSync;
    config.initialSyncWindow   = dna.initialSyncWindow;
    config.bandwidthBudget     = dna.bandwidthBudgetKBps * 1024;
    config.interpDelay         = dna.interpDelay;
    config.enableEncryption    = dna.enableEncryption;
    config.maxPeerRTT          = dna.maxPeerRTT;
    config.maxPeerQueueDepth   = dna.maxPeerQueueDepth;
    config.peerTimeout         = dna.peerTimeout;
    config.compress            = dna.compressPackets;
    std::memcpy(config.address, dna.address, sizeof(config.address));

    auto mgr = std::make_unique<NET_NetworkManager>(config);

    if (dna.autoStart) {
        if (config.role == NetRole::Server || config.role == NetRole::Host)
            mgr->StartServer();
        else
            mgr->ConnectToServer();
    }

    // Register replicated objects from scene
    LISTBASE_FOREACH(Object*, ob, &blenderScene->base) {
        if (!(ob->network_flags & OB_NET_REPLICATE)) continue;
        KX_GameObject* kxObj = kxScene->FindGameObject(ob);
        if (!kxObj) continue;
        mgr->GetReplication().Register(kxObj, ob->network_flags,
                                       ob->network_quant_profile,
                                       ob->network_relevance_radius);
    }

    return mgr;
}
```

### 5.2 Tick in the Game Loop

```cpp
// In KX_KetsjiEngine::NextFrame(), after physics, before render:
if (m_networkManager) {
    m_networkManager->Tick(m_frameTime);
}

void NET_NetworkManager::Tick(float dt) {
    m_arena.Reset();

    // 1. Connection state machine (timeouts, retries)
    m_connectionFSM.Update(dt);
    if (m_connectionFSM.state != ConnectionState::Connected) return;

    // 2. Poll provider (non-blocking, ~50–150µs)
    m_provider->Service(0);

    // 3. Process incoming (budget-capped, burst mode for initial sync)
    ProcessIncomingPackets();

    // 4. Apply interpolation EVERY frame (~3µs/object)
    m_replication.ApplyInterpolation(m_renderTime);

    // 5. Apply client prediction corrections (if any objects have it enabled)
    if (m_prediction) {
        m_prediction->ApplyCorrections(dt);
    }

    // 6. Send outgoing (only at send rate, bandwidth-budgeted)
    m_sendAccumulator += dt;
    m_replication.AdaptSendRate();
    if (m_sendAccumulator >= m_replication.GetEffectiveSendInterval()) {
        m_sendAccumulator -= m_replication.GetEffectiveSendInterval();
        m_replication.FlushOutgoing(m_config.bandwidthBudget
                                    * m_replication.GetEffectiveSendInterval());
    }

    // 7. Check peer health (server only)
    if (m_config.role == NetRole::Server || m_config.role == NetRole::Host) {
        CheckPeerHealth(dt);
    }

    // 8. Update stats (lock-free atomic writes)
    UpdateStats();
}
```

---

## 6. Performance Budget

All costs measured/estimated for 200 replicated objects, 16 connected peers, 60 FPS render, 20 Hz send rate.

| System | Cost | Notes |
|---|---|---|
| Arena Reset | ~1 ns | Single pointer store |
| Connection FSM Update | ~5 ns | One enum compare + float compare |
| ENet `Service()` poll | 50–150 µs | Non-blocking. Provider-bound. |
| ProcessIncoming (64 events) | ≤ 128 µs | Capped. Includes validation (~50ns/pkt). |
| Dirty-check (200 objects) | ~10 µs | FNV-1a + epoch (~50ns/obj). |
| Priority sort (`nth_element`) | ~2 µs | O(n). Only on send ticks (20 Hz). |
| Serialize snapshot (delta) | ~8 µs/obj | Quantization: bit-packing is cheaper than float copy. |
| `ApplyInterpolation` | ~3 µs/obj | Jitter buffer lookup is O(1) circular. |
| Client prediction | ~5 µs | Per predicted object. 0 if disabled. |
| Adaptive send rate | ~10 ns | Two float comparisons. |
| Peer health check | ~1 µs | 16 peers × ~60ns each. Server only. |
| GIL batch (Python RPCs) | ~105 µs | 50 RPCs worst case. |
| RPC dispatch (C++) | ~2 µs/call | Array index. |
| `UpdateStats` | ~20 ns | Atomic relaxed stores. |
| Logic Brick overhead | ~3 µs | 5 sensors + 5 actuators worst case. |

### 6.1 Total Tick() Cost Summary

| Scenario | Cost |
|---|---|
| Best case (no send tick, 0 events) | ~153 µs |
| Normal tick (send tick, 20 events, 50 dirty obj) | ~295 µs |
| Worst case (send tick, 64 events, 200 dirty obj) | ≤ 455 µs (guaranteed cap) |
| Burst mode (initial sync, 512 events) | ≤ 1.1 ms (first 2 seconds only) |

Burst mode exceeds the normal 0.3ms budget intentionally, for a maximum of 2 seconds at connection time. This is acceptable because the game is loading/syncing and the user expects a brief loading period.

---

## 7. Changes in the Blender Tree

Minimally invasive integration. The network module lives entirely within `source/gameengine/`. Blender-side changes are limited to DNA structs, RNA properties, and UI panels.

### 7.1 DNA Additions

**`source/blender/makesdna/DNA_scene_types.h`:**

```c
typedef struct GameNetworkData {
    short role;                 // GAME_NET_ROLE_NONE/SERVER/CLIENT/HOST
    short port;                 // Default: 7777
    short maxClients;           // Default: 16
    short tickRate;             // Default: 60
    short sendRate;             // Default: 20
    short maxEventsPerTick;     // Default: 64
    short maxEventsInitialSync; // Default: 512
    short bandwidthBudgetKBps;  // Default: 256
    short enableEncryption;     // Default: 0 (off)
    short autoStart;            // Default: 0 (off)
    short compressPackets;      // Default: 0 (reserved)
    short pad;
    float initialSyncWindow;    // Default: 2.0
    float interpDelay;          // Default: 0.1
    float maxPeerRTT;           // Default: 500.0
    float peerTimeout;          // Default: 10.0
    int   maxPeerQueueDepth;    // Default: 256
    char  address[64];          // Server address string
} GameNetworkData;  // 104 bytes, 8-byte aligned
```

**`source/blender/makesdna/DNA_object_types.h`:**

```c
// Added to Object struct:
short network_flags;            // OB_NET_REPLICATE | OB_NET_QUANTIZE | OB_NET_PREDICT
short network_quant_profile;    // QuantizationProfile enum (0–3)
float network_relevance_radius; // Priority distance weight
float network_sector_origin[3]; // Quantization sector origin

// Flag definitions:
#define OB_NET_REPLICATE  (1 << 0)
#define OB_NET_QUANTIZE   (1 << 1)
#define OB_NET_PREDICT    (1 << 2)
```

**`source/blender/makesdna/DNA_actuator_types.h`:** `bNetworkActuator` struct (see §4.5).

**`source/blender/makesdna/DNA_sensor_types.h`:** `bNetworkEventSensor` struct (see §4.5).

### 7.2 RNA Properties

**`source/blender/makesrna/intern/rna_scene.cc`:** `rna_def_scene_game_network()` exposes all `GameNetworkData` fields in the Network panel.

**`source/blender/makesrna/intern/rna_object.cc`:** Network section with replicate checkbox, quantization profile dropdown, prediction checkbox, relevance radius slider, sector origin vector.

**`source/blender/makesrna/intern/rna_actuator.cc`:** Network actuator RNA definition (mode enum, all mode-specific parameters).

**`source/blender/makesrna/intern/rna_sensor.cc`:** Network Event sensor RNA definition (mode enum, subject filter).

### 7.3 UI Panels

**`release/.../bl_ui/properties_game.py`:**

- `NET_PT_network` — Main panel: role, address, port, max clients, auto start
- `NET_PT_network_rates` — Sub-panel: tick rate, send rate, bandwidth budget
- `NET_PT_network_interpolation` — Sub-panel: interpolation delay slider
- `NET_PT_network_advanced` — Collapsed by default: max events/tick, burst budget, initial sync window, encryption checkbox, peer health thresholds

**`release/.../bl_ui/properties_object.py`:**

- `NET_PT_object_network` — Replicate checkbox, quantization profile dropdown, prediction checkbox, priority slider, relevance radius, sector origin

---

## 8. Python API — `bge.network`

| Function / Constant | Description |
|---|---|
| `net.start_server(port, address, max_clients, tick_rate, send_rate)` | Start server. Returns bool. |
| `net.connect(address, port)` | Connect to server as client. Returns bool. |
| `net.disconnect()` | Orderly disconnect. |
| `net.replicate(obj, transform, rotation, scale, quantize, owner_id)` | Register a `KX_GameObject` for replication. Returns `NetObjectID` (int). |
| `net.unreplicate(net_id)` | Unregister an object. Frees the `NetObjectID` for reuse. |
| `net.register_rpc(name, handler, target, reliable)` | Register a Python callable as RPC handler. Returns `RPCID` (int). |
| `net.call_rpc(name_or_id, obj_id, args, target, peer_id)` | Call an RPC. `args` is a Python list. |
| `net.get_stats()` | Returns dict: `bytes_sent/recv_per_sec`, `ping_ms`, `connected_peers`, `replicated_objects`, `packet_loss_pct`, `jitter_ms`, `arena_overflows`, `effective_send_rate`, `queue_depth`. |
| `net.get_connection_state()` | Returns string: `'disconnected'`, `'connecting'`, `'connected'`, `'disconnecting'`. |
| `net.set_object_priority(net_id, priority)` | Override base priority for an object (default 1.0). Higher = sent more often. |
| `net.set_interp_delay(seconds)` | Set interpolation delay (0.05–0.5). |
| `net.set_quant_profile(net_id, profile)` | Set quantization profile for object (0–3). |
| `net.enable_prediction(net_id, enabled)` | Enable/disable client prediction for an object. |
| `net.SERVER` / `ALL_CLIENTS` / `OWNER` / `OTHERS` | `RPCTarget` constants. |
| `net.QUANT_STANDARD` / `QUANT_HIGH` / `QUANT_WORLD` / `QUANT_NONE` | Quantization profile constants. |

---

## 9. Integration Checklist

| # | Action | Target File |
|---|---|---|
| **1** | Add `GameNetworkData` struct + `net` field to `GameData` | `source/blender/makesdna/DNA_scene_types.h` |
| **2** | Add `OB_NET_*` flags + quant profile + sector origin + predict flag to `Object` | `source/blender/makesdna/DNA_object_types.h` |
| **3** | Add `bNetworkActuator` to actuator types | `source/blender/makesdna/DNA_actuator_types.h` |
| **4** | Add `bNetworkEventSensor` to sensor types | `source/blender/makesdna/DNA_sensor_types.h` |
| **5** | Add `rna_def_scene_game_network()` with all fields | `source/blender/makesrna/intern/rna_scene.cc` |
| **6** | Add network properties to UPBGE block of `Object` | `source/blender/makesrna/intern/rna_object.cc` |
| **7** | Add RNA definition for Network actuator | `source/blender/makesrna/intern/rna_actuator.cc` |
| **8** | Add RNA definition for Network Event sensor | `source/blender/makesrna/intern/rna_sensor.cc` |
| **9** | Add `NET_PT_*` panels + Advanced sub-panel | `release/.../bl_ui/properties_game.py` |
| **10** | Add `NET_PT_object_network` panel | `release/.../bl_ui/properties_object.py` |
| **11** | `add_subdirectory(Network)` with all NET_ files | `source/gameengine/CMakeLists.txt` |
| **12** | Link `ge_network` to `blenderplayer` target | `source/creator/CMakeLists.txt` |
| **13** | Add `BL_ConvertNetwork()` + logic brick conversion cases | `source/gameengine/Converter/BL_Converter.cpp` |
| **14** | Store `NET_NetworkManager` in `KX_KetsjiEngine`, call `Tick()` | `source/gameengine/Ketsji/KX_KetsjiEngine.cpp` |
| **15** | Register `bge.network` module + all functions | `source/gameengine/Ketsji/KX_PythonInit.cpp` |
| **16** | `make makesdna` → `make blender` | Terminal |

**Summary:** 10 Blender-tree files modified. 20 gameengine files (18 in Network/ + 2 logic bricks in GameLogic/).

---

## 10. Gameplay Validation Examples

The network module guarantees data integrity. Gameplay validation is the game developer's responsibility.

**Python (server-side RPC handler):**

```python
import bge

@bge.network.register_rpc("player_move", target=bge.network.SERVER, reliable=False)
def validate_move(peer_id, obj_id, position, dt):
    obj = bge.network.get_replicated_object(obj_id)
    max_speed = obj["max_speed"]
    distance = (position - obj.worldPosition).length
    if distance > max_speed * dt * 1.1:  # 10% tolerance
        return  # Reject: speed hack
    obj.worldPosition = position
```

**Logic Bricks (chained validation):**

```
Sensors:                Controllers:          Actuators:
[Network Event      ]---[Python: validate]--- [Network: SEND_RPC]
  Mode: RPC_RECEIVED                            Subject: "move_rejected"
  Subject: "player_move"                        Target: OWNER

# Python controller checks property "max_speed" vs received distance
# If invalid, activates the rejection RPC actuator
# If valid, sets object position directly
```

---

## 11. Known Limitations & Future Candidates

### 11.1 Documented Limitations

| Item | Reason | Threshold |
|---|---|---|
| Main thread blocking | `KX_GameObject` is not thread-safe. ~450µs worst case is <3% of frame budget at 200 objects. | Reconsider at 800+ objects |
| Dirty check O(N) for static objects | Event-based approach requires intercepting every transform mutation — too invasive. ~10µs at 200 objects. | Reconsider at 500+ static objects |
| No spatial interest management | O(N) distance check costs ~1µs at 200 objects. `DistanceWeight()` is a configurable function. | Reconsider at 500+ objects |
| Prediction is simple | Position/rotation only. No physics rollback, no weapon prediction, no lag compensation. | Game-specific extensions |

### 11.2 Future Candidates

| Feature | Complexity | Trigger |
|---|---|---|
| Dedicated network thread with lock-free queues | High | 500+ replicated objects |
| Event-based dirty list (`OnTransformChange`) | High | 500+ static objects in replication |
| Adaptive jitter buffer (measure actual jitter) | Medium | Competitive game feedback |
| Spatial interest management (quadtree/grid) | Medium | 500+ replicated objects |
| Physics rollback for prediction | Very High | Fighting game / precision shooter |
| Batch snapshot compression (LZ4) | Low | High bandwidth environments |
| DTLS encryption (replacing custom encryption) | Medium | Security-critical deployments |
