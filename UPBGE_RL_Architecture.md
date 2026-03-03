# RL — Reinforcement Learning Module for UPBGE
**Version:** 1.0  
**Status:** Specification  
**Author:** UPBGE Core Team  

---

## 0. Integration with the Network Module

The RL module reuses the Network module infrastructure to avoid duplication
and ensure architectural consistency across the engine.

| Topic | Decision | Rationale |
|-------|----------|-----------|
| ENet transport | Use `NET_NetworkManager` directly | Reuses provider abstraction, connection FSM, peer health monitoring |
| Lifecycle | Explicit ownership in `KX_KetsjiEngine`, no singleton | Deterministic init/shutdown, consistent with Network module |
| Memory | `RL_Buffer` lock-free SPSC ring buffer for hot path | `NET_ArenaAllocator` is frame-scoped; RL needs persistent cross-frame buffer |
| File prefix | `RL_` | Consistent with `NET_` convention |
| RPC transport | `NET_RPCDispatcher` via named RPCs | Reliable delivery, replay protection, size validation for free |
| Python guards | All Python code under `#ifdef WITH_PYTHON` | Consistent with all other UPBGE modules |

### 0.1 How RL Uses NET_NetworkManager

- The **master instance** starts `NET_NetworkManager` in server mode on loopback
- **Worker instances** start `NET_NetworkManager` in client mode, connecting to master
- All RL messages travel as **named RPCs** via `NET_RPCDispatcher`
- Connection FSM, peer health, packet validation, and reliable delivery are all
  handled by `NET_NetworkManager` — RL reimplements none of this

**Without `WITH_PYTHON`:** the engine compiles, simulates, collects transitions,
and sends them via RPC — but `IsReady()` returns false, `GetAction()` returns a
zero action, and the training thread does not exist. The module is inert but safe.

---

## 1. Overview

`RL_` is a generalized reinforcement learning module integrated natively into UPBGE.
It allows any physical game object to serve as an RL agent, training neural network
policies entirely inside the engine using multiple parallel headless instances —
without external simulators, cloud services, or NVIDIA GPU.

### 1.1 Design Goals

| Priority | Goal |
|----------|------|
| 1 | **Performance** — parallel headless workers, lock-free local buffer, no GIL blocking game loop |
| 2 | **Generality** — works with any physics object, not tied to specific agent types |
| 3 | **Scalability** — auto-detects CPU count, launches N workers automatically |
| 4 | **Consistency** — lifecycle, transport, and conventions aligned with Network module |
| 5 | **Usability** — users define tasks in Python without touching engine code |
| 6 | **Portability** — exports to ONNX/TorchScript for deployment on any hardware |

### 1.2 Non-Goals

- Does **not** implement physics simulation (delegated to Bullet via UPBGE)
- Does **not** bundle a specific RL algorithm — users bring their own or use defaults
- Does **not** require internet access, cloud services, or NVIDIA GPU
- Does **not** reimplement ENet, connection management, or packet validation
- Training features are **not available** when compiled without `WITH_PYTHON`

---

## 2. Architecture

### 2.1 Multi-Process Overview

Training runs as a fleet of N headless UPBGE processes (workers) coordinated by
one master process (instance 0). The master hosts the policy network, trainer,
and curriculum manager. Workers simulate the environment and send experience.
All communication uses `NET_NetworkManager` on loopback.

```
+-------------------------------------------------------------------------+
|  Instance 0  --  MASTER  (blenderplayer --headless)                     |
|                                                                         |
|  +--------------+   +----------------------+   +---------------------+  |
|  |  RL_Manager  |   |  NET_NetworkManager  |   |  Python Trainer     |  |
|  |  (C++ core)  | > |  (server, loopback)  |   |  (background thread)|  |
|  |              |   |  NET_RPCDispatcher   | > |  gradient updates   |  |
|  +--------------+   +----------------------+   +---------------------+  |
|         ^                      ^               [#ifdef WITH_PYTHON]   v  |
|   local sim loop         receives batches         broadcast weights    |
|   local RL_Buffer        from all workers         via RPC to workers   |
+-------------------------------------------------------------------------+
                   NET_NetworkManager (loopback, 127.0.0.1)
                   RPCs over NET_ENetProvider
          +-----------------+-----------------+-----------------+
          v                 v                 v                 v
  +--------------+  +--------------+  +--------------+  +--------------+
  |  Instance 1  |  |  Instance 2  |  |  Instance 3  |  |  Instance N  |
  |  WORKER      |  |  WORKER      |  |  WORKER      |  |  WORKER      |
  |  (headless)  |  |  (headless)  |  |  (headless)  |  |  (headless)  |
  |  Bullet sim  |  |  Bullet sim  |  |  Bullet sim  |  |  Bullet sim  |
  |  RL_Manager  |  |  RL_Manager  |  |  RL_Manager  |  |  RL_Manager  |
  |  RL_Buffer   |  |  RL_Buffer   |  |  RL_Buffer   |  |  RL_Buffer   |
  |  NET (client)|  |  NET (client)|  |  NET (client)|  |  NET (client)|
  +--------------+  +--------------+  +--------------+  +--------------+
```

### 2.2 Instance Roles

| Role | Count | Responsibilities |
|------|-------|-----------------|
| **Master** (instance 0) | 1 | NET server, policy network, gradient updates, curriculum, weight broadcast |
| **Worker** (instance 1..N) | Auto (cpu_count - 1) | Bullet simulation, transition collection, send batches to master, apply received weights |

