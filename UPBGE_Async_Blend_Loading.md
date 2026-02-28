# UPBGE Async Selective Blend Loading — Implementation Architecture v3.2

## Table of Contents

1. [Overview](#overview)
2. [File Structure](#file-structure)
3. [Execution Flow](#execution-flow)
4. [Core Components](#core-components)
5. [Post-Merge Integration Pipeline](#post-merge-integration-pipeline)
6. [BLO Thread Safety Audit](#blo-thread-safety-audit)
7. [Threading Model](#threading-model)
8. [Integration Points](#integration-points)
9. [Python API](#python-api)
10. [Class Diagram](#class-diagram)
11. [Call Sequence](#call-sequence)
12. [Pre-Implementation Checklist](#pre-implementation-checklist)
13. [Performance Metrics](#performance-metrics)
14. [Time-Sliced Integration (Phases 2 + 2.1)](#time-sliced-integration-phases-2--21)
15. [Progress Reporting (Phase 3)](#progress-reporting-phase-3)
16. [Future Roadmap](#future-roadmap)

---

## Overview

### Goal

Eliminate game loop freezing during external `.blend` file loading via:

- **Selective loading** by collection/object using `BKE_blendfile_link` + `BKE_blendfile_append`
- **Real threading** via `BLI_task_pool` without `work_and_wait`
- **6-step post-merge pipeline** with collection linking, depsgraph eval, and conversion
- **Modifier evaluation at load time** — full depsgraph eval (default) or raw mesh (opt-in)
- **Time-sliced integration** with configurable per-frame budget

### Design Principles

```
┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #1: Performance First                                 │
│ - Main thread NEVER waits for the loading thread                │
│ - IntegratePendingLoads() returns immediately if nothing READY  │
│ - std::atomic<BlendLoadState> → zero locks on hot path          │
│ - Time-sliced integration: max 1 load per frame by default      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #2: Isolation + Correct Integration                   │
│ - Worker thread operates ONLY on staging_main                   │
│ - NEVER touches G.main, depsgraph, or KX_Scene structures       │
│ - Main thread handles: merge → collection linking →             │
│   depsgraph eval → conversion (6-step pipeline)                 │
│ - ReportList is local to worker thread                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #3: Minimal Invasiveness                              │
│ - DO NOT touch Blender code (public APIs only)                  │
│ - Changes to existing files: ~15 lines total                    │
│ - All new code lives in gameengine/Ketsji/                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #4: Safe Lifetime Management                          │
│ - shared_ptr for PendingBlendLoad (Python handle + loader)      │
│ - Double-buffer for lock-free iteration                         │
│ - BLI_task_pool per load with explicit free on completion       │
└─────────────────────────────────────────────────────────────────┘
```

---

## File Structure

### New Files (gameengine/Ketsji/)

```
gameengine/Ketsji/
├── KX_BlendFileLoader.h              [~200 lines]
│   ├── enum class BlendLoadState     (IDLE, LOADING, READY, CANCELLING, ERROR, DONE)
│   ├── struct PendingBlendLoad       (shared_ptr-managed, atomic state, TaskPool*)
│   └── class KX_BlendFileLoader      (RequestLoad, IntegratePendingLoads, CancelAll)
│
├── KX_BlendFileLoader.cpp            [~380 lines]
│   ├── RequestLoad()                 (resolves path, creates BLI_task_pool, pushes task)
│   ├── LoadThreadFunc()              (worker: BLO + BKE_blendfile_link + append)
│   ├── IntegratePendingLoads()       (main: drain incoming, BKE_main_merge, convert)
│   └── CancelAll()                   (shutdown: signal CANCELLING, free pools)
│
├── KX_PyBlendLoadHandle.h            [~50 lines]
│   └── class KX_PyBlendLoadHandle    (holds shared_ptr<PendingBlendLoad>)
│
├── KX_PyBlendLoadHandle.cpp          [~180 lines]
│   ├── pyattr_get_state()            ('loading' | 'error' | 'done')
│   ├── pyattr_get_error()            (str | None)
│   ├── pyattr_get_filepath()         (str)
│   ├── pyattr_get_collections()      (list[str])
│   └── py_cancel()                   (CAS atomic LOADING → CANCELLING)
│
└── (integrated in KX_PythonInit.cpp) [~60 lines]
    └── BGE_loadBlendAsync()          (Python entry: parse args, call RequestLoad)
```

### Modified Files (PATCH)

```diff
gameengine/Ketsji/KX_KetsjiEngine.h
+ #include "KX_BlendFileLoader.h"
+ private:
+   std::unique_ptr<KX_BlendFileLoader> m_blendFileLoader;
+ public:
+   KX_BlendFileLoader *GetBlendFileLoader();

gameengine/Ketsji/KX_KetsjiEngine.cpp
+ #include "KX_BlendFileLoader.h"
+ // In constructor:
+ m_blendFileLoader = std::make_unique<KX_BlendFileLoader>();
+ // Getter:
+ KX_BlendFileLoader *KX_KetsjiEngine::GetBlendFileLoader() {
+   return m_blendFileLoader.get();
+ }

gameengine/Ketsji/KX_Scene.cpp
+ #include "KX_BlendFileLoader.h"
+ // At the END of LogicEndFrame():
+ if (KX_BlendFileLoader *loader = KX_GetActiveEngine()->GetBlendFileLoader()) {
+   loader->IntegratePendingLoads(this);
+ }

gameengine/Ketsji/CMakeLists.txt
+ set(SRC
+   ...
+   KX_BlendFileLoader.cpp
+   KX_PyBlendLoadHandle.cpp
+ )
```

**Total lines modified in existing files: ~15**

---

## Execution Flow

### Complete Load Lifecycle

```
PYTHON USER CODE              MAIN THREAD                    WORKER THREAD
═══════════════════          ═══════════════════           ═══════════════════

handle = loadBlendAsync()
    │
    ├──► RequestLoad()
    │       │
    │       ├─ resolve '//' path to absolute (MAIN THREAD)
    │       │
    │       ├─ new shared_ptr<PendingBlendLoad>
    │       │  state = LOADING
    │       │
    │       ├─ BKE_main_new() → staging_main (MAIN THREAD)
    │       │  STRNCPY(staging_main->filepath, G_MAIN->filepath)
    │       │
    │       ├─ BLI_task_pool_create()
    │       ├─ BLI_task_pool_push(LoadThreadFunc)
    │       │  NO work_and_wait ✓
    │       │
    │       ├─ push to m_incoming (mutex ~ns)
    │       │
    │       └─ return shared_ptr ──────┐
    │                                   │
    ▼                                   │              LoadThreadFunc()
Game loop continues                     │                   │
100% uninterrupted                      │                   ├─ BKE_reports_init (LOCAL)
                                        │                   ├─ BLO_blendhandle_from_file
                                        │                   ├─ staging_main already prepared
                                        │                   ├─ LibraryLink_Params {
                                        │                   │    .bmain = staging_main,
                                        │                   │    .context.scene = nullptr,
                                        │                   │    NO FILE_RELPATH flag
                                        │                   │  }
                                        │                   ├─ BKE_blendfile_link(lapp, staging)
                                        │                   ├─ BKE_blendfile_append(lapp, staging)
                                        │                   │  (operates ONLY on staging_main)
                                        │                   │
                                        │                   ├─ NO context_init_done()
                                        │                   ├─ NO instantiate_loose()
                                        │                   ├─ NO context_finalize()
                                        │                   │
                                        │                   └─ state.store(READY) ─────────┐
                                        │                                                  │
═══ NEXT FRAME ═════════════════════════════════════════════════════════════════════════════
                                        │                                                  │
NextFrame() ◄───────────────────────────┘◄─────────────────────────────────────────────────┘
    │
    ├─ m_converter->MergeAsyncLoads()     (existing, legacy)
    │
    ├─► IntegratePendingLoads(engine) ◄── POSITION ① (top of frame)
    │       │
    │       ├─ Drain m_incoming → m_pending (mutex ~ns)
    │       │
    │       ├─ for load in m_pending:
    │       │     if state == READY:
    │       │
    │       │   ┌──────────────────────────────────────────────┐
    │       │   │ 6-STEP POST-MERGE PIPELINE                   │
    │       │   ├──────────────────────────────────────────────┤
    │       │   │ ① Safety clear G_MAIN tags                  │
    │       │   │ ② Collect ID pointers from staging_main     │
    │       │   │ ③ BKE_main_merge(G_MAIN, &staging)          │
    │       │   │ ④ BKE_collection_child_add → master_coll    │
    │       │   │ ⑤ BKE_main_collection_sync_remap(G_MAIN)    │
    │       │   │ ⑥ DEG eval (if evaluate=true)               │
    │       │   │   DEG_relations_tag_update + tag IDs         │
    │       │   │   BKE_scene_graph_update_tagged              │
    │       │   │ ⑦ Convert → KX_GameObjects                  │
    │       │   │   BL_ConvertBlenderObjects (per collection)  │
    │       │   └──────────────────────────────────────────────┘
    │       │
    │       ├─── on_ready callback (Python, main thread)
    │       ├─── state = DONE
    │       └─── (max 1 load integrated per frame by default)
    │
    ├─ for each scene:
    │     LogicBeginFrame()         ← new objects participate in logic
    │     UpdateParents()
    │     LogicUpdateFrame()
    │     LogicEndFrame()
    │     Physics                   ← new objects participate in physics
    │     UpdateParents()
    │
    └─ ProcessScheduledScenes()

Render()                               (separate from NextFrame)
    ├─ UpdateAnimations()               ← depsgraph already evaluated
    ├─ GetFrameRenderData()
    └─ RenderCamera() → EEVEE          ← new objects rendered
```

---

## Core Components

### 1. `BlendLoadState` — State Machine

```cpp
enum class BlendLoadState : int {
    IDLE = 0,       // Not in use / initial
    LOADING,        // Worker thread active
    READY,          // Worker done, awaiting integration
    INTEGRATING,    // Steps 1-4 done, time-sliced eval+convert in progress
    CANCELLING,     // Cancel requested, worker will check
    ERROR,          // Worker encountered error
    DONE            // Fully integrated, handle still queryable
};
```

State transitions:
```
                 RequestLoad()
    IDLE ─────────────────────► LOADING
                                   │
                     ┌─────────────┼──────────────┐
                     │             │              │
                  (cancel)    (success)       (failure)
                     │             │              │
                     ▼             ▼              ▼
                CANCELLING      READY          ERROR
                     │             │
                (thread ends) (Steps 1-4)
                     │             │
                     ▼             ▼
                   ERROR      INTEGRATING
                                   │
                              (all batches done)
                                   │
                                   ▼
                                 DONE
```

### 2. `PendingBlendLoad` — Load State (shared_ptr managed)

```cpp
struct PendingBlendLoad {
    // === INPUT (written by main thread before task push) ===
    std::string filepath;                       // Original path (may contain '//')
    std::string filepath_resolved;              // Absolute path (resolved on main thread)
    std::vector<std::string> collection_names;  // ID_GR names to load
    std::vector<std::string> object_names;      // ID_OB names to load
    bool evaluate = true;                       // true = depsgraph eval (modifiers applied)
                                                // false = raw mesh (no modifiers, faster)

    // === OUTPUT (written by worker thread) ===
    Main *staging_main = nullptr;               // Isolated Main, created in RequestLoad
                                                // with filepath copied from G_MAIN

    // === SYNCHRONIZATION ===
    std::atomic<BlendLoadState> state{BlendLoadState::IDLE};

    // === THREAD OWNERSHIP ===
    TaskPool *task_pool = nullptr;              // BLI_task_pool, one per load

    // === ERROR ===
    std::string error_msg;

    // === CALLBACK (optional) ===
    std::function<void(PendingBlendLoad *)> on_ready;

    // === SCENE TARGET ===
    KX_Scene *target_scene = nullptr;           // Scene to integrate into
    uint64_t scene_generation = 0;              // Monotonic counter, ABA-immune
};
```

**Ownership:** Managed via `std::shared_ptr`. Both `KX_BlendFileLoader::m_pending` and
`KX_PyBlendLoadHandle` hold shared references. The struct is destroyed only when ALL
references are released.

**Thread Safety:** Only the owning phase's thread touches mutable fields (except `state`
which is atomic). The `shared_ptr` ref count uses atomic operations internally.

### 3. `KX_BlendFileLoader` — Central Manager

```cpp
class KX_BlendFileLoader {
public:
    // Launch background load, returns shared_ptr (also held by Python handle)
    std::shared_ptr<PendingBlendLoad> RequestLoad(
        const std::string &filepath,
        const std::vector<std::string> &collections,
        const std::vector<std::string> &objects,
        std::function<void(PendingBlendLoad *)> on_ready,
        KX_Scene *target_scene,
        bool evaluate = true);

    // Called from top of NextFrame() — integrates completed loads
    // Receives engine (not scene) to access context, converter, all scenes
    void IntegratePendingLoads(KX_KetsjiEngine *engine);

    // Shutdown cleanup — blocks until all workers finish
    void CancelAll();

    // Configuration
    void SetMaxIntegrationsPerFrame(int max) { m_maxIntegrationsPerFrame = max; }
    int GetMaxIntegrationsPerFrame() const { return m_maxIntegrationsPerFrame; }

private:
    static void LoadThreadFunc(TaskPool *pool, void *taskdata);

    // Active loads being tracked
    std::vector<std::shared_ptr<PendingBlendLoad>> m_pending;

    // Double-buffer: new loads land here, drained into m_pending each frame
    std::vector<std::shared_ptr<PendingBlendLoad>> m_incoming;
    std::mutex m_incoming_mutex;  // Protects ONLY m_incoming add/drain

    // Time-slicing: max loads to integrate per frame (default: 1)
    int m_maxIntegrationsPerFrame = 1;
};
```

**Lifetime:** One instance per `KX_KetsjiEngine` (created in constructor).
**Destructor:** Calls `CancelAll()` automatically via `unique_ptr`.

### 4. `LoadThreadFunc` — Worker Thread (CRITICAL)

```cpp
static void KX_BlendFileLoader::LoadThreadFunc(TaskPool * /*pool*/, void *taskdata) {
    PendingBlendLoad *load = static_cast<PendingBlendLoad *>(taskdata);

    // ─── 1. LOCAL ReportList (NOT shared with main thread) ───
    ReportList reports;
    BKE_reports_init(&reports, RPT_ERROR);
    BlendFileReadReport bf_reports{};
    bf_reports.reports = &reports;

    // ─── 2. Open .blend (index only) ───
    BlendHandle *bh = BLO_blendhandle_from_file(
        load->filepath_resolved.c_str(), &bf_reports);
    if (!bh) {
        load->error_msg = "Cannot open blend file: " + load->filepath_resolved;
        load->state.store(BlendLoadState::ERROR, std::memory_order_release);
        BKE_reports_free(&reports);
        return;
    }

    // ─── 3. Check cancellation ───
    if (load->state.load(std::memory_order_acquire) == BlendLoadState::CANCELLING) {
        BLO_blendhandle_close(bh);
        load->error_msg = "Cancelled";
        load->state.store(BlendLoadState::ERROR, std::memory_order_release);
        BKE_reports_free(&reports);
        return;
    }

    // ─── 4. staging_main is ALREADY created by RequestLoad() on main thread ───
    //   - RequestLoad() called BKE_main_new()
    //   - RequestLoad() copied G_MAIN->filepath into staging_main->filepath
    //   This ensures blo_find_main_for_library_and_idname() can resolve
    //   library paths without accessing G_MAIN from the worker thread.
    BLI_assert(load->staging_main != nullptr);

    // ─── 5. Configure params ───
    //   CRITICAL: bmain = staging_main (all operations target staging)
    //   CRITICAL: context.scene = nullptr (prevents loose_data_instantiate)
    //   CRITICAL: NO FILE_RELPATH (prevents the ONLY global access in library_link_end)
    LibraryLink_Params params{};
    params.bmain = load->staging_main;
    params.flag = BLO_LIBLINK_APPEND_RECURSIVE;
    // params.flag does NOT include FILE_RELPATH → avoids BKE_main_blendfile_path_from_global()
    // params.context.scene = nullptr    ← default, prevents loose_data_instantiate
    // params.context.view_layer = nullptr
    // params.context.v3d = nullptr

    // ─── 6. Create link/append context ───
    BlendfileLinkAppendContext *lapp = BKE_blendfile_link_append_context_new(&params);

    int lib_idx = BKE_blendfile_link_append_context_library_add(
        lapp, load->filepath_resolved.c_str(), bh);

    for (const auto &col_name : load->collection_names) {
        BlendfileLinkAppendContextItem *item =
            BKE_blendfile_link_append_context_item_add(
                lapp, col_name.c_str(), ID_GR, nullptr);
        BKE_blendfile_link_append_context_item_library_index_enable(
            lapp, item, lib_idx);
    }

    for (const auto &obj_name : load->object_names) {
        BlendfileLinkAppendContextItem *item =
            BKE_blendfile_link_append_context_item_add(
                lapp, obj_name.c_str(), ID_OB, nullptr);
        BKE_blendfile_link_append_context_item_library_index_enable(
            lapp, item, lib_idx);
    }

    // ─── 7. Execute Link + Append ───
    //   DO NOT call context_init_done() → triggers BKE_callback_exec (Python handlers)
    //   DO NOT call instantiate_loose()  → accesses scene/viewlayer
    //   DO NOT call context_finalize()   → triggers BKE_callback_exec

    BKE_blendfile_link(lapp, &reports);

    if (!(params.flag & FILE_LINK)) {
        BKE_blendfile_append(lapp, &reports);
    }

    // ─── 8. Cleanup ───
    BKE_blendfile_link_append_context_free(lapp);
    BLO_blendhandle_close(bh);

    // Check for errors in reports
    if (BKE_reports_contain(&reports, RPT_ERROR)) {
        load->error_msg = "Errors during blend file loading";
        // staging_main may still have partial data — free it
        if (load->staging_main) {
            BKE_main_free(load->staging_main);
            load->staging_main = nullptr;
        }
        load->state.store(BlendLoadState::ERROR, std::memory_order_release);
        BKE_reports_free(&reports);
        return;
    }

    BKE_reports_free(&reports);

    // ─── 9. Final cancellation check ───
    if (load->state.load(std::memory_order_acquire) == BlendLoadState::CANCELLING) {
        if (load->staging_main) {
            BKE_main_free(load->staging_main);
            load->staging_main = nullptr;
        }
        load->error_msg = "Cancelled";
        load->state.store(BlendLoadState::ERROR, std::memory_order_release);
        return;
    }

    // ─── 10. Signal completion ───
    load->state.store(BlendLoadState::READY, std::memory_order_release);
}
```

**Thread safety — verified by static source audit:**

See [BLO Thread Safety Audit](#blo-thread-safety-audit) section for the complete
function-by-function analysis. All functions in the critical path have been verified
against `readfile.cc`, `readblenentry.cc`, `blendfile_link_append.cc`, and
`blendfile.cc` source code. Zero direct global access found. Indirect access through
subsystem callbacks (IDTypeInfo, do_versions, colormanagement) cannot be ruled out
without runtime validation. **TSAN recommended post-implementation as validation step.**

### 5. `IntegratePendingLoads` — Main Thread (7-Step Pipeline)

```cpp
void KX_BlendFileLoader::IntegratePendingLoads(KX_KetsjiEngine *engine) {
    // ─── 1. Drain incoming → pending (lock duration: ~nanoseconds) ───
    {
        std::lock_guard<std::mutex> lock(m_incoming_mutex);
        if (!m_incoming.empty()) {
            m_pending.insert(m_pending.end(),
                std::make_move_iterator(m_incoming.begin()),
                std::make_move_iterator(m_incoming.end()));
            m_incoming.clear();
        }
    }

    if (m_pending.empty()) return;  // Fast path: O(1)

    // ─── 2. Process completed loads (time-sliced) ───
    int integrated = 0;

    for (auto &load : m_pending) {
        if (integrated >= m_maxIntegrationsPerFrame) break;

        // Check if READY (acquire ensures staging_main visibility)
        if (load->state.load(std::memory_order_acquire) != BlendLoadState::READY) {
            continue;
        }

        // ─── Validate target scene still exists (ABA protection) ───
        KX_Scene *scene = load->target_scene;
        if (!scene || load->scene_generation != scene->GetGeneration()) {
            load->error_msg = "Target scene destroyed during load";
            load->state.store(BlendLoadState::ERROR, std::memory_order_release);
            // Free staging_main to prevent leak
            if (load->staging_main) {
                BKE_main_free(load->staging_main);
                load->staging_main = nullptr;
            }
            continue;
        }

        // ─── Free task pool (join is instantaneous, thread already done) ───
        if (load->task_pool) {
            BLI_task_pool_free(load->task_pool);
            load->task_pool = nullptr;
        }

        // ══════════════════════════════════════════════════════════
        //  6-STEP POST-MERGE PIPELINE
        // ══════════════════════════════════════════════════════════

        try {

        // ─── STEP 0: Safety Clear G_MAIN tags ───
        // CRITICAL: Prevent "tag pollution" from previous operations or Python scripts.
        // If we don't clear, we might accidentally pick up existing objects
        // that have residual ID_TAG_DOIT flags.
        BKE_main_id_tag_all(G_MAIN, ID_TAG_DOIT, false);

        // ─── STEP 1: Tag IDs in staging_main BEFORE merge ───
        // BKE_main_merge MOVES surviving IDs to G_MAIN and FREES staging_main.
        // On collision (same name from same library), the EXISTING ID in G_MAIN
        // wins — the colliding source ID stays in staging → gets freed.
        //
        // Strategy: tag all staging IDs with DOIT before merge, then post-merge
        // scan G_MAIN for tagged IDs → those are the ones that survived.
        // This is the same pattern used by BL_Converter::FreeBlendFile.
        BKE_main_id_tag_all(load->staging_main, ID_TAG_DOIT, true);

        // ─── STEP 2: Merge staging → G_MAIN ───
        // ~0.05-0.2ms — pointer operations only, moves ListBase nodes.
        // After this call: load->staging_main == nullptr (always freed).
        // Surviving IDs are now in G_MAIN with ID_TAG_DOIT set.
        // Collided IDs were freed with staging_main.
        MainMergeReport merge_report;
        BKE_main_merge(G_MAIN, &load->staging_main, merge_report);

        // ─── STEP 2b: Collect SURVIVING IDs from G_MAIN by tag ───
        // Only IDs with ID_TAG_DOIT were loaded by us and survived the merge.
        // This is O(N) over G_MAIN ListBases, but these are linked lists
        // and we only scan collections + objects — negligible cost.
        std::vector<Collection *> loaded_collections;
        std::vector<Object *> loaded_objects;

        LISTBASE_FOREACH(Collection *, col, &G_MAIN->collections) {
            if (col->id.tag & ID_TAG_DOIT) {
                loaded_collections.push_back(col);
                col->id.tag &= ~ID_TAG_DOIT;  // Clear tag to not interfere
            }
        }
        LISTBASE_FOREACH(Object *, ob, &G_MAIN->objects) {
            if (ob->id.tag & ID_TAG_DOIT) {
                loaded_objects.push_back(ob);
                ob->id.tag &= ~ID_TAG_DOIT;
            }
        }

        // Log collisions if any (useful for debugging duplicate loads)
        if (merge_report.num_remapped_ids > 0) {
            CM_Warning("Async blend load: " << merge_report.num_remapped_ids
                       << " IDs collided during merge (existing IDs kept)");
        }

        // ─── STEP 3: Link collections into target scene ───
        // Without this, SETLOOPER won't find objects and depsgraph
        // won't evaluate them (they'd be floating orphans in G_MAIN).
        Scene *bl_scene = scene->GetBlenderScene();

        for (Collection *co : loaded_collections) {
            BKE_collection_child_add(G_MAIN, bl_scene->master_collection, co);
        }

        // Individual objects not inside any loaded collection → add directly
        for (Object *ob : loaded_objects) {
            bool in_loaded_collection = false;
            for (Collection *co : loaded_collections) {
                if (BKE_collection_has_object_recursive(co, ob)) {
                    in_loaded_collection = true;
                    break;
                }
            }
            if (!in_loaded_collection) {
                BKE_collection_object_add(G_MAIN, bl_scene->master_collection, ob);
            }
        }

        // ─── STEP 4: Sync collection hierarchy ───
        // Rebuilds internal ListBase caches after structural changes.
        // Required after BKE_collection_child_add.
        BKE_main_collection_sync_remap(G_MAIN);

        // ─── STEP 5: Depsgraph evaluation (conditional) ───
        // Default (evaluate=true): full depsgraph eval → modifiers applied.
        // Optional (evaluate=false): raw mesh mode → zero eval cost.
        if (load->evaluate) {
            // Tell depsgraph that the graph topology changed (new ID nodes)
            DEG_relations_tag_update(G_MAIN);

            // Tag new IDs for full recalculation (geometry, transforms, modifiers)
            for (Collection *co : loaded_collections) {
                DEG_id_tag_update(&co->id, ID_RECALC_ALL);
            }
            for (Object *ob : loaded_objects) {
                DEG_id_tag_update(&ob->id, ID_RECALC_ALL);
            }

            // Force evaluation NOW — evaluates ONLY tagged IDs, not entire scene.
            // Cost is proportional to loaded content complexity:
            //   10 simple objects: ~0.1ms
            //   10 objects w/ subdiv 2: ~1-3ms
            //   10 objects w/ subdiv 4 + booleans: ~5-20ms
            //   GPU modifiers (next release): significantly faster
            bContext *C = engine->GetContext();
            Depsgraph *depsgraph = CTX_data_depsgraph_on_load(C);
            BKE_scene_graph_update_tagged(depsgraph, G_MAIN);
        }

        // ─── STEP 6: Convert to KX_GameObjects ───
        // BL_ConvertBlenderObjects uses SETLOOPER → finds objects through
        // scene->master_collection hierarchy (linked in Step 3).
        // DEG_get_evaluated returns properly evaluated data (Step 5).
        for (Collection *co : loaded_collections) {
            scene->ConvertBlenderCollection(co, true);  // libloading=true
        }

        // Convert individual objects not in loaded collections
        for (Object *ob : loaded_objects) {
            bool in_loaded_collection = false;
            for (Collection *co : loaded_collections) {
                if (BKE_collection_has_object_recursive(co, ob)) {
                    in_loaded_collection = true;
                    break;
                }
            }
            if (!in_loaded_collection) {
                scene->ConvertBlenderObject(ob);
            }
        }

        } catch (const std::bad_alloc &) {
            // Safety net: Blender compiled with -fno-exceptions, never throws.
            // But our own std::vector allocations could in extreme OOM.
            // staging_main already consumed by merge, nothing to free.
            load->error_msg = "Out of memory during integration";
            load->state.store(BlendLoadState::ERROR, std::memory_order_release);
            continue;
        }

        // ─── Execute Python callback (on main thread, GIL held) ───
        if (load->on_ready) {
            PyObject *result = PyObject_CallFunction(
                load->on_ready_pyobj, "(O)", load->GetProxy());
            if (!result) {
                PyErr_Print();
                PyErr_Clear();
            } else {
                Py_DECREF(result);
            }
            load->on_ready = nullptr;
        }

        // ─── Mark as DONE (Python handle can still query state) ───
        load->state.store(BlendLoadState::DONE, std::memory_order_release);
        integrated++;
    }

    // ─── 3. Cleanup finished loads from tracking vector ───
    //   shared_ptr ref count keeps PendingBlendLoad alive if Python handle exists
    m_pending.erase(
        std::remove_if(m_pending.begin(), m_pending.end(),
            [](const std::shared_ptr<PendingBlendLoad> &p) {
                auto s = p->state.load(std::memory_order_relaxed);
                return s == BlendLoadState::DONE || s == BlendLoadState::ERROR;
            }),
        m_pending.end());
}
```

**Hot path cost (per integration):**

| Step | Cost | Notes |
|---|---|---|
| No pending loads | O(1) | `if (m_pending.empty()) return` |
| N active loads, none READY | O(N) | Trivial atomic reads |
| Step 0: Clear G_MAIN tags | ~0.005ms | Negligible, iterates ListBase |
| Step 1: Tag staging IDs | ~0.001ms | Traverse staging_main ListBases |
| Step 2: BKE_main_merge | ~0.05-0.2ms | Pointer ops only |
| Step 3: Collection linking | ~0.01-0.05ms | BKE_collection_child_add |
| Step 4: Collection sync | ~0.1-0.5ms | Depends on scene hierarchy depth |
| Step 5: DEG eval (evaluate=true) | ~0.1-20ms | Proportional to modifier complexity |
| Step 5: DEG eval (evaluate=false) | 0ms | Skipped entirely |
| Step 6: Convert | ~0.5-5ms | ConvertBlenderCollection/Object |
| **Total (evaluate=true, simple)** | **~1-3ms** | 10 objects, no heavy modifiers |
| **Total (evaluate=false)** | **~0.5-2ms** | Raw mesh mode |
| Max per frame | Configurable | Default: 1 load/frame |

---

## Post-Merge Integration Pipeline

### Why 6 Steps?

The old async system (`async_convert`) was broken because it ran `BL_ConvertBlenderObjects`
on a worker thread, which accesses the depsgraph (`DEG_get_evaluated`), scene context
(`CTX_data_depsgraph_on_load`), and global state. Our architecture separates I/O (worker)
from integration (main thread) correctly, but the main thread integration itself has
ordering constraints discovered from the KX_KetsjiEngine frame loop analysis.

**Key discovery:** After `BKE_main_merge`, new IDs exist in G_MAIN but:
1. They are **not linked** to any Scene's `master_collection` → `SETLOOPER` won't find them
2. The **depsgraph doesn't know** they exist → `DEG_get_evaluated` returns unevaluated data
3. The collection hierarchy **caches are stale** → Blender internal lookups may fail

### Pipeline Diagram

```
BKE_main_id_tag_all(G_MAIN, DOIT, false)    ← Clear tags (prevent pollution)
       │
       ▼
BKE_main_id_tag_all(staging, DOIT, true)    ← Mark incoming IDs
       │
       ▼
BKE_main_merge()
       │
       │  Surviving IDs moved to G_MAIN (tag preserved)
       │  Colliding IDs stayed in staging → freed
       │  staging_main → nullptr (always freed)
       │
       ▼
Scan G_MAIN for ID_TAG_DOIT       ← Collect surviving IDs
Clear tags                          ← Clean up
       │
       │  IDs in G_MAIN but "floating" — not in any scene
       │
       ▼
BKE_collection_child_add()          ← Link to scene's master_collection
       │
       │  IDs now in scene hierarchy, but caches stale
       │
       ▼
BKE_main_collection_sync_remap()    ← Rebuild internal caches
       │
       │  Hierarchy correct, but depsgraph outdated
       │
       ▼
DEG_relations_tag_update()          ← Mark depsgraph topology as dirty
DEG_id_tag_update() per new ID     ← Mark new IDs for full recalc
       │
       ▼
BKE_scene_graph_update_tagged()     ← Evaluate ONLY tagged IDs
       │                               Modifiers applied, transforms computed
       │                               Cost: proportional to content complexity
       │
       ▼
BL_ConvertBlenderObjects()          ← Convert evaluated data to KX_GameObjects
       │                               SETLOOPER finds objects (Step 3)
       │                               DEG_get_evaluated returns correct data (Step 5)
       │
       ▼
Objects in scene, ready for logic + physics + render
```

### evaluate=true vs evaluate=false

| Aspect | evaluate=true (default) | evaluate=false (raw mode) |
|---|---|---|
| Modifiers | ✅ Applied (subdiv, mirror, booleans, geometry nodes) | ❌ Raw mesh only |
| Transforms | ✅ Fully evaluated | ✅ From .blend directly |
| Cost Step 5 | ~0.1-20ms (depends on complexity) | 0ms (skipped) |
| Use case | General purpose, runtime modifiers | Pre-baked assets, maximum performance |
| GPU modifiers | ✅ Benefits from GPU acceleration | N/A |

**When evaluate=false, Steps 5 is skipped entirely.** Steps 3-4 (collection linking + sync)
are still required so the objects are findable and renderable. EEVEE will evaluate the
depsgraph for rendering anyway during the Render() pass — but `BL_ConvertMesh` in Step 6
will use the raw mesh data (no modifiers applied at conversion time).

### Why This Position in the Frame Loop?

```
NextFrame() {
    ① m_converter->MergeAsyncLoads()     ← existing legacy (broken)
    ②  m_blendFileLoader->IntegratePendingLoads(engine)  ← OUR POSITION

    for each scene:                      ← ALL scenes see new objects
        LogicBeginFrame()
        UpdateParents()
        LogicUpdateFrame()
        LogicEndFrame()
        Physics
        UpdateParents()

    ProcessScheduledScenes()
}

Render() {                              ← AFTER NextFrame()
    UpdateAnimations()                   ← depsgraph already evaluated (Step 5)
    RenderCamera() → EEVEE              ← new objects rendered correctly
}
```

**Position ① (top of frame) was chosen over LogicEndFrame() because:**

| Criterion | LogicEndFrame() (inside per-scene loop) | Top of frame (before per-scene loop) |
|---|---|---|
| Physics completed? | ❌ Physics runs AFTER LogicEndFrame | ✅ Previous frame's physics done |
| All scenes see new objects? | ❌ Only current scene | ✅ All scenes in this frame tick |
| Depsgraph safe? | ✅ Runs in Render() | ✅ Same reasoning |
| Consistent with existing code? | ❌ New insertion point | ✅ Next to MergeAsyncLoads() |
| Objects get logic in first frame? | ⚠️ Partial — only actuators | ✅ Full logic cycle |

### API Functions Used (all confirmed in Blender 5.x headers)

| Function | Header | Purpose |
|---|---|---|
| `BKE_main_merge` | `BKE_main.hh` | Merge staging → G_MAIN |
| `BKE_collection_child_add` | `BKE_collection.hh:349` | Link collection to parent |
| `BKE_collection_has_object_recursive` | `BKE_collection.hh:195` | Check object membership |
| `BKE_collection_object_add` | `BKE_collection.hh:216` | Add loose object to collection |
| `BKE_main_collection_sync_remap` | `BKE_collection.hh` | Rebuild collection caches |
| `DEG_relations_tag_update` | `DEG_depsgraph_build.hh` | Mark depsgraph topology dirty |
| `DEG_id_tag_update` | `DEG_depsgraph.hh` | Tag specific IDs for recalc |
| `BKE_scene_graph_update_tagged` | `BKE_scene.hh:216` | Evaluate tagged IDs only |

---

## BLO Thread Safety Audit

### Methodology

Full source code review of all functions in the worker thread critical path, searching for
any access to `G_MAIN`, `G.main`, `BKE_main_blendfile_path_from_global()`, or other global
state. Files reviewed:

- `source/blender/blenloader/intern/readfile.cc` (6216 lines)
- `source/blender/blenloader/intern/readblenentry.cc` (465 lines)
- `source/blender/blenkernel/intern/blendfile_link_append.cc` (2318 lines)
- `source/blender/blenkernel/intern/blendfile.cc` (2335 lines)
- `source/blender/blenloader/intern/blend_validate.cc`

### Result: Thread-Safe by Static Audit ✅

Only **ONE** global access was found across the entire call chain, and it is **conditional
on a flag we do not set**. Indirect access through subsystem callbacks (IDTypeInfo,
do_versions, colormanagement) cannot be ruled out without runtime validation.
**TSAN recommended post-implementation as validation step, not a blocker.**

### Global Access Inventory (exhaustive)

```
grep -rn "G_MAIN\|G\.main\|BKE_main_blendfile_path_from_global" across all BLO files:

readfile.cc:5361         ← ONLY HIT in entire chain
blend_validate.cc:210    ← comment only, not code
```

The single hit at `readfile.cc:5361` is inside `library_link_end()`:

```cpp
// readfile.cc:5355-5362, inside library_link_end():
if (flag & FILE_RELPATH) {                                        // ← CONDITIONAL
    STRNCPY(curlib->filepath, curlib->runtime->filepath_abs);
    BLI_path_rel(curlib->filepath, BKE_main_blendfile_path_from_global());  // ← GLOBAL
}
```

**This code only executes if `FILE_RELPATH` is set in `params->flag`.** Since we configure
`params.flag = BLO_LIBLINK_APPEND_RECURSIVE` (without `FILE_RELPATH`), this branch is
**never taken**. Relative paths are unnecessary for the game engine — we work with absolute
paths resolved on the main thread in `RequestLoad()`.

### Function-by-Function Verification

#### Layer 1: BLO (Blend Loader)

| Function | Source | Global Access | Notes |
|---|---|---|---|
| `BLO_blendhandle_from_file` | readblenentry.cc:65 | ❌ None | Calls `blo_filedata_from_file()` — pure file I/O |
| `BLO_library_link_begin` | readfile.cc:5303 | ❌ None | Calls `library_link_begin(params->bmain, fd, filepath)` |
| ↳ `library_link_begin` | readfile.cc:5228 | ❌ None | `fd->bmain = mainvar` (staging), iterates `mainvar` only |
| ↳ `blo_find_main_for_library_and_idname` | readfile.cc | ❌ None | Uses `BKE_main_blendfile_path(mainvar)` — reads `staging_main->filepath` (set by RequestLoad) |
| `BLO_library_link_named_part` | readfile.cc:5207 | ❌ None | Calls `link_named_part(mainl, fd, ...)` — all local |
| ↳ `link_named_part` | readfile.cc:5156 | ❌ None | `mainl` + `fd` only, reads bheads from file |
| ↳ `read_libblock` | readfile.cc | ❌ None | Allocates into `mainl` (staging split) |
| `BLO_library_link_end` | readfile.cc:5463 | ❌ None** | Calls `library_link_end(mainl, &fd, params->flag, reports)` |
| ↳ `library_link_end` | readfile.cc:5335 | ⚠️ **Only if FILE_RELPATH** | See line 5361 above — **we never set this flag** |
| ↳ `expand_main(fd, mainl, ...)` | readfile.cc:5108 | ❌ None | Iterates `mainvar` (staging), BKE_library_foreach uses local bmain=nullptr |
| ↳ `read_libraries(fd)` | readfile.cc:5742 | ❌ None | `basefd->bmain` = staging, opens library files with local fd |
| ↳ `read_library_file_data` | readfile.cc:5663 | ❌ None | `fd->bmain = bmain` (staging), file I/O local |
| ↳ `lib_link_all(fd, mainvar)` | readfile.cc:3871 | ❌ None | Iterates `bmain` passed as argument |
| ↳ `after_liblink_merged_bmain_process(mainvar)` | readfile.cc:3943 | ❌ None | Validates only `mainvar` (staging) |
| ↳ `fix_relpaths_library(BKE_main_blendfile_path(mainvar), mainvar)` | readfile.cc:5447 | ❌ None | Uses `mainvar->filepath` (staging, set by RequestLoad) |

** With the condition that `FILE_RELPATH` is NOT set in flags.

#### Layer 2: BKE (Blend Kernel)

| Function | Source | Global Access | Notes |
|---|---|---|---|
| `BKE_blendfile_link_append_context_new` | blendfile_link_append.cc:136 | ❌ None | Stores `params` pointer, allocates context |
| `BKE_blendfile_link` | blendfile_link_append.cc:1656 | ❌ None | All bmain via `params->bmain` |
| `BKE_blendfile_append` | blendfile_link_append.cc:1406 | ❌ None | All bmain via `lapp_context->params->bmain` |
| ↳ `blendfile_append_define_actions` | blendfile_link_append.cc:1186 | ❌ None | `bmain = lapp_context.params->bmain` |
| ↳ `BKE_lib_id_make_local(bmain, ...)` | — | ❌ None | Receives bmain as argument |
| ↳ `BKE_libblock_relink_to_newid(bmain, ...)` | — | ❌ None | Receives bmain as argument |
| ↳ `BKE_main_id_tag_all(bmain, ...)` | — | ❌ None | Receives bmain as argument |
| `BKE_blendfile_link_append_context_free` | blendfile_link_append.cc:144 | ❌ None | Frees context-local data |

#### Layer 3: Infrastructure

| Function | Thread-Safe | Notes |
|---|---|---|
| `MEM_guardedalloc` (MEM_mallocN, MEM_freeN, etc.) | ✅ Yes | Thread-safe allocator with internal mutex |
| `CLG_log` (CLOG_DEBUG, CLOG_ERROR, etc.) | ✅ Yes | Thread-safe logging system |
| `BKE_reports_init / BKE_reports_free` | ✅ Yes | Per-instance, no shared state |

### Critical Safety Requirements (summary)

Three conditions MUST be met for full thread safety:

1. **`params->flag` must NOT include `FILE_RELPATH`** — prevents the only global access
   (readfile.cc:5361)
2. **`params->context.scene` must be `nullptr`** — prevents `loose_data_instantiate`
   which accesses Scene/ViewLayer (blendfile_link_append.cc:1621)
3. **DO NOT call `context_init_done()` or `context_finalize()`** — they trigger
   `BKE_callback_exec` which executes Python handlers (blendfile_link_append.cc:365,380)

All three conditions are enforced in the `LoadThreadFunc` implementation above.

### `staging_main->filepath` Requirement

`library_link_begin` calls `blo_find_main_for_library_and_idname()` which uses
`BKE_main_blendfile_path(mainvar)` to resolve library paths. This reads
`mainvar->filepath` — our staging_main.

If `staging_main->filepath` is empty, library paths referenced inside the loaded .blend
file will not resolve correctly (e.g., a .blend that links data from another .blend).

**Solution:** `RequestLoad()` copies `G_MAIN->filepath` into `staging_main->filepath`
on the main thread, before launching the worker:

```cpp
load->staging_main = BKE_main_new();
STRNCPY(load->staging_main->filepath, BKE_main_blendfile_path(G_MAIN));
```

This is safe because `RequestLoad()` runs on the main thread where `G_MAIN` access is
valid.

---

## Threading Model

### Synchronization Model

```
┌────────────────────────────────────────────────────────────────────┐
│ std::atomic<BlendLoadState> — Lock-Free Synchronization            │
├────────────────────────────────────────────────────────────────────┤
│ WORKER writes:    LOADING → READY | ERROR                          │
│ MAIN THREAD reads: polling in IntegratePendingLoads()              │
│ CANCEL (from Python): CAS LOADING → CANCELLING                    │
│                                                                     │
│ Memory ordering:                                                    │
│   Worker:  store(READY, memory_order_release)                      │
│            └─ guarantees staging_main writes visible to main       │
│   Main:    load(state, memory_order_acquire)                       │
│            └─ guarantees staging_main reads see worker's writes    │
│   Cancel:  compare_exchange_strong(LOADING, CANCELLING, acq_rel)  │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ std::mutex m_incoming_mutex — Double-Buffer (Structural Only)      │
├────────────────────────────────────────────────────────────────────┤
│ Protects ONLY:                                                      │
│   RequestLoad():          push_back to m_incoming                  │
│   IntegratePendingLoads(): drain m_incoming → m_pending            │
│                                                                     │
│ Lock duration: ~nanoseconds (move iterators, clear vector)         │
│ m_pending iteration is NEVER under lock                            │
│                                                                     │
│ This eliminates the deadlock risk from v1 where a single mutex     │
│ protected both add and iterate (blocking RequestLoad during        │
│ BKE_main_merge + ConvertBlenderCollection + Python callback).      │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ BLI_task_pool — Thread Lifecycle (one per load)                    │
├────────────────────────────────────────────────────────────────────┤
│ Created:    in RequestLoad() via BLI_task_pool_create()            │
│ Task push:  BLI_task_pool_push() — NO work_and_wait               │
│ Freed:      in IntegratePendingLoads() when state == READY/ERROR  │
│             BLI_task_pool_free() does implicit join                │
│             (instantaneous because thread already finished)        │
│ Shutdown:   CancelAll() frees all pools (may block briefly)       │
│                                                                     │
│ Consistent with UPBGE threading model — no mixing with            │
│ std::thread or TBB.                                                │
└────────────────────────────────────────────────────────────────────┘
```

### Ownership by Phase

| Phase | Owner | Can Touch | shared_ptr Holders |
|---|---|---|---|
| `LOADING` | Worker thread | staging_main, error_msg, state | Loader + Python handle |
| `READY` | Main thread | staging_main (consume), state | Loader + Python handle |
| `ERROR` | Main thread | error_msg (read), state | Loader + Python handle |
| `DONE` | Nobody (idle) | Python handle reads state/error | Python handle only |
| Destroyed | — | shared_ptr ref count → 0 | None |

### Lifetime Guarantee (v1 → v2 fix)

**Problem in v1:** `unique_ptr<PendingBlendLoad>` in `m_pending` was erased after
integration. If a Python handle retained a raw pointer, accessing it caused
**use-after-free**.

**Solution in v2:** `shared_ptr<PendingBlendLoad>` is held by both the loader vector and
the Python handle object. When `m_pending` erases its reference after DONE/ERROR, the
struct survives as long as the Python handle exists. Python can safely query `state`,
`error`, `filepath` at any time.

**Cost:** One atomic increment (in `RequestLoad` when creating the Python handle) and one
atomic decrement (when the Python handle is garbage collected or when `m_pending` erases).
Both are **off hot-path** — they happen once per load, never per frame.

---

## Integration Points

### In `KX_KetsjiEngine::NextFrame()` — Position ①

```cpp
bool KX_KetsjiEngine::NextFrame() {
    m_logger.StartLog(tc_services);
    const FrameTimes times = GetFrameTimes();
    if (times.frames == 0) { /* ... */ return false; }

    for (unsigned short i = 0; i < times.frames; ++i) {
        m_frameTime += times.framestep;

        m_converter->MergeAsyncLoads();  // existing legacy (broken, to be removed)

        // [ASYNC_BLEND_LOAD] — integrate completed async loads
        if (m_blendFileLoader) {
            m_blendFileLoader->IntegratePendingLoads(this);
        }

        m_inputDevice->ReleaseMoveEvent();
        // ... rest of frame loop (logic, physics, scenegraph) ...
    }
}
```

**Why Position ① (top of frame, inside tick loop):**

- ✅ Previous frame's physics + logic + scenegraph ALL completed
- ✅ New objects visible to ALL scenes in this tick (logic, physics, render)
- ✅ Depsgraph evaluated before Render() uses it
- ✅ Consistent with existing `MergeAsyncLoads()` position
- ✅ Clean slate — no subsystem in mid-update

### In `KX_KetsjiEngine`

```cpp
class KX_KetsjiEngine {
    std::unique_ptr<KX_BlendFileLoader> m_blendFileLoader;
public:
    KX_BlendFileLoader *GetBlendFileLoader();
};

// Constructor:
KX_KetsjiEngine::KX_KetsjiEngine(...) {
    m_blendFileLoader = std::make_unique<KX_BlendFileLoader>();
}

// unique_ptr destructor → ~KX_BlendFileLoader() → CancelAll()
```

**Lifetime:** One instance per engine, lives for entire game.
**Destruction:** `unique_ptr` calls destructor → `CancelAll()` ensures clean shutdown.

### In `RequestLoad()` — Path Resolution + staging_main

```cpp
std::shared_ptr<PendingBlendLoad> KX_BlendFileLoader::RequestLoad(
    const std::string &filepath,
    const std::vector<std::string> &collections,
    const std::vector<std::string> &objects,
    std::function<void(PendingBlendLoad *)> on_ready,
    KX_Scene *target_scene,
    bool evaluate)
{
    auto load = std::make_shared<PendingBlendLoad>();
    load->filepath = filepath;
    load->collection_names = collections;
    load->object_names = objects;
    load->on_ready = std::move(on_ready);
    load->target_scene = target_scene;
    load->scene_generation = target_scene->GetGeneration();  // ABA protection
    load->evaluate = evaluate;

    // CRITICAL: Resolve '//' paths on main thread
    // BLI_path_abs needs BKE_main_blendfile_path(G_MAIN) which is NOT safe from worker
    char resolved[FILE_MAX];
    STRNCPY(resolved, filepath.c_str());
    BLI_path_abs(resolved, BKE_main_blendfile_path(G_MAIN));
    load->filepath_resolved = resolved;

    // Verify file exists before launching worker thread
    if (!BLI_exists(load->filepath_resolved.c_str())) {
        load->error_msg = "File not found: " + load->filepath_resolved;
        load->state.store(BlendLoadState::ERROR, std::memory_order_release);
        return load;
    }

    // CRITICAL: Create staging_main on main thread and copy G_MAIN filepath
    // Worker thread needs staging_main->filepath for blo_find_main_for_library_and_idname()
    // to resolve library paths inside the loaded .blend file
    load->staging_main = BKE_main_new();
    STRNCPY(load->staging_main->filepath, BKE_main_blendfile_path(G_MAIN));

    load->state.store(BlendLoadState::LOADING, std::memory_order_release);

    // Create BLI task pool and push work
    load->task_pool = BLI_task_pool_create(nullptr, TASK_PRIORITY_LOW);
    BLI_task_pool_push(load->task_pool, LoadThreadFunc, load.get(), false, nullptr);

    // Register in incoming buffer (lock ~nanoseconds)
    {
        std::lock_guard<std::mutex> lock(m_incoming_mutex);
        m_incoming.push_back(load);
    }

    return load;
}
```

### `CancelAll()` — Clean Shutdown

```cpp
void KX_BlendFileLoader::CancelAll() {
    // 1. Drain any incoming
    {
        std::lock_guard<std::mutex> lock(m_incoming_mutex);
        m_pending.insert(m_pending.end(),
            std::make_move_iterator(m_incoming.begin()),
            std::make_move_iterator(m_incoming.end()));
        m_incoming.clear();
    }

    // 2. Signal cancellation to all active loads
    for (auto &load : m_pending) {
        BlendLoadState expected = BlendLoadState::LOADING;
        load->state.compare_exchange_strong(expected, BlendLoadState::CANCELLING,
            std::memory_order_acq_rel);
    }

    // 3. Free all task pools (implicit join — may block briefly, acceptable at shutdown)
    for (auto &load : m_pending) {
        if (load->task_pool) {
            BLI_task_pool_free(load->task_pool);
            load->task_pool = nullptr;
        }
        // Free staging_main if it wasn't consumed
        if (load->staging_main) {
            BKE_main_free(load->staging_main);
            load->staging_main = nullptr;
        }
    }

    m_pending.clear();
}
```

---

## Python API

### `bge.logic.loadBlendAsync()`

```python
handle = bge.logic.loadBlendAsync(
    filepath,            # str: path to .blend (accepts '//')
    collections=[...],   # list[str]: ID_GR names
    objects=[...],       # list[str]: ID_OB names
    on_ready=callback,   # callable(handle) | None
    evaluate=True        # bool: True = depsgraph eval (modifiers applied)
                         #        False = raw mesh (no modifiers, faster)
)
```

### `KX_BlendLoadHandle` (returned object)

| Attribute | Type | Description |
|---|---|---|
| `.state` | `str` | `'loading'` \| `'integrating'` \| `'error'` \| `'done'` |
| `.progress` | `float` | 0.0-1.0 (always safe to read, see Phase 3) |
| `.error` | `str\|None` | Error message if `state == 'error'` |
| `.filepath` | `str` | Path to the loaded .blend |
| `.collections` | `list[str]` | Requested collection names |
| `.evaluate` | `bool` | Whether depsgraph evaluation was requested |
| `.cancel()` | `method` | Request cancellation (best-effort) |

**Lifetime guarantee:** The handle is always safe to query, even after integration or error.
The underlying `PendingBlendLoad` is kept alive by the handle's `shared_ptr` until the
handle is garbage collected.

**Cancel semantics:** `cancel()` is best-effort. It signals the worker to stop, but if the
worker is inside `BKE_blendfile_append` reading from disk, it will finish the I/O operation
before checking the cancellation flag. The full I/O cost is paid regardless. This is
documented behavior — cancelling avoids only the integration step.

**evaluate parameter:** When `True` (default), the full depsgraph is evaluated after merge,
applying all modifiers (subdivision, booleans, geometry nodes, etc.) to the loaded objects.
When `False`, objects use raw mesh data from the .blend — no modifiers are applied, but
integration is faster. Use `False` when assets have pre-applied modifiers or when modifiers
are not needed at runtime.

### Usage Example

```python
import bge

scene = bge.logic.getCurrentScene()

def on_trigger_enter(cont):
    own = cont.owner
    if cont.sensors['Near'].positive and not own.get('loading'):
        own['loading'] = True

        # Default: evaluate=True → modifiers applied at load time
        handle = bge.logic.loadBlendAsync(
            '//levels/zone_02.blend',
            collections=['Terrain_Z2', 'Enemies_Z2', 'Props_Z2'],
            on_ready=lambda h: on_zone_ready(own, h)
        )
        own['load_handle'] = handle

def on_trigger_enter_fast(cont):
    """For pre-baked assets where modifiers are already applied in .blend"""
    own = cont.owner
    if cont.sensors['Near'].positive and not own.get('loading'):
        own['loading'] = True

        handle = bge.logic.loadBlendAsync(
            '//props/decorations.blend',
            collections=['Bushes', 'Rocks'],
            on_ready=lambda h: on_props_ready(own, h),
            evaluate=False  # Raw mesh — no modifier eval, fastest path
        )

def on_zone_ready(trigger_obj, handle):
    print(f'Zone loaded: {handle.filepath}')

    # Objects are already in the scene as KX_GameObjects
    for obj in scene.objects:
        if obj.get('zone') == 'Z2':
            obj.visible = True
            obj.worldPosition = [0, 0, 0]

    trigger_obj['loading'] = False

# Safe to query handle at any time:
def on_update(cont):
    own = cont.owner
    handle = own.get('load_handle')
    if handle:
        if handle.state == 'done':
            print('Load completed previously')
        elif handle.state == 'error':
            print(f'Load failed: {handle.error}')
        # Even if the load was integrated 1000 frames ago,
        # this is safe — no use-after-free.
```

---

## Class Diagram

```
┌──────────────────────┐
│  KX_KetsjiEngine     │
├──────────────────────┤
│ - m_blendFileLoader  │───────┐
│                      │       │ owns (unique_ptr)
│ + GetBlendFileLoader()       │
└──────────────────────┘       │
                               ▼
                    ┌──────────────────────────────┐
                    │  KX_BlendFileLoader           │
                    ├──────────────────────────────┤
                    │ - m_pending                   │──┐ vector<shared_ptr>
                    │ - m_incoming                  │  │
                    │ - m_incoming_mutex            │  │
                    │ - m_maxIntegrationsPerFrame   │  │
                    │                              │  │
                    │ + RequestLoad()              │  │
                    │ + IntegratePendingLoads()    │  │
                    │ + CancelAll()                │  │
                    │ + Set/GetMaxIntegrations()   │  │
                    │ # LoadThreadFunc() [static]  │  │
                    └──────────────────────────────┘  │
                                                      ▼
                                            ┌────────────────────────┐
                                            │ PendingBlendLoad       │
                                            ├────────────────────────┤
                                            │ + filepath             │
                                            │ + filepath_resolved    │
                                            │ + collection_names     │
                                            │ + object_names         │
                                            │ + evaluate (bool)      │
                                            │ + staging_main         │
                                            │ + state (atomic)       │
                                            │ + task_pool (TaskPool*)│
                                            │ + error_msg            │
                                            │ + on_ready             │
                                            │ + target_scene         │
                                            │ + scene_generation     │
                                            └────────────────────────┘
                                              ▲                ▲
                                shared_ptr───┘                └───shared_ptr
                                              │
                                    ┌─────────┴───────────────┐
                                    │ KX_PyBlendLoadHandle    │
                                    ├─────────────────────────┤
                                    │ - m_load (shared_ptr)   │ ◄── SAFE lifetime
                                    │                         │
                                    │ + state (property)      │
                                    │ + error (property)      │
                                    │ + filepath (property)   │
                                    │ + collections (property)│
                                    │ + evaluate (property)   │
                                    │ + cancel() (method)     │
                                    └─────────────────────────┘
```

---

## Call Sequence

### Nominal Case (No Error)

```
[Python]  bge.logic.loadBlendAsync('level.blend', collections=['Enemies'])
    │
    ├──► [C++] BGE_loadBlendAsync()
    │       │
    │       ├─ Parse Python args
    │       ├─ std::vector<string> collections = {"Enemies"}
    │       │
    │       └──► KX_BlendFileLoader::RequestLoad()
    │               │
    │               ├─ make_shared<PendingBlendLoad>
    │               │  state = LOADING
    │               │
    │               ├─ BLI_path_abs(filepath)  ◄── resolve '//' on MAIN THREAD
    │               │
    │               ├─ BKE_main_new() → staging_main   ◄── on MAIN THREAD
    │               │  STRNCPY(staging_main->filepath, G_MAIN->filepath)
    │               │
    │               ├─ BLI_task_pool_create()
    │               ├─ BLI_task_pool_push(LoadThreadFunc, load.get())
    │               │  (NO work_and_wait ✓)
    │               │
    │               ├─ lock(m_incoming_mutex)
    │               │  m_incoming.push_back(load)
    │               │  unlock
    │               │
    │               └─ return shared_ptr ───────────────────┐
    │                                                        │
    ├─ new KX_PyBlendLoadHandle(shared_ptr)                 │
    └─ return handle (Python object) ──────────────────────┤
                                                             │
[Python continues, game loop runs normally]                │
                                                             │
[Worker Thread]                                            │
LoadThreadFunc(pool, load*) ◄──────────────────────────────┘
    │
    ├─ BKE_reports_init (LOCAL ReportList)
    ├─ BLO_blendhandle_from_file("level.blend")
    │  └─ handle = {...}  (block index)
    │
    ├─ staging_main already prepared by RequestLoad()
    │  └─ Has filepath from G_MAIN for library path resolution
    │
    ├─ LibraryLink_Params params {
    │    .bmain = staging_main,       ◄── ALL ops target staging
    │    .context.scene = nullptr,    ◄── NO instantiation
    │    NO FILE_RELPATH              ◄── avoids ONLY global in BLO
    │  }
    │
    ├─ BlendfileLinkAppendContext *lapp = ...(params)
    ├─ BKE_blendfile_link_append_context_library_add(lapp, ...)
    ├─ BKE_blendfile_link_append_context_item_add(lapp, "Enemies", ID_GR)
    │
    ├─ BKE_blendfile_link(lapp, &reports)        ◄── reads .blend → staging_main
    ├─ BKE_blendfile_append(lapp, &reports)       ◄── makes IDs local in staging_main
    │  (NO context_init_done, NO instantiate_loose, NO context_finalize)
    │
    ├─ BKE_blendfile_link_append_context_free(lapp)
    ├─ BLO_blendhandle_close(handle)
    ├─ BKE_reports_free(&reports)
    │
    └─ load->state.store(READY, memory_order_release) ─────────────┐
                                                                     │
[Main Thread - Next Frame]                                          │
KX_KetsjiEngine::NextFrame()                                        │
    │                                                               │
    ├─ m_converter->MergeAsyncLoads()  (legacy)                    │
    │                                                               │
    └──► KX_BlendFileLoader::IntegratePendingLoads(engine)         │
            │                                                       │
            ├─ lock(m_incoming_mutex)                              │
            │  drain m_incoming → m_pending                        │
            │  unlock (~nanoseconds)                               │
            │                                                       │
            ├─ for (load in m_pending):                            │
            │     if (load->state == READY) ◄───────────────────────┘
            │
            │  ── Validate scene_generation (ABA protection) ──
            │     if (scene destroyed) → ERROR, free staging_main
            │
            ├──► BLI_task_pool_free(load->task_pool)
            │    └─ Implicit join (~0 ns, thread already done)
            │
            │  ═══ 6-STEP POST-MERGE PIPELINE ═══
            │
            │  Step 0: Safety clear
            ├──► BKE_main_id_tag_all(G_MAIN, ID_TAG_DOIT, false)
            │
            │  Step 1: Tag staging IDs
            ├──► BKE_main_id_tag_all(staging_main, ID_TAG_DOIT, true)
            │
            │  Step 2: Merge + collect survivors
            ├──► BKE_main_merge(G_MAIN, &staging_main, report)
            │    └─ staging_main = nullptr (always freed)
            ├──► Scan G_MAIN for ID_TAG_DOIT → loaded_collections, loaded_objects
            │    └─ Clear tags after collection
            │
            │  Step 3: Link to scene hierarchy
            ├──► BKE_collection_child_add(G_MAIN, scene->master_coll, co)
            │    └─ for each loaded_collection
            ├──► BKE_collection_object_add(G_MAIN, master_coll, ob)
            │    └─ for loose objects not in any loaded_collection
            │
            │  Step 4: Sync collection caches
            ├──► BKE_main_collection_sync_remap(G_MAIN)
            │
            │  Step 5: Depsgraph eval (if evaluate=true)
            ├──► DEG_relations_tag_update(G_MAIN)
            ├──► DEG_id_tag_update(&co->id, ID_RECALC_ALL)  per new ID
            ├──► BKE_scene_graph_update_tagged(depsgraph, G_MAIN)
            │    └─ Evaluates ONLY tagged IDs (modifiers, transforms)
            │
            │  Step 6: Convert
            ├──► scene->ConvertBlenderCollection(co, libloading=true)
            │    └─ BL_ConvertBlenderObjects → SETLOOPER finds objects
            │       DEG_get_evaluated returns evaluated mesh
            │
            ├──► if (load->on_ready)
            │        PyObject_CallFunction(...) → Python callback
            │
            └──► load->state = DONE
                 (erased from m_pending next frame; shared_ptr keeps it alive
                  if Python handle still references it)
```

---

## Pre-Implementation Checklist

### ~~BLOCKER 1: TSAN Validation of `BLO_library_link_begin`~~ → RESOLVED ✅

Full source code audit of `readfile.cc` confirmed that `BLO_library_link_begin` →
`library_link_begin` operates entirely on `params->bmain` (staging_main). No access to
`G_MAIN` or other globals. The only global access in the entire BLO chain is in
`library_link_end` line 5361, conditional on `FILE_RELPATH` which we do not set.

See [BLO Thread Safety Audit](#blo-thread-safety-audit) for complete analysis.

**TSAN recommended post-implementation as validation step, not a blocker.**

### ~~BLOCKER 2: Name-based lookup post-merge~~ → RESOLVED ✅

Name-based lookup (`BKE_collection_find_by_name(G_MAIN, "Enemies")`) after merge finds
the OLD "Enemies", not the newly loaded "Enemies.001". Silent logic bug.

**Fix:** Identify IDs using `ID_TAG_DOIT` (Step 0-2b). Requires clearing `G_MAIN` tags
before merge to prevent pollution from previous operations. O(N) and immune to renaming.

### ~~BLOCKER 3: scene_generation validation~~ → RESOLVED ✅

Scene could be destroyed during async load, new scene could reuse same pointer (ABA).

**Fix:** `scene_generation` is a `uint64_t` monotonic counter checked in
`IntegratePendingLoads` before any integration step. If mismatch: set ERROR, free
staging_main, skip. 2^64 overflow is impossible.

### ~~BLOCKER 4: Depsgraph phase safety~~ → RESOLVED ✅

UPBGE uses both depsgraph and scenegraph. Depsgraph evaluates in `Render()` (inside
`RenderAfterCameraSetup` → EEVEE), AFTER `NextFrame()`.

**Fix:** Integration at Position ① (top of `NextFrame`) is safe. Step 5 forces
targeted depsgraph evaluation for new IDs only. EEVEE render sees fully evaluated data.

### ~~BLOCKER 5: Collection linking post-merge~~ → RESOLVED ✅

After `BKE_main_merge`, loaded IDs exist in G_MAIN but are "floating" — not linked to
any Scene's `master_collection`. `SETLOOPER` won't find them, depsgraph won't eval them.

**Fix:** Steps 3-4 of post-merge pipeline. `BKE_collection_child_add` links to
`scene->master_collection`, `BKE_main_collection_sync_remap` rebuilds caches.

### ~~VERIFY: `BKE_main_merge` Signature~~ → VERIFIED ✅

**Where:** `source/blender/blenkernel/BKE_main.hh:506`

**Confirmed signature:**
```cpp
void BKE_main_merge(Main *bmain_dst, Main **r_bmain_src, MainMergeReport &reports);
```

**Key behavior from docs:** On collision (same name, same library), existing ID in bmain_dst
wins. Source ID stays in staging_main → freed. `num_remapped_ids` counts collisions.
staging_main is ALWAYS freed by this call.

### ~~VERIFY: `ConvertBlenderCollection` Method Name~~ → VERIFIED ✅

**Where:** `KX_Scene.h:312`

**Confirmed signatures:**
```cpp
void ConvertBlenderObject(blender::Object *ob);
void ConvertBlenderObjectsList(std::vector<blender::Object *> objectslist, bool asynchronous);
void ConvertBlenderCollection(blender::Collection *co, bool asynchronous);
void ConvertBlenderAction(blender::bAction *act);
```

**Note:** The `asynchronous` parameter name in the existing API matches our use case
(passing `true` for async-loaded content). We use `ConvertBlenderCollection(co, true)`
for collections and `ConvertBlenderObject(ob)` for loose objects.

### ~~VERIFY: `KX_Scene::GetGeneration()` Availability~~ → RESOLVED ✅

**Confirmed:** KX_Scene does NOT have a generation counter. Must be added (~5 lines).

**`std::atomic<uint64_t>` is used** for `s_generation_counter` — even though the constructor
runs only on the main thread, the atomic guarantees correct initialization ordering and
prevents any compiler/hardware reordering without cost (atomic increment of a
non-contended variable ≈ 1ns):

```cpp
// In KX_Scene.h:
private:
    static std::atomic<uint64_t> s_generation_counter;  // Global monotonic counter (atomic for safety)
    uint64_t m_generation;
public:
    uint64_t GetGeneration() const { return m_generation; }
// In KX_Scene.cpp (Global scope):
    std::atomic<uint64_t> KX_Scene::s_generation_counter{0};
// In KX_Scene constructor:
    m_generation = ++s_generation_counter;
```

**ABA protection:** When a load is requested, `scene_generation` is captured. If the scene
is destroyed and a NEW scene happens to occupy the same pointer (ABA problem), its
generation counter will be different — the stale pointer is detected and the load errors
gracefully instead of writing into a destroyed scene.

### ~~VERIFY: G_MAIN->filepath in blenderplayer/standalone~~ → VERIFIED ✅

**Tested in all 5 execution modes (screenshots provided):**

| Mode | Maggie1 (m_maggie) | G_MAIN | Match? |
|---|---|---|---|
| Embedded (editor) | `C:\...\Untitled.blend` | `C:\...\Untitled.blend` | ✅ |
| Blenderplayer inside (from editor) | `C:\...\Untitled.blend~` | `C:\...\Untitled.blend~` | ✅ |
| Blenderplayer outside (external) | `E:\...\bin\Release\Untitled.blend` | `E:\...\bin\Release\Untitled.blend` | ✅ |
| Runtime (.exe export) | `E:\Test\Test\Test.exe.blend` | `E:\Test\Test\Test.exe.blend` | ✅ |
| Runtime + linked blend | Changes to linked.blend | Changes to linked.blend | ✅⚠️ |

**Note on runtime + linked blend:** G_MAIN changes when another .blend is loaded.
This is safe for our system because `staging_main->filepath` is copied once in
`RequestLoad()` and the worker uses that copy. If G_MAIN changes later, the worker
already has the correct filepath for resolving library paths within the .blend being loaded.

### POST-IMPLEMENTATION: TSAN Validation Run

**Action:** After implementation, run with TSAN enabled to catch:
- Indirect global access through IDTypeInfo callbacks
- do_versions code that might touch singletons
- colormanagement lazy initialization
- Any other hidden shared state

---

## Performance Metrics

### Before (Blocking)

```
Frame with level load:    ~200-500ms (FROZEN)
├─ Full .blend read:       ~150-300ms
├─ Sync conversion:        ~50-200ms
└─ Merge:                  negligible

FPS during load: 0-2 fps (visible freeze)
```

### After (Async, v3.1)

```
Frame with RequestLoad():           ~0.05ms (negligible)
├─ Create PendingBlendLoad:          ~0.01ms
├─ BLI_path_abs + BLI_exists:        ~0.005ms
├─ BKE_main_new + filepath copy:     ~0.01ms
├─ BLI_task_pool_create + push:      ~0.03ms
├─ m_incoming push (locked):         ~0.005ms
└─ Return handle:                    ~0.004ms

Frame with IntegratePendingLoads() — evaluate=true:   ~1-10ms
├─ Drain m_incoming (locked):        ~0.005ms
├─ BLI_task_pool_free:               ~0.001ms (thread already done)
├─ Step 0: Clear tags:               ~0.005ms
├─ Step 1: Tag staging IDs:          ~0.001ms
├─ Step 2: BKE_main_merge:           ~0.05-0.2ms (pointer ops)
├─ Step 3: Collection linking:       ~0.01-0.05ms
├─ Step 4: Collection sync:          ~0.1-0.5ms
├─ Step 5: DEG eval:                 ~0.1-5ms (proportional to content)
├─ Step 6: Convert:                  ~0.5-5ms (main cost)
└─ Python callback:                  variable (user code)

Frame with IntegratePendingLoads() — evaluate=false:  ~0.5-3ms
├─ Steps 1-4: same as above:         ~0.2-0.8ms
├─ Step 5: SKIPPED:                  0ms
├─ Step 6: Convert (raw mesh):       ~0.3-2ms
└─ Python callback:                  variable

FPS during load: 60 fps (no freeze)
Background I/O: 150-300ms in worker thread (invisible to game loop)
```

**Worst case with time-slicing (default: 1 integration/frame):**

If 3 collections finish simultaneously, they integrate over 3 consecutive frames
(~1-10ms each), instead of all at once (~3-30ms spike). At 60fps each frame has 16.6ms
budget. Even the expensive case (10ms with complex modifiers) = ~60% of budget for ONE
frame — acceptable for a loading event. Simple assets (1-3ms) = ~6-18% of budget.

---

## Time-Sliced Integration (Phases 2 + 2.1)

Steps 5 (DEG eval) and 6 (Convert) can take significant time with complex content.
Rather than executing the entire 6-step pipeline in one frame, the system supports
**incremental integration** across multiple frames, controlled by a per-frame time budget.

### Integration Budget System

```cpp
struct IntegrationBudget {
    float max_ms_per_frame = 4.0f;  // Default: 4ms budget (25% of 16.6ms @ 60fps)
    int max_objects_per_batch = 10;  // Objects to evaluate+convert per sub-step
};
```

### Modified IntegratePendingLoads — Incremental Mode

When a load enters READY state, integration proceeds in sub-steps across frames:

```
Frame N:   Steps 1-4 (tag, merge, link, sync)     ← Always complete in one frame (~0.3-0.8ms)
           Step 5a: DEG eval batch 1 (10 objects)  ← Within budget
           Step 6a: Convert batch 1                ← Within budget
           → state = INTEGRATING (new state)

Frame N+1: Step 5b: DEG eval batch 2 (10 objects)
           Step 6b: Convert batch 2
           → budget exceeded → pause

Frame N+2: Step 5c: DEG eval batch 3 (remaining)
           Step 6c: Convert batch 3
           → all batches done → state = DONE
```

### State Machine Update

```
                 RequestLoad()
    IDLE ─────────────────────► LOADING
                                   │
                     ┌─────────────┼──────────────┐
                     │             │              │
                  (cancel)    (success)       (failure)
                     │             │              │
                     ▼             ▼              ▼
                CANCELLING      READY          ERROR
                     │             │
                (thread ends) (Steps 1-4)
                     │             │
                     ▼             ▼
                   ERROR      INTEGRATING ◄── NEW: time-sliced eval+convert
                                   │
                              (all batches)
                                   │
                                   ▼
                                 DONE
```

### Batch Processing in IntegratePendingLoads

```cpp
// For loads in INTEGRATING state (partially integrated):
if (load->state.load() == BlendLoadState::INTEGRATING) {
    auto timer_start = BLI_time_now_seconds();

    while (load->next_batch_index < load->loaded_objects.size()) {
        // Batch boundaries
        int batch_start = load->next_batch_index;
        int batch_end = std::min(batch_start + budget.max_objects_per_batch,
                                 (int)load->loaded_objects.size());

        // Step 5: DEG eval this batch
        for (int i = batch_start; i < batch_end; i++) {
            DEG_id_tag_update(&load->loaded_objects[i]->id, ID_RECALC_ALL);
        }
        BKE_scene_graph_update_tagged(depsgraph, G_MAIN);

        // Step 6: Convert this batch
        for (int i = batch_start; i < batch_end; i++) {
            scene->ConvertBlenderObject(load->loaded_objects[i]);
        }

        load->next_batch_index = batch_end;

        // Check time budget
        double elapsed_ms = (BLI_time_now_seconds() - timer_start) * 1000.0;
        if (elapsed_ms >= budget.max_ms_per_frame) {
            break;  // Resume next frame
        }
    }

    // Collection conversion (after all objects in the collection are done)
    // Collections are converted as a unit once all their objects are evaluated
    if (load->next_batch_index >= load->loaded_objects.size()) {
        // All objects evaluated — now convert collections
        for (Collection *co : load->loaded_collections) {
            scene->ConvertBlenderCollection(co, true);
        }
        load->state.store(BlendLoadState::DONE);
    }
}
```

### PendingBlendLoad Additions for Incremental State

```cpp
struct PendingBlendLoad {
    // ... existing fields ...

    // === INCREMENTAL INTEGRATION STATE (main thread only) ===
    std::vector<Collection *> loaded_collections;  // Surviving IDs post-merge
    std::vector<Object *> loaded_objects;           // Surviving IDs post-merge
    int next_batch_index = 0;                       // Progress through object list
};
```

### Python API — Budget Configuration

```python
# Global configuration
bge.logic.setAsyncLoadBudget(
    max_ms_per_frame=4.0,      # ms budget for integration per frame
    max_objects_per_batch=10    # objects per eval+convert batch
)

# Per-handle query
handle.state       # Now includes 'integrating' state
handle.progress    # float 0.0-1.0 (based on next_batch_index / total)
```

---

## Progress Reporting (Phase 3)

### Worker Thread Progress

The worker thread reports loading progress via atomic float:

```cpp
struct PendingBlendLoad {
    // ... existing fields ...
    std::atomic<float> progress{0.0f};  // 0.0 = started, 1.0 = fully integrated
};
```

Progress stages:
```
0.0       → Worker started
0.0-0.4   → BLO reading .blend file (estimated from BLO report callbacks)
0.4-0.5   → BKE_blendfile_link + append
0.5       → READY (worker done, awaiting main thread)
0.5-0.9   → Steps 1-6 integration (proportional to batch progress)
0.9-1.0   → Python callback executing
1.0       → DONE
```

### Python API

```python
handle.progress        # float 0.0-1.0 (always safe to read)
handle.state           # 'loading' | 'integrating' | 'done' | 'error'

# Usage: loading bar in HUD
def update_loading_bar(cont):
    handle = cont.owner.get('load_handle')
    if handle:
        bar_width = handle.progress * 200  # pixels
        # Render bar...
```

### Implementation in Worker Thread

```cpp
// In LoadThreadFunc:
load->progress.store(0.0f, std::memory_order_relaxed);

// After BLO_blendhandle_from_file:
load->progress.store(0.1f, std::memory_order_relaxed);

// After BKE_blendfile_link:
load->progress.store(0.3f, std::memory_order_relaxed);

// After BKE_blendfile_append:
load->progress.store(0.5f, std::memory_order_relaxed);

// In IntegratePendingLoads (incremental):
float integration_progress = 0.5f + 0.4f * (next_batch_index / total_objects);
load->progress.store(integration_progress, std::memory_order_relaxed);

// After callback:
load->progress.store(1.0f, std::memory_order_relaxed);
```

**Cost:** atomic float store = ~1ns per update. Zero impact.

---

## Future Roadmap

### Deferred: Legacy System Cleanup (Phase 1.1)

**Action (post-implementation):** Remove broken legacy async loading system:
- `BL_Converter::MergeAsyncLoads()` / `FinalizeAsyncLoads()` / `AddScenesToMergeQueue()`
- `async_convert()` static function
- `KX_LibLoadStatus` class (replace with `KX_BlendLoadHandle`)
- `BL_Converter::m_mergequeue`, `m_status_map`, `ThreadInfo`
- Remove `m_converter->MergeAsyncLoads()` call from `KX_KetsjiEngine::NextFrame()`
- Remove `m_converter->FinalizeAsyncLoads()` call from `KX_KetsjiEngine::StopEngine()`

**Why deferred:** The legacy code doesn't interfere with the new system. Removing it is a
cleanup task that can happen safely after the new system is tested and validated.

### Deferred: Texture Streaming (Phase 4)

**Goal:** Large textures (4K, 8K) cause GPU upload stalls during conversion. Implement deferred upload.

**Proposed Solution:**
- `KX_TextureStreamer` queue for deferred `GPU_texture_create_from_image`.
- Objects use placeholder texture until real texture uploads.
- Per-frame upload budget (e.g., 2 textures or 4MB).

**Why deferred:** Integration with UPBGE's render engine (caching, bindless textures, material invalidation) is complex. The current system solves the main thread freeze; texture streaming is an optimization to smooth out frame hitches during `Convert`.

---

## Executive Summary

| Metric | Value |
|---|---|
| **New files** | 5 (all in `gameengine/Ketsji/`) |
| **New code lines** | ~1200 lines (headers + impl + Python) |
| **Changes to existing** | ~25 lines in 4 files |
| **Blender APIs used** | 100% public (BKE/BLO/BLI/DEG) |
| **Modifications to Blender** | 0 (zero) |
| **Threading model** | BLI_task_pool (consistent with UPBGE) |
| **Thread safety** | Verified via source audit — TSAN recommended post-impl |
| **Lifetime management** | shared_ptr (Python handle safe) |
| **Lock contention** | Double-buffer, ~nanosecond lock |
| **Post-merge pipeline** | 7 steps: clear → tag → merge → link → sync → eval → convert |
| **Merge collision handling** | ID_TAG_DOIT pattern (safe with G_MAIN pre-clear) |
| **Time-slicing** | Configurable ms budget + batch size for eval+convert |
| **Progress reporting** | atomic float, 0.0-1.0, stages from load to integration |
| **Modifier support** | Full depsgraph eval (default) or raw mesh (opt-in) |
| **Technical complexity** | Medium-High (threading, atomics, depsgraph) |
| **Performance impact** | High positive (eliminates freeze) |
| **Regression risk** | Low (new code is isolated) |
| **All VERIFYs** | ✅ Resolved (BKE_main_merge, KX_Scene, G_MAIN paths) |
| **Blockers** | 0 (zero) |
| **Estimated impl. time** | 5-7 days (core + phases + testing) |

---

## Changes from v1

| Issue | v1 | v2 | v2.2 | v3.1 | v3.2 |
|---|---|---|---|---|---|
| BLO thread safety | "Verify later" | TSAN spike required | Source audit: VERIFIED | Downgraded to "by static audit, TSAN recommended" | — |
| Handle lifetime | `unique_ptr` + raw | `shared_ptr` shared | — | — | — |
| Worker safety | "Verify later" | scene=nullptr | + NO FILE_RELPATH | — | — |
| Path resolution | Unclear thread | Main thread | + BLI_exists check | — | — |
| staging_main | In worker | In worker | **Main thread** | — | — |
| Multi-scene | Not addressed | target_scene | ABA: uint64_t monotonic | **`std::atomic<uint64_t>` monotonic counter in KX_Scene** | **Confirmed: `std::atomic<uint64_t>` (was incorrectly labeled in v3.1)** |
| Integration point | LogicEndFrame | LogicEndFrame | **Top of NextFrame** | — | — |
| Post-merge pipeline | Not addressed | BKE_main_merge only | 6-step pipeline | **7-step: Pre-clear tags in G_MAIN** | — |
| Collision ID logic | Not addressed | Not addressed | Pointer collection | **ID_TAG_DOIT (safer)** | — |
| Depsgraph | Not addressed | Not addressed | Forced eval + raw mode | — | — |
| Texture Streaming | Not addressed | Not addressed | Phase 4 | **Deferred to Roadmap** | — |
| Modified files count | — | — | — | ~~5~~ (error) | **4 (corrected)** |
| Blockers | Multiple unknowns | 1 (TSAN) | 0 | **0 — ready to implement** | **0** |

---

## References

- **Design document:** `UPBGE_AsyncBlendLoading.docx`
- **Architecture v1:** `UPBGE_AsyncBlendLoading_v1.md`
- **Blender PR #151907:** Introduction of `BKE_main_merge`
- **Source audit (BKE layer):** `blendfile_link_append.cc` — confirmed `params->bmain` pattern, zero `G_MAIN` access
- **Source audit (BLO layer):** `readfile.cc` — confirmed `library_link_begin`, `link_named_part`, `library_link_end` all use `mainvar`/`params->bmain`. Single `BKE_main_blendfile_path_from_global()` at line 5361 gated behind `FILE_RELPATH` flag
- **Source audit (BLO entry):** `readblenentry.cc` — `BLO_blendhandle_from_file` is pure file I/O, no globals
- **Frame loop analysis:** `KX_KetsjiEngine.cpp` — `NextFrame()` line 488: `MergeAsyncLoads()` position confirmed as integration point. `Render()` line 781: depsgraph eval in EEVEE (after NextFrame)
- **Legacy system analysis:** `BL_Converter.cpp` — `async_convert` broken (runs depsgraph on worker thread), `MergeAsyncLoads` operates at KX_Scene level not Blender Main level
- **APIs used:**
  - `source/blender/blenloader/BLO_readfile.hh` (BLO_blendhandle_*, BLO_library_link_*)
  - `source/blender/blenkernel/BKE_blendfile_link_append.hh` (BKE_blendfile_link, BKE_blendfile_append)
  - `source/blender/blenkernel/BKE_main.hh` (BKE_main_merge, BKE_main_new, BKE_main_free, BKE_main_id_tag_all)
  - `source/blender/blenkernel/BKE_collection.hh` (BKE_collection_child_add, BKE_collection_object_add, BKE_collection_has_object_recursive, BKE_main_collection_sync_remap)
  - `source/blender/blenkernel/BKE_scene.hh` (BKE_scene_graph_update_tagged)
  - `source/blender/depsgraph/DEG_depsgraph.hh` (DEG_id_tag_update, DEG_relations_tag_update)
  - `source/blender/blenlib/BLI_task.h` (BLI_task_pool_*)

---

**Maintainer:** UPBGE Team
**Version:** 3.2
**Date:** 2026-02-19
**Reviewed by:** Claude (Architecture Review + BLO Source Audit) + Gemini pro
**Note:** v3.2 restores `std::atomic<uint64_t>` monotonic counter for multi-scene ABA protection, and corrects modified file count to 4.