### 2.3 Thread Model

**Master:**

| Thread | Responsibility | Blocking |
|--------|---------------|---------|
| Game thread (C++) | Sim step, local inference, RL_Buffer write | No |
| NET tick thread | `NET_NetworkManager::Tick()` — receive batches, dispatch RPCs | Bounded (< 0.3 ms) |
| Training thread (Python) | Gradient updates, curriculum advance | Yes — `#ifdef WITH_PYTHON` only |

**Worker:**

| Thread | Responsibility | Blocking |
|--------|---------------|---------|
| Game thread (C++) | Sim step, inference from local policy copy, RL_Buffer write | No |
| NET tick thread | `NET_NetworkManager::Tick()` — flush buffer to master, receive weights | Bounded |

The game thread **never waits** on the network or training thread on either role.

### 2.4 RPC Protocol

RL communication uses named RPCs registered via `NET_RPCDispatcher`. This provides
reliable/ordered delivery, replay protection, and size validation at no extra cost.

#### Registered RPCs

| RPC Name | Direction | Delivery | Payload |
|----------|-----------|---------|---------|
| `rl.experience_batch` | Worker → Master | Reliable, ordered | `RL_ExperienceBatch` header + `RL_Transition[]` bytes |
| `rl.weight_update` | Master → Worker | Reliable, ordered | `RL_WeightUpdate` header + state_dict bytes |
| `rl.curriculum_update` | Master → Worker | Reliable, ordered | `RL_CurriculumUpdate` struct |
| `rl.worker_stats` | Worker → Master | Unreliable | `RL_WorkerStats` struct |

`rl.weight_update` payload is only populated when `WITH_PYTHON` is enabled.
The RPC slot is registered unconditionally so workers handle it gracefully.

#### Message Structs

```cpp
// RL_Protocol.h

// Worker -> Master: batch of transitions ready for training
struct RL_ExperienceBatch {
    uint8_t  worker_id;
    uint32_t num_transitions;
    uint32_t byte_size;
    // Followed by raw RL_Transition[num_transitions] bytes
};

// Master -> Worker: serialized policy weights after a training step
// Payload only sent when WITH_PYTHON is enabled
struct RL_WeightUpdate {
    uint32_t training_step;
    uint32_t byte_size;
    // Followed by state_dict bytes (torch.save format)
};

// Master -> Worker: curriculum stage advanced
struct RL_CurriculumUpdate {
    uint8_t  new_task_id;
    uint32_t global_episode;
    float    avg_reward;
};

// Worker -> Master: per-episode stats (unreliable, monitoring only)
struct RL_WorkerStats {
    uint8_t  worker_id;
    uint32_t episode;
    float    total_reward;
    float    episode_steps;
    uint8_t  task_id;
};
```

### 2.5 Synchronization Strategy

Workers operate fully asynchronously — they never wait for weight updates:

```
Worker:
  frames 0..N-1:  simulate, write transitions to local RL_Buffer
  frame N:        drain RL_Buffer -> call_rpc("rl.experience_batch")
  frames N+1..:   continue simulating with current (possibly stale) weights
                  (stale weights acceptable for off-policy algorithms: SAC, TD3)

Master [#ifdef WITH_PYTHON]:
  NET tick:       OnExperienceBatchReceived -> append to replay buffer
  training thread: sample batch -> gradient step -> call_rpc("rl.weight_update")
  workers:        OnWeightUpdateReceived -> deserialize -> update local policy copy
```

For on-policy algorithms (PPO), an optional barrier sync is available via
`sync_on_policy = true` in `RL_Config`. When enabled, the master waits for all
workers to signal episode end before broadcasting weights and starting the next
rollout. Implemented as an additional `rl.episode_done` RPC (Worker → Master).

---

## 3. Lifecycle

`RL_Manager` follows the same explicit-lifetime pattern as `NET_NetworkManager`:
owned by `KX_KetsjiEngine`, no singleton, no static state, deterministic shutdown.

### 3.1 KX_KetsjiEngine Integration

```cpp
// KX_KetsjiEngine.h  (minimal addition)
class KX_KetsjiEngine {
    std::unique_ptr<NET_NetworkManager> m_networkManager;  // existing
    std::unique_ptr<RL_Manager>         m_rlManager;       // null if RL disabled
};

// KX_KetsjiEngine.cpp
void KX_KetsjiEngine::ConvertScene(KX_Scene *scene) {
    // ... existing code ...
    if (scene->GetRLConfig().enabled) {
        m_rlManager = std::make_unique<RL_Manager>(
            scene->GetRLConfig(), m_networkManager.get());
        m_rlManager->Initialize();
    }
}

void KX_KetsjiEngine::NextFrame() {
    // ... existing code ...
    // RL_Manager::Tick() runs after NET_NetworkManager::Tick()
    // so incoming RPCs (weight updates) are already dispatched
    if (m_rlManager)
        m_rlManager->Tick(m_deltaTime);
}

void KX_KetsjiEngine::EndGame() {
    // Deterministic shutdown order: RL before Network
    if (m_rlManager) {
        m_rlManager->Shutdown();
        m_rlManager.reset();
    }
    // ... existing network shutdown ...
}
```

`RL_Manager` receives a **non-owning pointer** to `NET_NetworkManager`.
It never outlives it. Shutdown order is enforced by `KX_KetsjiEngine`.

---

## 4. C++ Core Layer

### 4.1 File Structure

```
gameengine/AI/
├── CMakeLists.txt
├── RL_Types.h          # POD structs, constants, enums
├── RL_Protocol.h       # RPC message structs
├── RL_Buffer.h         # Lock-free SPSC ring buffer (header-only)
├── RL_Manager.h        # Main manager interface
├── RL_Manager.cpp      # C++ implementation (always) + pybind11 (#ifdef WITH_PYTHON)
└── RL_Launcher.h/.cpp  # Worker process spawner
```

### 4.2 RL_Types.h

```cpp
#pragma once
#include <atomic>
#include <cstdint>

// Maximum dimensions — avoids heap allocation in hot path
constexpr int RL_MAX_OBS_DIM    = 64;
constexpr int RL_MAX_ACTION_DIM = 16;

struct RL_Observation {
    float data[RL_MAX_OBS_DIM];
    int   dim;
};

struct RL_Action {
    float data[RL_MAX_ACTION_DIM];
    int   dim;
};

// Fixed-size — safe to memcpy directly into RPC payload
struct RL_Transition {
    RL_Observation obs;
    RL_Observation next_obs;
    RL_Action      action;
    float          reward;
    bool           done;
    uint8_t        task_id;
    uint8_t        worker_id;
    uint8_t        _pad[1];   // align to 4 bytes
};

struct RL_EpisodeStats {
    int   episode;
    int   steps;
    float total_reward;
    float avg_reward;     // exponential moving average
    int   task_id;
    int   worker_id;
    float inference_ms;
};

struct RL_Config {
    bool     enabled;
    int      obs_dim;
    int      action_dim;
    int      buffer_capacity;       // local ring buffer size; must be power of 2
    int      worker_id;             // 0 = master; auto-set from --rl-worker-id CLI arg
    uint16_t master_port;           // ENet port for RL RPCs (default 7700)
    int      flush_every_n_steps;   // frames between buffer flushes (default 100)
    bool     sync_on_policy;        // barrier sync for PPO (default false)
    float    ema_alpha;             // avg_reward EMA alpha (default 0.05)
};
```

### 4.3 RL_Buffer.h

```cpp
#pragma once
#include <atomic>
#include <cstddef>

// Single-Producer Single-Consumer lock-free ring buffer.
// Producer: game thread. Consumer: NET tick thread (batch flush).
// Cache-line aligned to prevent false sharing between producer and consumer.
template<typename T, std::size_t Capacity>
class RL_Buffer {
    static_assert((Capacity & (Capacity - 1)) == 0,
                  "Capacity must be a power of 2");
public:
    // Called from game thread — no locks, no alloc, no blocking
    bool Push(const T &item) {
        const size_t head = m_head.load(std::memory_order_relaxed);
        m_data[head] = item;
        m_head.store((head + 1) & (Capacity - 1),
                     std::memory_order_release);
        return true;
    }

    // Called from NET tick thread
    bool Pop(T &item) {
        const size_t tail = m_tail.load(std::memory_order_relaxed);
        const size_t head = m_head.load(std::memory_order_acquire);
        if (tail == head) return false;
        item = m_data[tail];
        m_tail.store((tail + 1) & (Capacity - 1),
                     std::memory_order_release);
        return true;
    }

    // Batch drain for RPC flush — avoids per-item call overhead
    int DrainTo(T *out, int max_count) {
        int n = 0;
        while (n < max_count && Pop(out[n])) ++n;
        return n;
    }

    size_t Size() const {
        const size_t h = m_head.load(std::memory_order_acquire);
        const size_t t = m_tail.load(std::memory_order_acquire);
        return (h - t) & (Capacity - 1);
    }

private:
    alignas(64) T                        m_data[Capacity];
    alignas(64) std::atomic<std::size_t> m_head{0};
    alignas(64) std::atomic<std::size_t> m_tail{0};
};
```

### 4.4 RL_Manager.h

```cpp
#pragma once
#include "RL_Types.h"
#include "RL_Buffer.h"
#include "../Network/NET_NetworkManager.h"
#include <atomic>
#include <memory>

#ifdef WITH_PYTHON
// Use UPBGE's standard Python include wrapper, not <Python.h> directly
#  include "EXP_Python.h"
#endif

class RL_Launcher;

class RL_Manager {
public:
    // Non-owning pointer to NET_NetworkManager — must outlive RL_Manager
    explicit RL_Manager(const RL_Config &config, NET_NetworkManager *net);
    ~RL_Manager();

    RL_Manager(const RL_Manager &) = delete;
    RL_Manager &operator=(const RL_Manager &) = delete;

    void Initialize();
    void Tick(float dt);   // called from KX_KetsjiEngine::NextFrame()
    void Shutdown();

    // --- Hot path (game thread, every frame) ---
    // Always available regardless of WITH_PYTHON.
    // Without Python: GetAction() returns zero action, IsReady() returns false.
    void      StoreTransition(const RL_Transition &t);
    RL_Action GetAction(const RL_Observation &obs, int task_id);
    bool      IsReady()  const;
    bool      IsMaster() const;
    int       WorkerID() const;

    // --- Episode management (always available) ---
    void ResetEpisode(int task_id);
    int  GetCurrentTask()  const;
    int  GetEpisodeCount() const;
    int  GetTotalSteps()   const;  // global across all workers (master only)

    // --- Stats (always available) ---
    RL_EpisodeStats GetLastEpisodeStats() const;
    float           GetInferenceMs()      const;
    int             GetAliveWorkers()     const;  // master only

#ifdef WITH_PYTHON
    // Training interface — master only, requires PyTorch via Python.
    // Not compiled when WITH_PYTHON is disabled.
    PyObject *SampleBatch(int batch_size);
    void      SetPolicyCallable(PyObject *fn);
    void      BroadcastWeights(PyObject *state_dict);
    void      NotifyTrainStep(float loss);
#endif  // WITH_PYTHON

private:
    // RPC handlers — registered unconditionally in Initialize()
    // Called from NET tick thread
    void OnExperienceBatchReceived(PeerID peer, const void *data, uint32_t size);
    void OnCurriculumUpdateReceived(PeerID peer, const void *data, uint32_t size);
    void OnWorkerStatsReceived(PeerID peer, const void *data, uint32_t size);

#ifdef WITH_PYTHON
    // Weight deserialization requires Python/PyTorch
    void OnWeightUpdateReceived(PeerID peer, const void *data, uint32_t size);
#endif

    // Drains local RL_Buffer and sends via rl.experience_batch RPC
    void FlushBufferToMaster();

    RL_Config           m_config;
    NET_NetworkManager *m_net;        // non-owning
    bool                m_initialized{false};
    bool                m_ready{false};

    // Local ring buffer: game thread -> FlushBufferToMaster()
    RL_Buffer<RL_Transition, 65536> m_localBuffer;

    // Worker launcher (master only)
    std::unique_ptr<RL_Launcher> m_launcher;

#ifdef WITH_PYTHON
    // Policy callable — null until SetPolicyCallable() is called.
    // Not present in builds without Python.
    PyObject *m_policy{nullptr};
#endif

    // Atomics — safe to read from any thread
    std::atomic<int>   m_totalSteps{0};
    std::atomic<int>   m_episodeCount{0};
    std::atomic<int>   m_currentTask{0};
    std::atomic<float> m_avgReward{0.0f};
    std::atomic<float> m_inferenceMs{0.0f};
    std::atomic<int>   m_trainStep{0};
};
```

### 4.5 RL_Manager.cpp — WITH_PYTHON Structure

C++ logic is unconditionally compiled. Python bindings live in a guarded block
at the bottom of the file, following the pattern of `KX_PythonInit.cpp`.

```cpp
// RL_Manager.cpp

#include "RL_Manager.h"
#include "RL_Protocol.h"
#include "RL_Launcher.h"

// ==========================================================================
// C++ implementation — always compiled regardless of WITH_PYTHON
// ==========================================================================

void RL_Manager::Initialize() {
    m_net->RegisterRPC("rl.experience_batch",
        [this](PeerID p, const void *d, uint32_t s) {
            OnExperienceBatchReceived(p, d, s); },
        RPCTarget::SERVER, /*reliable=*/true);

    m_net->RegisterRPC("rl.curriculum_update",
        [this](PeerID p, const void *d, uint32_t s) {
            OnCurriculumUpdateReceived(p, d, s); },
        RPCTarget::ALL_CLIENTS, /*reliable=*/true);

    m_net->RegisterRPC("rl.worker_stats",
        [this](PeerID p, const void *d, uint32_t s) {
            OnWorkerStatsReceived(p, d, s); },
        RPCTarget::SERVER, /*reliable=*/false);

    // Slot registered unconditionally so workers handle it gracefully;
    // actual payload is only sent when WITH_PYTHON is enabled
    m_net->RegisterRPC("rl.weight_update",
#ifdef WITH_PYTHON
        [this](PeerID p, const void *d, uint32_t s) {
            OnWeightUpdateReceived(p, d, s); },
#else
        nullptr,
#endif
        RPCTarget::ALL_CLIENTS, /*reliable=*/true);

    if (IsMaster() && m_config.enabled)
        m_launcher = std::make_unique<RL_Launcher>(m_config.master_port);

    m_initialized = true;
}

void RL_Manager::Tick(float dt) {
    if (!m_initialized) return;
    m_totalSteps.fetch_add(1, std::memory_order_relaxed);
    if (!IsMaster() &&
        m_totalSteps % m_config.flush_every_n_steps == 0)
    {
        FlushBufferToMaster();
    }
}

void RL_Manager::StoreTransition(const RL_Transition &t) {
    m_localBuffer.Push(t);
}

RL_Action RL_Manager::GetAction(const RL_Observation &obs, int task_id) {
#ifdef WITH_PYTHON
    if (m_policy && m_ready) {
        // Acquire GIL, call policy callable, release GIL
        // Convert returned tensor to RL_Action
    }
#endif
    // Fallback: zero action (no policy loaded, or Python unavailable)
    RL_Action action{};
    action.dim = m_config.action_dim;
    return action;
}

void RL_Manager::FlushBufferToMaster() {
    static RL_Transition scratch[512];  // stack-allocated — no heap
    const int n = m_localBuffer.DrainTo(scratch, 512);
    if (n == 0) return;

    RL_ExperienceBatch header;
    header.worker_id       = static_cast<uint8_t>(m_config.worker_id);
    header.num_transitions = static_cast<uint32_t>(n);
    header.byte_size       = static_cast<uint32_t>(n * sizeof(RL_Transition));

    m_net->CallRPC("rl.experience_batch", /*obj_id=*/0,
                   &header, sizeof(header),
                   scratch, n * sizeof(RL_Transition),
                   RPCTarget::SERVER, /*reliable=*/true);
}

void RL_Manager::OnExperienceBatchReceived(
    PeerID peer, const void *data, uint32_t size)
{
#ifdef WITH_PYTHON
    // Parse header + append transitions to Python-side replay buffer
#else
    (void)peer; (void)data; (void)size;
#endif
}

void RL_Manager::OnCurriculumUpdateReceived(
    PeerID peer, const void *data, uint32_t size)
{
    const auto *msg = static_cast<const RL_CurriculumUpdate *>(data);
    m_currentTask.store(msg->new_task_id, std::memory_order_release);
}

void RL_Manager::OnWorkerStatsReceived(
    PeerID peer, const void *data, uint32_t size)
{
    // Update per-worker stats for monitoring — no Python needed
}

// ==========================================================================
#ifdef WITH_PYTHON
// Python-dependent implementation.
// Follows the pattern of KX_PythonInit.cpp and other UPBGE Python modules.
// ==========================================================================

#include "EXP_Python.h"
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>

namespace py = pybind11;

void RL_Manager::OnWeightUpdateReceived(
    PeerID peer, const void *data, uint32_t size)
{
    // Deserialize state_dict bytes and update local policy weights
}

PyObject *RL_Manager::SampleBatch(int batch_size)
{
    // Sample from Python-side replay buffer and return as tensor tuple
    return nullptr;
}

void RL_Manager::SetPolicyCallable(PyObject *fn)
{
    Py_XDECREF(m_policy);
    m_policy = fn;
    Py_XINCREF(m_policy);
    m_ready = (m_policy != nullptr);
}

void RL_Manager::BroadcastWeights(PyObject *state_dict)
{
    // Serialize state_dict -> bytes -> RPC to all workers
}

void RL_Manager::NotifyTrainStep(float loss)
{
    m_trainStep.fetch_add(1, std::memory_order_relaxed);
}

// pybind11 module: exposes RL_Manager to Python as rl._bridge
PYBIND11_MODULE(rl_bridge, m) {
    m.doc() = "UPBGE RL module C++ bridge";

    py::class_<RL_Manager>(m, "RLManager")
        .def("initialize",          &RL_Manager::Initialize)
        .def("is_ready",            &RL_Manager::IsReady)
        .def("is_master",           &RL_Manager::IsMaster)
        .def("worker_id",           &RL_Manager::WorkerID)
        .def("get_current_task",    &RL_Manager::GetCurrentTask)
        .def("get_episode_count",   &RL_Manager::GetEpisodeCount)
        .def("get_total_steps",     &RL_Manager::GetTotalSteps)
        .def("get_alive_workers",   &RL_Manager::GetAliveWorkers)
        .def("sample_batch",        &RL_Manager::SampleBatch)
        .def("set_policy_callable", &RL_Manager::SetPolicyCallable)
        .def("broadcast_weights",   &RL_Manager::BroadcastWeights)
        .def("notify_train_step",   &RL_Manager::NotifyTrainStep);
}

// ==========================================================================
#endif  // WITH_PYTHON
// ==========================================================================
```

### 4.6 CMakeLists.txt

```cmake
# gameengine/AI/CMakeLists.txt

set(INC
    .
    ../Network
    ../Ketsji
    ../Common
    ${CMAKE_SOURCE_DIR}/source/blender/makesdna
)

set(SRC
    RL_Manager.cpp
    RL_Launcher.cpp
    RL_Types.h
    RL_Protocol.h
    RL_Buffer.h
    RL_Manager.h
    RL_Launcher.h
)

if(WITH_PYTHON)
    list(APPEND INC ${PYTHON_INCLUDE_DIRS})
    # Install Python package alongside other UPBGE Python modules
    install(
        DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/python/rl
        DESTINATION ${BLENDER_SCRIPTS_DIR}/modules
    )
endif()

blender_add_lib(ge_rl "${SRC}" "${INC}" "" "")
```

### 4.7 RL_Launcher.h

```cpp
#pragma once
#include <vector>
#include <string>

struct RL_WorkerInfo {
    int         worker_id;
    uint16_t    port;
    std::string blend_file;
    int         pid;
    bool        alive;
};

class RL_Launcher {
public:
    explicit RL_Launcher(uint16_t base_port);

    // Spawns (cpu_count - 1) workers, or max_workers if specified (-1 = auto).
    // Injects --rl-worker-id and --rl-master-port into each blenderplayer call.
    // Returns number of workers actually spawned.
    int  SpawnWorkers(const std::string &blend_file, int max_workers = -1);

    void ShutdownWorkers();
    void RestartWorker(int worker_id);  // restart a crashed worker
    int  AliveCount() const;

    const std::vector<RL_WorkerInfo> &GetWorkers() const;

private:
    int  SpawnOne(int worker_id, const std::string &blend_file);
    int  DetectCPUCount() const;

    std::vector<RL_WorkerInfo> m_workers;
    uint16_t                   m_basePort;
};
```

Workers are identified by CLI args injected automatically — the user never types these:

```bash
blenderplayer --headless scene.blend \
    --rl-worker-id 2                  \
    --rl-master-port 7700
```

`RL_Manager::Initialize()` reads `--rl-worker-id` to determine its role.
If the argument is absent, the instance is master (worker_id = 0).

## 5. Python Layer

The entire `gameengine/AI/python/rl/` package is only installed when `WITH_PYTHON`
is enabled, mirroring the handling of `bge`, `bgl`, and other UPBGE Python modules.

### 5.1 File Structure

```
gameengine/AI/python/
└── rl/                  # Installed only when WITH_PYTHON=ON
    ├── __init__.py      # Public API
    ├── trainer.py       # Training loop (master only, background thread)
    ├── policy.py        # Default actor-critic policy (nn.Module)
    ├── curriculum.py    # Curriculum manager
    ├── task.py          # RLTask base class + task registry
    ├── weights.py       # state_dict <-> bytes serialization
    ├── launcher.py      # Python wrapper for RL_Launcher pybind11 binding
    └── tasks/
        ├── __init__.py
        ├── hover.py         # Reference implementation: hover
        ├── waypoint.py      # Reference implementation: waypoint navigation
        ├── obstacle.py      # Reference implementation: obstacle avoidance
        └── tracking.py      # Reference implementation: object tracking
```

### 5.2 Public API

```python
# rl/__init__.py
from rl._bridge    import initialize, shutdown, is_ready, is_master, worker_id
from rl._bridge    import get_action, store, reset
from rl._bridge    import save, load, export_onnx, export_torchscript
from rl.trainer    import start_training, stop_training
from rl.launcher   import start_workers, stop_workers, worker_count
from rl.curriculum import set_curriculum, current_task, current_stage
from rl.task       import RLTask, register_task
from rl.policy     import RLDefaultPolicy
from rl._stats     import get_stats, on_episode_end, on_stage_advance
```

### 5.3 RLTask Base Class

```python
# rl/task.py

class RLTask:
    """
    Base class for all RL tasks.
    Subclass to define a new task. The same class runs on master and all workers.
    No engine code needs to be modified to add a task.
    """
    task_id:    int = -1
    task_name:  str = "unnamed"
    obs_dim:    int = 0
    action_dim: int = 0

    def get_observation(self, obj) -> list[float]:
        """Return flat float list of length obs_dim. Called every frame."""
        raise NotImplementedError

    def compute_reward(self, obj) -> tuple[float, bool]:
        """Return (reward: float, done: bool). Called every frame."""
        raise NotImplementedError

    def reset(self, obj) -> None:
        """Reset object and internal task state for a new episode."""
        raise NotImplementedError

    def apply_action(self, obj, action: list[float]) -> None:
        """Apply action vector to UPBGE game object physics."""
        raise NotImplementedError

    def on_episode_end(self, stats: dict) -> None:
        """Optional hook: called at episode end on the local instance."""
        pass


def register_task(id: int, name: str):
    """Decorator to register a task class with the curriculum system."""
    def decorator(cls):
        cls.task_id   = id
        cls.task_name = name
        _registry[id] = cls
        return cls
    return decorator

_registry: dict[int, type] = {}
```

### 5.4 Trainer

```python
# rl/trainer.py
import threading
import torch
import rl

class RLTrainer:
    """
    Runs on master (worker_id == 0) in a background thread.
    Samples from the aggregated replay buffer populated by worker RPCs,
    performs gradient updates, and broadcasts updated weights via RPC.
    Subclass and override _compute_loss() for custom algorithms (PPO, SAC, TD3).
    """

    def __init__(self, policy, optimizer, batch_size: int = 256,
                 train_every_n_steps: int = 1000,
                 broadcast_every_n_steps: int = 500):
        self.policy                  = policy
        self.optimizer               = optimizer
        self.batch_size              = batch_size
        self.train_every_n_steps     = train_every_n_steps
        self.broadcast_every_n_steps = broadcast_every_n_steps
        self._thread                 = None
        self._running                = False

    def start(self):
        assert rl.is_master(), "RLTrainer must only run on master (worker_id=0)"
        self._running = True
        self._thread  = threading.Thread(target=self._loop, daemon=True)
        self._thread.start()

    def stop(self):
        self._running = False
        if self._thread:
            self._thread.join()

    def _loop(self):
        last_train_step       = 0
        steps_since_broadcast = 0

        while self._running:
            total = rl.get_stats().total_steps
            if total - last_train_step < self.train_every_n_steps:
                threading.Event().wait(0.005)  # 5 ms sleep, avoid spin
                continue

            batch = rl._bridge.sample_batch(self.batch_size)
            if batch is None:
                continue

            loss = self._compute_loss(batch)
            self.optimizer.zero_grad()
            loss.backward()
            self.optimizer.step()

            rl._bridge.notify_train_step(loss.item())
            last_train_step       = total
            steps_since_broadcast += 1

            if steps_since_broadcast >= self.broadcast_every_n_steps:
                rl._bridge.broadcast_weights(self.policy.state_dict())
                steps_since_broadcast = 0

    def _compute_loss(self, batch):
        """Default policy gradient loss. Override for PPO, SAC, TD3, etc."""
        obs, actions, rewards, next_obs, dones = batch
        mean, std, _ = self.policy(obs, task_id=rl.current_task().task_id)
        dist  = torch.distributions.Normal(mean, std)
        log_p = dist.log_prob(actions).sum(-1)
        return -(log_p * rewards).mean()
```

### 5.5 Default Policy Network

```python
# rl/policy.py
import torch
import torch.nn as nn

class RLDefaultPolicy(nn.Module):
    """
    Default actor-critic MLP for continuous action spaces.
    Supports multi-task conditioning via task one-hot vector.
    Compatible with PPO and SAC out of the box.
    Replace with any nn.Module via rl.set_policy().
    """

    def __init__(self, obs_dim: int, action_dim: int,
                 num_tasks: int = 1, hidden_dim: int = 256):
        super().__init__()
        self.num_tasks  = num_tasks
        self.action_dim = action_dim

        self.shared = nn.Sequential(
            nn.Linear(obs_dim + num_tasks, hidden_dim),
            nn.LayerNorm(hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.LayerNorm(hidden_dim), nn.ReLU(),
        )
        self.actor_mean    = nn.Linear(hidden_dim, action_dim)
        self.actor_log_std = nn.Parameter(torch.zeros(action_dim))
        self.critic        = nn.Linear(hidden_dim, 1)

    def forward(self, obs: torch.Tensor, task_id: int):
        onehot = torch.zeros(*obs.shape[:-1], self.num_tasks, device=obs.device)
        onehot[..., task_id] = 1.0
        x        = torch.cat([obs, onehot], dim=-1)
        features = self.shared(x)
        mean     = torch.tanh(self.actor_mean(features))
        std      = torch.exp(self.actor_log_std.clamp(-4, 2))
        return mean, std, self.critic(features)

    @torch.no_grad()
    def get_action(self, obs: torch.Tensor, task_id: int,
                   deterministic: bool = False) -> torch.Tensor:
        mean, std, _ = self.forward(obs, task_id)
        if deterministic:
            return mean
        return torch.distributions.Normal(mean, std).sample().clamp(-1, 1)
```

### 5.6 Weight Serialization

```python
# rl/weights.py
import torch, io

def state_dict_to_bytes(state_dict: dict) -> bytes:
    """Serialize policy state_dict to bytes for RPC transmission."""
    buf = io.BytesIO()
    torch.save(state_dict, buf)
    return buf.getvalue()

def bytes_to_state_dict(data: bytes) -> dict:
    """Deserialize state_dict bytes received via RPC."""
    return torch.load(io.BytesIO(data), map_location="cpu")
```

### 5.7 Curriculum Manager

```python
# rl/curriculum.py
from dataclasses import dataclass

@dataclass
class CurriculumStage:
    task_id:      int
    task_name:    str
    min_reward:   float   # avg reward required to advance
    min_episodes: int     # minimum episodes before advancing
    max_episodes: int     # force advance after this to avoid getting stuck

class CurriculumManager:
    """
    Progressive task difficulty manager. Runs on master only.
    Stage advances are broadcast to workers via rl.curriculum_update RPC.
    """

    def __init__(self, stages: list[CurriculumStage], ema_alpha: float = 0.05):
        self.stages      = stages
        self.ema_alpha   = ema_alpha
        self._stage_idx  = 0
        self._episodes   = 0
        self._avg_reward = -float('inf')

    @property
    def current_stage(self) -> CurriculumStage:
        return self.stages[self._stage_idx]

    @property
    def current_task_id(self) -> int:
        return self.current_stage.task_id

    def update(self, episode_reward: float) -> bool:
        """Update stats with latest episode reward. Returns True if stage advanced."""
        self._episodes   += 1
        self._avg_reward  = (self.ema_alpha * episode_reward +
                             (1 - self.ema_alpha) * self._avg_reward)

        can_advance = (
            self._stage_idx < len(self.stages) - 1 and
            self._episodes  >= self.current_stage.min_episodes and
            (self._avg_reward >= self.current_stage.min_reward or
             self._episodes   >= self.current_stage.max_episodes)
        )

        if can_advance:
            self._stage_idx  += 1
            self._episodes    = 0
            self._avg_reward  = -float('inf')
            return True
        return False
```

---

## 6. User-Facing Integration

This is the complete code a user writes to train any agent with parallel workers.
Role detection and worker spawning are automatic.

```python
# scene_init.py — runs once on EVERY instance (master and workers)
import rl
import torch
from my_tasks import CarDrivingTask

# Role auto-detected from --rl-worker-id CLI arg; absent means master
rl.initialize(obs_dim=14, action_dim=2)

# Curriculum registered on all instances (workers need task definitions too)
rl.set_curriculum([
    {"task": CarDrivingTask, "stage": 0, "min_reward": 50,  "min_episodes": 500},
    {"task": CarDrivingTask, "stage": 1, "min_reward": 150, "min_episodes": 2000},
])

# Master only: create policy, start trainer, spawn workers
if rl.is_master():
    policy    = rl.RLDefaultPolicy(obs_dim=14, action_dim=2, num_tasks=2)
    optimizer = torch.optim.Adam(policy.parameters(), lr=3e-4)
    rl.set_policy(policy)
    rl.start_training(policy, optimizer)
    n = rl.start_workers("my_scene.blend")  # auto: cpu_count - 1
    print(f"[RL] Master ready — {n} workers spawned")


# agent_controller.py — runs every frame on the agent object (all instances)
import rl

def main(cont):
    obj          = cont.owner
    task         = rl.current_task()
    obs          = task.get_observation(obj)
    action       = rl.get_action(obs)
    task.apply_action(obj, action)
    reward, done = task.compute_reward(obj)
    rl.store(obs, action, reward, done)
    if done:
        rl.reset(obj)
```

---

## 7. Headless Mode

Workers are spawned via `blenderplayer --headless`, reusing the flag already
implemented in the blenderplayer project. No render pipeline runs on workers.

```bash
# Fully headless training session
blenderplayer --headless my_scene.blend

# Master renders for monitoring; workers remain headless
blenderplayer my_scene.blend
```

Stdout output per instance during training:

```
[RL:master] step=12400 | task=waypoint | reward=87.3 | avg=74.1 | workers=7 | loss=0.042
[RL:w1]     ep=340     | task=waypoint | reward=91.2 | steps=34K
[RL:w2]     ep=338     | task=waypoint | reward=85.7 | steps=33K
```

---

## 8. Save / Export

```python
rl.save("checkpoint.rl")             # full checkpoint (resume training)
rl.load("checkpoint.rl")             # resume from checkpoint
rl.export_onnx("policy.onnx")        # ONNX Runtime — no PyTorch required at runtime
rl.export_torchscript("policy.pt")   # TorchScript — minimal Python required
```

Checkpoint format (`.rl`) — ZIP archive:

| File | Content |
|------|---------|
| `policy.pt` | TorchScript model (inference-only) |
| `optimizer.pt` | Optimizer state (for resuming training) |
| `curriculum.json` | Current stage, episode count, avg reward |
| `config.json` | obs_dim, action_dim, num_tasks, worker_count |

---

## 9. Performance Targets

| Metric | 1 instance | 8 workers | 16 workers | Notes |
|--------|-----------|-----------|------------|-------|
| `StoreTransition()` latency | < 100 ns | < 100 ns | < 100 ns | Lock-free ring buffer, no alloc |
| `GetAction()` latency | < 1 ms | < 1 ms | < 1 ms | Local policy copy inference |
| `Tick()` overhead | < 0.1 ms | < 0.1 ms | < 0.1 ms | Flush only every N steps |
| RPC batch send latency | — | < 2 ms | < 2 ms | NET loopback |
| Weight broadcast latency | — | < 10 ms | < 20 ms | Scales with model size |
| Sim steps/second | ~50K | ~400K | ~800K | Headless, no render |
| Gradient steps/second | ~500 | ~500 | ~500 | Single trainer thread |
| Car policy training time | ~4 hours | ~30 min | ~15 min | Estimated, SAC algorithm |

---

## 10. Dependencies

| Dependency | Condition | Notes |
|-----------|-----------|-------|
| NET_NetworkManager | Always | Already implemented in Network module |
| pybind11 | `WITH_PYTHON` only | Already in UPBGE |
| PyTorch >= 2.0 | `WITH_PYTHON` only | Policy network + training |
| ONNX Runtime | Optional | Export and deploy only |

---

## 11. Blender/DNA Integration

RL configuration is stored in scene DNA for Blender UI integration,
following the same pattern as `GameNetworkData` in the Network module.

```c
/* source/blender/makesdna/DNA_scene_types.h */
typedef struct GameRLData {
    short enabled;
    short worker_count;         /* 0 = auto (cpu_count - 1) */
    short obs_dim;
    short action_dim;
    int   buffer_capacity;
    int   flush_every_n_steps;
    int   master_port;
    float ema_alpha;
    short sync_on_policy;
    short _pad[3];
} GameRLData;

/* Field added to GameData struct: */
GameRLData rl;
```

Blender files modified (minimal — 3 files):

| File | Change |
|------|--------|
| `DNA_scene_types.h` | Add `GameRLData` struct + field in `GameData` |
| `rna_scene.cc` | Add `rna_def_scene_game_rl()` to expose fields in UI |
| `properties_game.py` | Add `RL_PT_training` panel — enable toggle, worker count, port |

All other files live in `gameengine/AI/` — no further Blender tree modifications.

---

## 12. Implementation Phases

| Phase | Scope | Key Files |
|-------|-------|-----------|
| **1** | C++ types, buffer, protocol structs | `RL_Types.h`, `RL_Buffer.h`, `RL_Protocol.h` |
| **2** | RL_Manager skeleton + NET RPC registration | `RL_Manager.h`, `RL_Manager.cpp` |
| **3** | RPC handlers (C++ side, always compiled) | `RL_Manager.cpp` |
| **4** | Process launcher | `RL_Launcher.h`, `RL_Launcher.cpp` |
| **5** | KX_KetsjiEngine integration | `KX_KetsjiEngine.h/.cpp` (minimal) |
| **6** | pybind11 bindings (`#ifdef WITH_PYTHON` block) | `RL_Manager.cpp` |
| **7** | Python task base + curriculum | `task.py`, `curriculum.py` |
| **8** | Python trainer + weight serialization | `trainer.py`, `weights.py` |
| **9** | Python public API + launcher wrapper | `__init__.py`, `launcher.py`, `policy.py` |
| **10** | Reference task implementations | `tasks/hover.py`, `tasks/waypoint.py`, ... |
| **11** | Save/export pipeline | checkpoint, ONNX, TorchScript |
| **12** | DNA + RNA + UI panel | `DNA_scene_types.h`, `rna_scene.cc`, `properties_game.py` |
