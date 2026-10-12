# UPBGE Async Selective Blend Loading — Implementation Architecture v3.3

## Table of Contents

1. [Overview](#overview)
2. [File Structure](#file-structure)
3. [Execution Flow](#execution-flow)
4. [Core Components](#core-components)
5. [Two-Stage Integration Pipeline](#two-stage-integration-pipeline)
6. [BLO Thread Safety Audit](#blo-thread-safety-audit)
7. [Threading Model](#threading-model)
8. [Integration Points](#integration-points)
9. [Batch Conversion](#batch-conversion)
10. [Unloading](#unloading)
11. [Python API](#python-api)
12. [Class Diagram](#class-diagram)
13. [Call Sequence](#call-sequence)
14. [Pre-Implementation Checklist](#pre-implementation-checklist)
15. [Performance Model](#performance-model)
16. [Batching and Budget (replaces Phases 2 + 2.1)](#batching-and-budget-replaces-phases-2--21)
17. [Progress Reporting (Phase 3)](#progress-reporting-phase-3)
18. [Future Roadmap](#future-roadmap)

All source references (`file:line`) in this document are anchored to UPBGE master
`e7c5aac` (2026-10-12) unless stated otherwise.

---

## Overview

### Goal

Eliminate game loop freezing during external `.blend` file loading via:

- **Selective loading** by collection/object using `BKE_blendfile_link` + `BKE_blendfile_append`
- **Real background I/O** on one dedicated loader thread (`BLI_task_pool_create_background_serial`),
  outside the TBB arena, so the main thread can never end up executing the load
- **Two-stage, batched integration** on the main thread: merge + hidden link (frame N),
  conversion + reveal (frame N+1)
- **No forced depsgraph evaluation** — the evaluation already performed by the regular render
  pass is reused, never duplicated
- **Explicit unload** (`handle.free()`) with exact ownership of every loaded ID

### Design Principles

```
┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #1: Performance First                                 │
│ - Main thread NEVER waits for the loader thread and can never   │
│   execute its work (dedicated thread, not a TBB task)           │
│ - IntegratePendingLoads() is O(1) when nothing is pending       │
│ - std::atomic<BlendLoadState> → zero locks on the hot path      │
│ - Expensive phases are split across frames, never repeated:     │
│   one relations rebuild and one conversion pass per batch       │
│ - Budget expressed in objects (known before work starts),       │
│   not in milliseconds (only known after it is spent)            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #2: Isolation + Correct Integration                   │
│ - Loader thread operates ONLY on staging_main                   │
│ - Loader thread must not reach code that takes G.main           │
│   implicitly (see audit) — TSAN is a blocker, not an extra      │
│ - Main thread: merge → hidden link (stage 1),                   │
│   convert → reveal (stage 2)                                    │
│ - ReportList is local to the loader thread                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #3: Minimal Invasiveness                              │
│ - DO NOT touch Blender code (public BKE/BLO/BLI/DEG APIs only)  │
│ - Every change lives in source/gameengine/                      │
│ - Changes to existing files are small and mechanical            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRINCIPLE #4: Safe Lifetime Management                          │
│ - shared_ptr<PendingBlendLoad> held by the loader, the Python   │
│   handle AND the queued task (lifetime independent of timing)   │
│ - PyObject references are released only on the main thread      │
│ - One background-serial pool per loader, joined in Shutdown()   │
└─────────────────────────────────────────────────────────────────┘
```

---

## File Structure

### New Files (gameengine/Ketsji/)

```
gameengine/Ketsji/
├── KX_BlendFileLoader.h              [~170 lines]
│   ├── enum class BlendLoadState     (LOADING, CANCEL_REQUESTED, READY, INTEGRATING,
│   │                                  DONE, ERROR, CANCELLED, FREED)
│   ├── struct PendingBlendLoad       (shared_ptr-managed, atomic state, ownership snapshot)
│   └── class KX_BlendFileLoader      (RequestLoad, IntegratePendingLoads, Cancel, Free,
│                                      Shutdown, FreeLoadedIds)
│
├── KX_BlendFileLoader.cpp            [~520 lines]
│   ├── RequestLoad()                 (resolves path, creates staging_main, pushes task)
│   ├── LoadThreadFunc()              (loader thread: link + append + ownership snapshot)
│   ├── IntegratePendingLoads()       (main: stage dispatch + terminal-load reaping)
│   ├── StageMergeAndLinkHidden()     (frame N)
│   ├── StageConvertAndReveal()       (frame N+1)
│   ├── Unload()                      (cancel-while-integrating, handle.free(), exit)
│   └── Shutdown() / FreeLoadedIds()  (engine stop)
│
├── KX_PyBlendLoadHandle.h            [~60 lines]
│   └── class KX_PyBlendLoadHandle    (holds shared_ptr<PendingBlendLoad>)
│
└── KX_PyBlendLoadHandle.cpp          [~220 lines]
    ├── pyattr_get_state()            ('loading' | 'integrating' | 'done' | 'error' |
    │                                  'cancelled' | 'freed')
    ├── pyattr_get_progress()         (float 0.0-1.0)
    ├── pyattr_get_error()            (str | None, read only in terminal states)
    ├── pyattr_get_filepath()         (str)
    ├── pyattr_get_collections()      (list[str], requested names)
    ├── pyattr_get_objects()          (list[KX_GameObject], built on access)
    ├── py_cancel()                   (main thread, see Cancel semantics)
    └── py_free()                     (main thread, unload)
```

### Modified Files (PATCH)

All modified files are inside `source/gameengine/`. No Blender file is modified.

```diff
gameengine/Ketsji/KX_KetsjiEngine.h
+ class KX_BlendFileLoader;
+ private:
+   std::unique_ptr<KX_BlendFileLoader> m_blendFileLoader;
+   bool m_freeAsyncLoadsOnExit = false;
+ public:
+   KX_BlendFileLoader *GetBlendFileLoader();
+   void SetFreeAsyncLoadsOnExit(bool free) { m_freeAsyncLoadsOnExit = free; }
  enum {
    ...
    tc_services,
+   tc_asyncload,  // stage 1 + stage 2 of async blend integration
    ...

gameengine/Ketsji/KX_KetsjiEngine.cpp
+ #include "KX_BlendFileLoader.h"
  // Constructor:
+ m_blendFileLoader = std::make_unique<KX_BlendFileLoader>();
  // NextFrame(), after the 'times.frames == 0' early return, BEFORE the tick loop:
+ m_logger.StartLog(tc_asyncload);
+ m_blendFileLoader->IntegratePendingLoads(this);
+ m_logger.StartLog(tc_services);
  // StopEngine(), first statement inside 'if (m_bInitialized)':
+ m_blendFileLoader->Shutdown();
  // StopEngine(), after the scene removal loop:
+ if (m_freeAsyncLoadsOnExit) {
+   m_blendFileLoader->FreeLoadedIds();
+ }
  // m_profileLabels: add the "AsyncLoad:" label at the tc_asyncload index.

gameengine/Ketsji/KX_Scene.h / KX_Scene.cpp
+ static std::atomic<uint64_t> s_generation_counter;  // ABA guard, see checklist
+ uint64_t m_generation;
+ uint64_t GetGeneration() const;
+ void ConvertBlenderObjectsBatch(const blender::Set<const blender::Object *> &batch);
  // ConvertBlenderObject(ob) now builds a one-element Set and calls the batch path.

gameengine/Converter/BL_DataConversion.h / BL_DataConversion.cpp
- blender::Object *runtime_converted_object,
+ const blender::Set<const blender::Object *> *runtime_batch,
  // ~15 mechanical predicate replacements + hoisted collection remap (see Batch Conversion).

gameengine/Converter/BL_SceneConverter.cpp
  // FindGameObject(): find() instead of operator[] (no insertion on lookup).

gameengine/Converter/BL_Converter.h / BL_Converter.cpp
+ void RemoveConvertedData(blender::FunctionRef<bool(const blender::ID *)> is_owned);
  // Body extracted from FreeBlendFile(); FreeBlendFile() calls it with IS_TAGGED.

gameengine/Ketsji/KX_PythonInit.cpp
+ gLoadBlendAsync(), gSetAsyncLoadBudget(), gGetAsyncLoadBudget() + method table entries.

gameengine/Ketsji/KX_PythonInitTypes.cpp
+ PyType_Ready_Attr(dict, KX_PyBlendLoadHandle, init_getset);

gameengine/Launcher/LA_BlenderLauncher.cpp
+ m_ketsjiEngine->SetFreeAsyncLoadsOnExit(!(m_startScene->gm.flag & GAME_USE_UNDO));

gameengine/Ketsji/CMakeLists.txt
+   KX_BlendFileLoader.cpp
+   KX_BlendFileLoader.h
+   KX_PyBlendLoadHandle.cpp
+   KX_PyBlendLoadHandle.h
```

**Totals:** 4 new files (~970 lines). 13 existing files modified: ~140 lines written (~40 of them
mechanical predicate replacements), plus ~150 lines moved unchanged inside `BL_Converter.cpp`.

---

## Execution Flow

### Complete Load Lifecycle

```
PYTHON USER CODE              MAIN THREAD                        LOADER THREAD (dedicated)
═══════════════════          ═══════════════════                ══════════════════════════

handle = loadBlendAsync()
    │
    ├──► RequestLoad()
    │       ├─ resolve '//' path, BLI_exists()
    │       ├─ BKE_main_new() + BKE_main_init_from_reference(staging, G_MAIN)
    │       │  (filepath AND working colorspace)
    │       ├─ state = LOADING
    │       ├─ BLI_task_pool_push(m_pool, LoadThreadFunc,
    │       │                     new shared_ptr(load), true, LoadTaskFree)
    │       ├─ m_pending.append(load)          (main thread only, no mutex)
    │       └─ return shared_ptr ─────────┐
    │                                      │
    ▼                                      │       LoadThreadFunc()  (serial FIFO queue)
Game loop continues                        │           ├─ CANCEL_REQUESTED? → CANCELLED, exit
                                           │           ├─ BKE_reports_init(RPT_STORE)
                                           │           ├─ context_new(params on stack)
                                           │           ├─ library_add(lapp, path, nullptr)
                                           │           ├─ item_add() per requested name
                                           │           ├─ BKE_blendfile_link()
                                           │           ├─ BKE_blendfile_append()
                                           │           ├─ roots = item_newid_get() per item
                                           │           ├─ context_free()
                                           │           ├─ ownership snapshot of staging:
                                           │           │  force_merge_ids, owned_ids,
                                           │           │  owned_uids, num_objects
                                           │           └─ CAS LOADING → READY (release)
                                           │                    │
═══ FRAME N — NextFrame(), before the tick loop ═══════════════════════════════════════════
                                           │                    │
IntegratePendingLoads() ◄──────────────────┘◄───────────────────┘
    ├─ stage == IDLE → GatherReadyBatch()
    │     READY loads of one live target scene, up to max_objects_per_batch (>= 1 load)
    └─ StageMergeAndLinkHidden()
          ├─ BKE_main_merge(G_MAIN, &force_merge_ids, &staging_main)   per load
          ├─ ntree tag + BKE_main_ensure_invariants(new node trees)
          ├─ container = BKE_collection_add(master_collection, "AsyncLoad")  per load
          ├─ roots → container (no-sync variants), one lazy collection sync
          ├─ container LayerCollection |= LAYER_COLLECTION_HIDE
          ├─ DEG_relations_tag_update(G_MAIN)
          └─ state = INTEGRATING, stage = AWAIT_EVAL

Tick loop (logic, physics) — loaded objects not yet converted, not visible.

Render()  (existing code, KX_Scene.cpp:727)
    └─ BKE_scene_graph_update_tagged()
          ├─ relations rebuild (once for the whole batch)
          └─ evaluation of the new objects (evaluated, NOT drawn: hidden layer collection)

═══ FRAME N+1 — NextFrame(), before the tick loop ═════════════════════════════════════════

IntegratePendingLoads()
    └─ stage == AWAIT_EVAL → StageConvertAndReveal()
          ├─ revalidate target scene (pointer + generation, no dereference before match)
          ├─ scene->ReinitBlenderContextVariables()   (conversion reads the context depsgraph)
          ├─ BKE_scene_graph_evaluated_ensure()       (O(1) when Render N already evaluated)
          ├─ scene->ConvertBlenderObjectsBatch(batch) (one SETLOOPER pass)
          ├─ container LayerCollection &= ~LAYER_COLLECTION_HIDE
          ├─ DEG_id_tag_update_ex(scene, ID_RECALC_BASE_FLAGS)  (no relations rebuild)
          └─ state = DONE, stage = IDLE
    └─ ReapTerminalLoads() → on_finish(handle) callbacks, after all iteration

Tick loop — new objects take part in logic and physics.
Render() — new objects drawn for the first time.
```

---

## Core Components

Code in this document omits the `blender::` qualifier for brevity.

### 1. `BlendLoadState` — State Machine

```cpp
enum class BlendLoadState : uint8_t {
  LOADING,           // Queued or running on the loader thread
  CANCEL_REQUESTED,  // Cancel asked while LOADING; the loader thread finishes it as CANCELLED
  READY,             // Loader thread done; staging_main complete; waiting for a batch slot
  INTEGRATING,       // Stage 1 done (merged into G_MAIN, linked hidden); stage 2 next frame
  DONE,              // Converted and revealed
  ERROR,             // Terminal; error_msg is valid
  CANCELLED,         // Terminal; nothing of this load remains in G_MAIN
  FREED,             // Terminal; unloaded after DONE (handle.free())
};
```

State transitions:
```
                    RequestLoad()
                         │            (file missing / nothing requested)
                         ▼ ─────────────────────────────────────► ERROR
                      LOADING ──(cancel, CAS)──► CANCEL_REQUESTED
                         │                              │
            ┌────────────┤                     (loader thread exits,
            │            │                      frees staging_main)
       (failure)   (success, CAS)                       │
            │            │                              ▼
            ▼            ▼                          CANCELLED
          ERROR        READY ──(cancel)──────────────► CANCELLED
                         │   └─(scene removed)───────► ERROR
                     (stage 1, frame N)
                         ▼
                    INTEGRATING ──(cancel: unload)───► CANCELLED
                         │      └─(scene removed: unload)► ERROR
                     (stage 2, frame N+1)
                         ▼
                       DONE ──(handle.free(): unload)──► FREED
```

| Transition | Thread | Mechanism |
|---|---|---|
| `LOADING → READY` | loader | `compare_exchange_strong` (acq_rel) — a racing cancel is never overwritten |
| `LOADING → ERROR` | loader | release store (terminal either way if it races with a cancel) |
| `LOADING → CANCEL_REQUESTED` | main | `compare_exchange_strong` (acq_rel) |
| `CANCEL_REQUESTED → CANCELLED` | loader | release store |
| every transition out of `READY` and later | main | relaxed store (the loader thread is finished) |

### 2. `PendingBlendLoad` — Load State (shared_ptr managed)

```cpp
struct PendingBlendLoad {
  /* === INPUT: written by the main thread before the task push (published by the queue) === */
  std::string filepath;                       // As given (may contain '//')
  std::string filepath_resolved;              // Absolute, resolved on the main thread
  std::vector<std::string> collection_names;  // ID_GR names
  std::vector<std::string> object_names;      // ID_OB names
  KX_Scene *target_scene = nullptr;           // Never dereferenced before validation
  uint64_t scene_generation = 0;              // ABA guard
  PyObject *on_finish = nullptr;              // Owned reference, released on the main thread

  /* === LOADER OUTPUT: published by the CAS to READY (release) === */
  Main *staging_main = nullptr;               // Created by RequestLoad(), consumed by the merge
  Vector<Collection *> root_collections;      // Requested ID_GR, resolved by pointer
  Vector<Object *> root_objects;              // Requested ID_OB, resolved by pointer
  Set<ID *> force_merge_ids;                  // Every local id of staging_main (see stage 1)
  Set<uint32_t> owned_uids;                   // session_uid of owned ids (+ container, stage 1)
  int num_objects = 0;                        // Budget unit
  std::string error_msg;                      // Read only once the state is terminal

  /* === MAIN THREAD ONLY (after READY) === */
  Collection *container = nullptr;            // Per-load "AsyncLoad" collection

  /* === SYNCHRONIZATION === */
  std::atomic<BlendLoadState> state{BlendLoadState::LOADING};
  std::atomic<float> progress{0.0f};

  ~PendingBlendLoad()
  {
    /* May run on the loader thread when LoadTaskFree() drops the last reference. Anything that
     * needs the main thread or the GIL must already have been released. */
    BLI_assert(on_finish == nullptr);
    BLI_assert(staging_main == nullptr);
  }
};
```

**Ownership:** `std::shared_ptr`, held by the loader (`m_pending`, then `m_loaded`), by every
`KX_PyBlendLoadHandle`, and by the queued task itself (`new shared_ptr` as taskdata, deleted by
`LoadTaskFree`). The struct therefore outlives the loader thread's last access regardless of
what the main thread does. Cost: one atomic increment and decrement per load.

**Thread safety:** a field is written by exactly one phase owner. Input fields are published
to the loader thread by the task queue's internal mutex; loader output is published to the main
thread by the release CAS to `READY` and read after an acquire load.

### 3. `KX_BlendFileLoader` — Central Manager

```cpp
class KX_BlendFileLoader {
 public:
  KX_BlendFileLoader();   // m_pool = BLI_task_pool_create_background_serial(nullptr, TASK_PRIORITY_LOW)
  ~KX_BlendFileLoader();  // Frees m_pool only if Shutdown() never ran (engine never initialized)

  std::shared_ptr<PendingBlendLoad> RequestLoad(const std::string &filepath,
                                                std::vector<std::string> collections,
                                                std::vector<std::string> objects,
                                                PyObject *on_finish,
                                                KX_Scene *target_scene);

  /* Once per NextFrame(), outside the tick loop. */
  void IntegratePendingLoads(KX_KetsjiEngine *engine);

  void Cancel(PendingBlendLoad &load);  // Main thread
  void Free(PendingBlendLoad &load);    // Main thread; DONE -> FREED, otherwise == Cancel()

  /* StopEngine(), before scenes are removed: cancel, join the loader thread, drop staging
   * Mains and PyObject references (takes the GIL itself). */
  void Shutdown();
  /* StopEngine(), after scenes are removed: delete every ID still owned by a load (embedded
   * mode without GAME_USE_UNDO only). */
  void FreeLoadedIds();

  void SetMaxObjectsPerBatch(int max) { m_maxObjectsPerBatch = std::max(1, max); }
  int GetMaxObjectsPerBatch() const { return m_maxObjectsPerBatch; }

 private:
  enum class Stage : uint8_t { IDLE, AWAIT_EVAL };

  static void LoadThreadFunc(TaskPool *__restrict pool, void *taskdata);
  static void LoadTaskFree(TaskPool *__restrict pool, void *taskdata);
  static void FreeMainTask(TaskPool *__restrict pool, void *taskdata);

  bool GatherReadyBatch(KX_KetsjiEngine *engine);
  void StageMergeAndLinkHidden();
  void StageConvertAndReveal(KX_KetsjiEngine *engine);
  void Unload(PendingBlendLoad &load);
  void ReleaseStagingAsync(PendingBlendLoad &load);
  void ReapTerminalLoads();

  TaskPool *m_pool = nullptr;  // One dedicated pthread, FIFO; never a TBB task

  /* Main thread only, hence no mutex: RequestLoad() and IntegratePendingLoads() both run on
   * the main thread, and Python callbacks run after all iteration (ReapTerminalLoads()). */
  Vector<std::shared_ptr<PendingBlendLoad>> m_pending;  // LOADING .. INTEGRATING, terminal until reaped
  Vector<std::shared_ptr<PendingBlendLoad>> m_loaded;   // DONE, until freed
  Vector<std::shared_ptr<PendingBlendLoad>> m_batch;    // In flight between stage 1 and stage 2

  Stage m_stage = Stage::IDLE;
  KX_Scene *m_batchScene = nullptr;
  uint64_t m_batchGeneration = 0;
  int m_maxObjectsPerBatch = 256;  // Initial value only; calibrate with tc_asyncload (checklist)
};
```

**Lifetime:** one instance per `KX_KetsjiEngine`. `Shutdown()` is called explicitly from
`StopEngine()`, before Python is finalized. The destructor only covers an engine that was never
initialized: it frees the (thread-less) pool and asserts that no `PyObject` is still held.

### 4. `LoadThreadFunc` — Loader Thread (CRITICAL)

```cpp
void KX_BlendFileLoader::LoadThreadFunc(TaskPool * /*pool*/, void *taskdata)
{
  PendingBlendLoad *load = static_cast<std::shared_ptr<PendingBlendLoad> *>(taskdata)->get();

  /* Serial queue: this load may have waited behind others. A cancelled load exits before any
   * I/O, which is also what makes Shutdown() fast: queued tasks drain at this check. */
  if (load->state.load(std::memory_order_acquire) == BlendLoadState::CANCEL_REQUESTED) {
    FinishOnLoaderThread(*load, BlendLoadState::CANCELLED);
    return;
  }
  load->progress.store(0.1f, std::memory_order_relaxed);

  ReportList reports;
  /* The second argument is a ReportListFlags mask, not a severity: without RPT_STORE nothing is
   * stored and BKE_reports_contain() can never return true (report.cc). */
  BKE_reports_init(&reports, RPT_STORE);

  /* Params live on this stack frame: the context keeps a pointer to them and is freed below.
   * - bmain = staging_main: every allocation of link/append targets the private Main.
   * - No FILE_RELPATH: the only BLO-level global access is gated behind it (readfile.cc:5676).
   * - context.scene = nullptr: no instantiation here; stage 1 instantiates on the main thread. */
  LibraryLink_Params params{};
  params.bmain = load->staging_main;
  params.flag = BLO_LIBLINK_APPEND_RECURSIVE;

  BlendfileLinkAppendContext *lapp = BKE_blendfile_link_append_context_new(&params);
  /* nullptr handle: the context opens the file and owns the handle. library_add() always steals
   * a given handle (BKE_blendfile_link_append.hh:233), so closing one here would double-close.
   * It returns void; the single library has index 0. */
  BKE_blendfile_link_append_context_library_add(lapp, load->filepath_resolved.c_str(), nullptr);

  Vector<BlendfileLinkAppendContextItem *> items;
  for (const std::string &name : load->collection_names) {
    items.append(BKE_blendfile_link_append_context_item_add(lapp, name.c_str(), ID_GR, nullptr));
  }
  for (const std::string &name : load->object_names) {
    items.append(BKE_blendfile_link_append_context_item_add(lapp, name.c_str(), ID_OB, nullptr));
  }
  for (BlendfileLinkAppendContextItem *item : items) {
    BKE_blendfile_link_append_context_item_library_index_enable(lapp, item, 0);
  }

  /* init_done() and finalize() are deliberately NOT called: both execute Python handlers
   * (BKE_CB_EVT_BLENDIMPORT_PRE/POST). finalize() also does real work (node tree tagging and
   * BKE_main_ensure_invariants()), which stage 1 redoes on the main thread for loaded trees. */
  BKE_blendfile_link(lapp, &reports);
  load->progress.store(0.3f, std::memory_order_relaxed);
  BKE_blendfile_append(lapp, &reports);
  load->progress.store(0.45f, std::memory_order_relaxed);

  /* Requested roots are resolved by pointer, never by name: local names may change during the
   * merge (".001"). A requested root that is missing or still linked after append is an error. */
  std::string missing;
  const int64_t num_collections = int64_t(load->collection_names.size());
  for (const int64_t i : items.index_range()) {
    ID *id = BKE_blendfile_link_append_context_item_newid_get(lapp, items[i]);
    if (id == nullptr || ID_IS_LINKED(id)) {
      missing += (i < num_collections) ? load->collection_names[i] :
                                         load->object_names[i - num_collections];
      missing += ' ';
      continue;
    }
    if (id->id_type() == ID_GR) {
      load->root_collections.append(reinterpret_cast<Collection *>(id));
    }
    else {
      load->root_objects.append(reinterpret_cast<Object *>(id));
    }
  }
  BKE_blendfile_link_append_context_free(lapp);

  /* Ownership snapshot, built while staging_main is still private to this thread.
   * BKE_main_merge() matches LOCAL ids by name only (main.cc, are_ids_from_different_mains_
   * matching(): two local ids with equal names match), so every local id is force-merged:
   * moved and renamed (".001") instead of being silently replaced by an unrelated G_MAIN id.
   * Linked ids and Libraries keep the normal dedup semantics (same file => same data). */
  ID *id;
  FOREACH_MAIN_ID_BEGIN (load->staging_main, id) {
    if (!ID_IS_LINKED(id) && id->id_type() != ID_LI) {
      load->force_merge_ids.add_new(id);
      load->owned_uids.add_new(id->session_uid);
      load->num_objects += int(id->id_type() == ID_OB);
    }
  }
  FOREACH_MAIN_ID_END;

  if (!missing.empty() || BKE_reports_contain(&reports, RPT_ERROR)) {
    if (!missing.empty()) {
      load->error_msg = "Not found or not appendable: " + missing;
    }
    if (char *msg = BKE_reports_string(&reports, RPT_WARNING)) {
      load->error_msg += msg;
      MEM_freeN(msg);
    }
    BKE_reports_free(&reports);
    FinishOnLoaderThread(*load, BlendLoadState::ERROR);
    return;
  }
  BKE_reports_free(&reports);

  /* Publish with CAS, never a plain store: a cancel that arrived after the entry check must not
   * be overwritten by READY. Release ordering publishes staging_main and every output field. */
  BlendLoadState expected = BlendLoadState::LOADING;
  if (load->state.compare_exchange_strong(
          expected, BlendLoadState::READY, std::memory_order_acq_rel))
  {
    load->progress.store(0.5f, std::memory_order_relaxed);
    return;
  }
  FinishOnLoaderThread(*load, BlendLoadState::CANCELLED);  // expected == CANCEL_REQUESTED
}

/* Terminal exit on the loader thread. BKE_main_free() clears with LIB_ID_FREE_NO_DEG_TAG
 * (main.cc, BKE_main_clear()), so freeing the private Main touches no depsgraph state.
 * After the final store the loader thread never touches `load` again. */
static void FinishOnLoaderThread(PendingBlendLoad &load, const BlendLoadState terminal)
{
  load.force_merge_ids.clear_and_shrink();
  load.root_collections.clear_and_shrink();
  load.root_objects.clear_and_shrink();
  if (load.staging_main) {
    BKE_main_free(load.staging_main);
    load.staging_main = nullptr;
  }
  load.state.store(terminal, std::memory_order_release);
}

void KX_BlendFileLoader::LoadTaskFree(TaskPool * /*pool*/, void *taskdata)
{
  /* Drops the task's reference. If it is the last one, ~PendingBlendLoad runs on the loader
   * thread, which is why the destructor may only free plain memory. */
  delete static_cast<std::shared_ptr<PendingBlendLoad> *>(taskdata);
}

void KX_BlendFileLoader::FreeMainTask(TaskPool * /*pool*/, void *taskdata)
{
  BKE_main_free(static_cast<Main *>(taskdata));
}

void KX_BlendFileLoader::ReleaseStagingAsync(PendingBlendLoad &load)
{
  /* A cancelled or orphaned READY load still owns a full staging Main. Freeing it can take
   * milliseconds, so it is queued behind the loads on the loader thread instead. */
  BLI_task_pool_push(m_pool, FreeMainTask, std::exchange(load.staging_main, nullptr), false, nullptr);
  load.force_merge_ids.clear_and_shrink();
}
```

**Thread safety:** see [BLO Thread Safety Audit](#blo-thread-safety-audit). The static audit is
**not** sufficient on its own: implicit `G.main` access exists in reachable BKE code. TSAN on a
dedicated test corpus is a blocker before merging.

### 5. `IntegratePendingLoads` — Main Thread (Stage Dispatch)

```cpp
void KX_BlendFileLoader::IntegratePendingLoads(KX_KetsjiEngine *engine)
{
  BLI_assert(BLI_thread_is_main());
  if (m_pending.is_empty() && m_stage == Stage::IDLE) {
    return;  // Common case: one branch per frame.
  }
  /* At most one batch in flight: overlapping stage 1 of batch k+1 with stage 2 of batch k would
   * put back into one frame exactly the costs this pipeline splits. */
  if (m_stage == Stage::AWAIT_EVAL) {
    StageConvertAndReveal(engine);  // Frame N+1
  }
  else if (GatherReadyBatch(engine)) {
    StageMergeAndLinkHidden();      // Frame N
  }
  ReapTerminalLoads();
}

/* Never dereference target_scene before proving it is alive: the raw pointer is compared against
 * the engine's live list first (a handful of entries), then the generation rules out ABA. */
static KX_Scene *FindLiveScene(KX_KetsjiEngine *engine, KX_Scene *scene, uint64_t generation)
{
  for (KX_Scene *live : engine->CurrentScenes()) {
    if (live == scene) {
      return (live->GetGeneration() == generation) ? live : nullptr;
    }
  }
  return nullptr;
}

bool KX_BlendFileLoader::GatherReadyBatch(KX_KetsjiEngine *engine)
{
  BLI_assert(m_batch.is_empty());
  int budget = m_maxObjectsPerBatch;
  for (std::shared_ptr<PendingBlendLoad> &load : m_pending) {
    if (load->state.load(std::memory_order_acquire) != BlendLoadState::READY) {
      continue;
    }
    KX_Scene *scene = FindLiveScene(engine, load->target_scene, load->scene_generation);
    if (scene == nullptr) {
      ReleaseStagingAsync(*load);
      load->error_msg = "Target scene removed during load";
      load->state.store(BlendLoadState::ERROR, std::memory_order_relaxed);
      continue;
    }
    if (m_batch.is_empty()) {
      m_batchScene = scene;  // FIFO: the oldest READY load decides the batch's scene.
      m_batchGeneration = load->scene_generation;
    }
    else if (scene != m_batchScene || load->num_objects > budget) {
      continue;  // Next batch. The first load is always admitted, so nothing starves.
    }
    budget -= load->num_objects;
    m_batch.append(load);
  }
  return !m_batch.is_empty();
}

void KX_BlendFileLoader::ReapTerminalLoads()
{
  /* Terminal loads leave m_pending first; callbacks run afterwards on a local vector, so a
   * callback calling loadBlendAsync(), cancel() or free() never mutates a vector being iterated. */
  Vector<std::shared_ptr<PendingBlendLoad>> finished;
  m_pending.remove_if([&](std::shared_ptr<PendingBlendLoad> &load) {
    const BlendLoadState s = load->state.load(std::memory_order_acquire);
    const bool terminal = ELEM(
        s, BlendLoadState::DONE, BlendLoadState::ERROR, BlendLoadState::CANCELLED);
    if (terminal) {
      finished.append(load);
    }
    return terminal;
  });

  for (std::shared_ptr<PendingBlendLoad> &load : finished) {
    if (load->state.load(std::memory_order_relaxed) == BlendLoadState::DONE) {
      m_loaded.append(load);  // Kept for handle.free() and FreeLoadedIds().
    }
    if (PyObject *callback = std::exchange(load->on_finish, nullptr)) {
      /* Called in every terminal state, so a script waiting on it never waits forever. */
      PyObject *py_handle = (new KX_PyBlendLoadHandle(load))->NewProxy(true);
      PyObject *result = PyObject_CallFunctionObjArgs(callback, py_handle, nullptr);
      if (result == nullptr) {
        PyErr_Print();
        PyErr_Clear();
      }
      Py_XDECREF(result);
      Py_DECREF(py_handle);
      Py_DECREF(callback);
    }
  }
}
```

Stage 1 (`StageMergeAndLinkHidden`) and stage 2 (`StageConvertAndReveal`) are described in
[Two-Stage Integration Pipeline](#two-stage-integration-pipeline). `Cancel()`, `Free()` and
`Unload()` are described in [Unloading](#unloading).

---

## Two-Stage Integration Pipeline

### Why Two Stages?

The integration cost is dominated by work that is proportional to the **whole scene**, not to
the loaded content:

| Work | Proportional to | Where it happens |
|---|---|---|
| `BKE_main_merge` destination name map | all IDs in G_MAIN (one `std::string` + `Vector` per ID) | stage 1 |
| Depsgraph relations rebuild | whole scene graph (the builder is not incremental) | render pass of frame N |
| Evaluation of the new objects | loaded content (modifiers, geometry nodes) | render pass of frame N |
| Conversion | scene bases (SETLOOPER) + loaded objects | stage 2 |
| First draw (shaders, VBOs, textures) | loaded content | render pass of frame N+1 |

None of these can be sliced per object without repeating the scene-proportional parts. v3.2's
per-object batches (Phase 2) did not reduce the spike: all objects were linked in one step, so
the first evaluation already processed every new ID node, and each batch repeated the
evaluation call. v3.3 splits by **phase** instead:

- **Frame N:** merge, instantiate hidden, tag relations. The render pass that UPBGE already runs
  (`KX_Scene.cpp:727`) performs the rebuild and the evaluation — no extra evaluation call.
- **Frame N+1:** convert the evaluated data, reveal.

Total work is identical to a single-frame integration; the peak is split across two frames.
The added latency is one frame on a load that already took hundreds of milliseconds on the
loader thread.

### Stage 1 — Merge and Link Hidden (frame N)

```cpp
void KX_BlendFileLoader::StageMergeAndLinkHidden()
{
  Scene *bl_scene = m_batchScene->GetBlenderScene();
  ViewLayer *view_layer = BKE_view_layer_default_view(bl_scene);
  Vector<ID *> new_ntrees;

  for (std::shared_ptr<PendingBlendLoad> &load : m_batch) {
    for (ID *id : load->force_merge_ids) {
      if (id->id_type() == ID_NT) {
        new_ntrees.append(id);
      }
    }
    /* Forced ids are always moved, never freed with the staging Main, so every pointer held by
     * the load stays valid; colliding local names are renamed by BKE_libblock_management_main_add()
     * (".001"), which is Blender's own append semantics. */
    MainMergeReport report;
    BKE_main_merge(G_MAIN, &load->force_merge_ids, &load->staging_main, report);
    BLI_assert(load->staging_main == nullptr);
    load->force_merge_ids.clear_and_shrink();

    /* One container per load: a single LayerCollection to hide and reveal, and a single
     * collection to unlink on unload. Only requested roots go in: child collections arrive
     * through their parents, and instance collections are reached through their instancers. */
    load->container = BKE_collection_add(G_MAIN, bl_scene->master_collection, "AsyncLoad");
    load->owned_uids.add(load->container->id.session_uid);
    for (Collection *root : load->root_collections) {
      BKE_collection_child_add_no_sync(G_MAIN, load->container, root);
    }
    for (Object *ob : load->root_objects) {
      /* An object requested by name that also lives in a requested collection is linked once.
       * Requested lists are short (user-provided names), so this is not a hot loop. */
      bool in_root_collection = false;
      for (Collection *root : load->root_collections) {
        in_root_collection |= BKE_collection_has_object_recursive(root, ob);
      }
      if (!in_root_collection) {
        BKE_collection_object_add(G_MAIN, load->container, ob);
      }
    }
    load->state.store(BlendLoadState::INTEGRATING, std::memory_order_relaxed);
    load->progress.store(0.75f, std::memory_order_relaxed);
  }

  /* Non-handler part of BKE_blendfile_link_append_context_finalize(), skipped on the loader
   * thread because finalize() runs Python handlers. Scoped to the loaded node trees only. */
  for (ID *id : new_ntrees) {
    BKE_ntree_update_tag_all(reinterpret_cast<bNodeTree *>(id));
  }
  if (!new_ntrees.is_empty()) {
    BKE_main_ensure_invariants(*G_MAIN, new_ntrees.as_span());
  }

  /* Lazy: BKE_main_collection_sync() only tags view layers out of sync (layer.cc). */
  BKE_main_collection_sync(G_MAIN);
  BKE_view_layer_synced_ensure(*G_MAIN, bl_scene, view_layer);

  /* LAYER_COLLECTION_HIDE keeps BASE_ENABLED_VIEWPORT, so the depsgraph pulls the objects in and
   * evaluates them (deg_builder.cc:70-73), but drops BASE_ENABLED_AND_VISIBLE_IN_DEFAULT_VIEWPORT,
   * so they are not drawn (layer.cc:1127-1130). Render N evaluates; nothing pops in early. */
  for (std::shared_ptr<PendingBlendLoad> &load : m_batch) {
    LayerCollection *lc = BKE_layer_collection_first_from_scene_collection(view_layer,
                                                                           load->container);
    lc->flag |= LAYER_COLLECTION_HIDE;
  }
  BKE_view_layer_need_resync_tag(view_layer);

  /* New ID nodes: one relations rebuild for the whole batch, performed by the existing
   * BKE_scene_graph_update_tagged() call of the render pass. */
  DEG_relations_tag_update(G_MAIN);
  m_stage = Stage::AWAIT_EVAL;
}
```

### Stage 2 — Convert and Reveal (frame N+1)

```cpp
void KX_BlendFileLoader::StageConvertAndReveal(KX_KetsjiEngine *engine)
{
  m_stage = Stage::IDLE;
  Vector<std::shared_ptr<PendingBlendLoad>> batch = std::move(m_batch);
  KX_Scene *scene = FindLiveScene(engine, m_batchScene, m_batchGeneration);

  if (scene == nullptr) {
    for (std::shared_ptr<PendingBlendLoad> &load : batch) {
      if (load->state.load(std::memory_order_relaxed) == BlendLoadState::INTEGRATING) {
        Unload(*load);  // Nothing converted yet: owned ids + container only.
        load->error_msg = "Target scene removed during integration";
        load->state.store(BlendLoadState::ERROR, std::memory_order_relaxed);
      }
    }
    return;
  }

  /* Flat set of every object to convert. Loads cancelled between the two stages were already
   * unloaded by Cancel() and are skipped. */
  Set<const Object *> objects;
  for (std::shared_ptr<PendingBlendLoad> &load : batch) {
    if (load->state.load(std::memory_order_relaxed) != BlendLoadState::INTEGRATING) {
      continue;
    }
    FOREACH_COLLECTION_OBJECT_RECURSIVE_BEGIN (load->container, ob) {
      objects.add(ob);
    }
    FOREACH_COLLECTION_OBJECT_RECURSIVE_END;
  }
  if (objects.is_empty()) {
    return;
  }

  /* Conversion reads the *context* depsgraph (BL_DataConversion.cpp:370). The context scene is
   * whichever scene rendered last (KX_Scene::ReinitBlenderContextVariables() runs per render
   * pass), so it must point at the target scene first; the next render pass resets it. */
  scene->ReinitBlenderContextVariables();
  bContext *C = engine->GetContext();
  /* O(1) when render N already evaluated the graph (DEG_is_fully_evaluated() early-out,
   * scene.cc:2801). Evaluates here only if the target scene did not render last frame. */
  BKE_scene_graph_evaluated_ensure(CTX_data_depsgraph_pointer(C), G_MAIN);

  scene->ConvertBlenderObjectsBatch(objects);  // One SETLOOPER pass, see Batch Conversion.

  Scene *bl_scene = scene->GetBlenderScene();
  ViewLayer *view_layer = BKE_view_layer_default_view(bl_scene);
  for (std::shared_ptr<PendingBlendLoad> &load : batch) {
    if (load->state.load(std::memory_order_relaxed) != BlendLoadState::INTEGRATING) {
      continue;
    }
    /* Re-lookup: a resync may reallocate LayerCollections, so none is cached across frames. */
    LayerCollection *lc = BKE_layer_collection_first_from_scene_collection(view_layer,
                                                                           load->container);
    lc->flag &= ~LAYER_COLLECTION_HIDE;
    load->state.store(BlendLoadState::DONE, std::memory_order_relaxed);
    load->progress.store(1.0f, std::memory_order_relaxed);
  }
  BKE_view_layer_need_resync_tag(view_layer);
  /* Topology unchanged (BASE_ENABLED_VIEWPORT was already set): base flags only, no rebuild. */
  DEG_id_tag_update_ex(G_MAIN, &bl_scene->id, ID_RECALC_BASE_FLAGS);
}
```

### Why the Hidden Link Is Correct

Verified in source:

- **Evaluated while hidden.** The depsgraph builder pulls every base with `BASE_ENABLED_VIEWPORT`
  into the graph (`deg_builder.cc:70-73`). `LAYER_COLLECTION_HIDE` still sets
  `BASE_ENABLED_VIEWPORT | BASE_ENABLED_AND_MAYBE_VISIBLE_IN_VIEWPORT` and only withholds
  `BASE_ENABLED_AND_VISIBLE_IN_DEFAULT_VIEWPORT` (`layer.cc:1127-1130`).
- **Not drawn while hidden.** `BKE_object_is_visible_in_viewport()` tests
  `BASE_ENABLED_AND_VISIBLE_IN_DEFAULT_VIEWPORT` (`layer.cc:1726`). To be confirmed visually in
  the game render path (see checklist).
- **Converted as active.** UPBGE's converter treats
  `BASE_ENABLED_AND_MAYBE_VISIBLE_IN_VIEWPORT` as "in active layer" (`BL_DataConversion.cpp:1316`),
  so it does not force `OB_HIDE_VIEWPORT`. `base_flag` on the original object is valid because it
  is synced back during the render-pass evaluation (`object_update.cc:311`).
- **KX visibility unaffected.** `KX_GameObject` visibility comes from `OB_HIDE_RENDER`
  (`BL_DataConversion.cpp:916`), not from layer collections.
- **Reveal is cheap.** Clearing the flag does not change graph topology; only base flags are
  re-evaluated (`ID_RECALC_BASE_FLAGS`).

### Why This Position in the Frame Loop?

```
NextFrame() {
    times = GetFrameTimes();
    if (times.frames == 0) return false;

    IntegratePendingLoads(this);          ← v3.3: ONCE per NextFrame(), outside the tick loop

    for (i = 0; i < times.frames; ++i) {  ← catch-up ticks when the engine is behind
        m_converter->MergeAsyncLoads();   ← existing legacy (inside the loop)
        for each scene: logic, physics, scenegraph
    }
}

Render() {
    per scene: BKE_scene_graph_update_tagged()  ← stage-1 work is evaluated here
}
```

| Criterion | Inside the tick loop (v3.2) | Before the tick loop (v3.3) |
|---|---|---|
| Integrations per rendered frame | up to `times.frames` — the spike lands exactly when the engine is already behind | exactly one stage |
| All scenes see new objects in the first tick | ✅ | ✅ |
| Previous frame's physics and logic completed | ✅ | ✅ |
| Depsgraph evaluated before conversion | ❌ needed a forced evaluation | ✅ evaluated by render N |

### API Functions Used (verified in UPBGE `e7c5aac`)

| Function | Header | Purpose |
|---|---|---|
| `BKE_main_merge` (overload with `force_merge_src_ids`) | `BKE_main.hh:545` | Merge staging → G_MAIN, local ids always moved |
| `BKE_main_init_from_reference` | `BKE_main.hh:482` | Staging inherits filepath and working colorspace |
| `BKE_collection_add` | `BKE_collection.hh:101` | Per-load container |
| `BKE_collection_child_add_no_sync` | `BKE_collection.hh:394` | Link root collections without a sync per call |
| `BKE_collection_object_add` | `BKE_collection.hh:237` | Link requested loose objects |
| `BKE_collection_has_object_recursive` | `BKE_collection.hh` | Avoid double-linking requested objects |
| `BKE_main_collection_sync` | `BKE_collection.hh` | Lazy view-layer resync tag |
| `BKE_layer_collection_first_from_scene_collection` | `BKE_layer.hh:233` | Container's LayerCollection |
| `BKE_view_layer_need_resync_tag` | `BKE_layer.hh:677` | Apply the hide/reveal flag |
| `BKE_ntree_update_tag_all` | `BKE_node_tree_update.hh:32` | Replaces finalize() node tree tagging |
| `BKE_main_ensure_invariants` | `BKE_main_invariants.hh` | Replaces finalize() propagation, scoped |
| `DEG_relations_tag_update` | `DEG_depsgraph_build.hh` | New ID nodes |
| `DEG_id_tag_update_ex` (`ID_RECALC_BASE_FLAGS`) | `DEG_depsgraph.hh` | Reveal without relations rebuild |
| `BKE_scene_graph_evaluated_ensure` | `BKE_scene.hh` | O(1) guard before conversion |
| `BKE_id_multi_delete` | `BKE_lib_id.hh:649` | Unload (Set-based, no global tags) |

---

## BLO Thread Safety Audit

### Methodology

Full source code review of all functions in the loader thread critical path, searching for
any access to `G_MAIN`, `G.main`, `BKE_main_blendfile_path_from_global()`, or other global
state. Files reviewed:

- `source/blender/blenloader/intern/readfile.cc` (6216 lines)
- `source/blender/blenloader/intern/readblenentry.cc` (465 lines)
- `source/blender/blenkernel/intern/blendfile_link_append.cc` (2318 lines)
- `source/blender/blenkernel/intern/blendfile.cc` (2335 lines)
- `source/blender/blenloader/intern/blend_validate.cc`

**v3.3 addendum:** the v3.2 review covered these five files with a grep for explicit names. It
could not see implicit global access in the BKE code they call (see below), and it did not cover
the `blend_read_*` callbacks of every ID type or the versioning files (`versioning_*.cc`).

### Result: Not Proven Thread-Safe — TSAN Is a Blocker ⚠️

The explicit grep finds **one** global access in the BLO chain, conditional on a flag we do not
set. But reachable BKE code accesses `G.main` **implicitly**, which a grep for `G_MAIN|G.main`
cannot find. TSAN on the corpus defined in the checklist is required before merging.

### Global Access Inventory

#### Explicit (grep `G_MAIN|G\.main|BKE_main_blendfile_path_from_global` in BLO files)

```
readfile.cc:5681         ← only code hit, inside library_link_end(), gated by FILE_RELPATH
blend_validate.cc:210    ← comment only, not code   (line as audited for v3.2)
```

```cpp
// readfile.cc:5676-5682 (e7c5aac), inside library_link_end() (readfile.cc:5655):
if (flag & FILE_RELPATH) {                                        // ← CONDITIONAL
    STRNCPY(curlib->filepath, curlib->runtime->filepath_abs);
    BLI_path_rel(curlib->filepath, BKE_main_blendfile_path_from_global());  // ← GLOBAL
}
```

**This code only executes if `FILE_RELPATH` is set in `params->flag`.** Since we configure
`params.flag = BLO_LIBLINK_APPEND_RECURSIVE` (without `FILE_RELPATH`), this branch is
**never taken**. Relative paths are unnecessary for the game engine — we work with absolute
paths resolved on the main thread in `RequestLoad()`.

#### Implicit (missed by v3.2)

```cpp
// depsgraph_tag.cc:844-846
void DEG_id_tag_update(ID *id, uint flags)
{
  DEG_id_tag_update_ex(G.main, id, flags);   // ← G.main, whatever Main the id lives in
}
```

`DEG_id_tag_update_ex(G.main, ...)` walks every depsgraph registered for G.main. The registry
lookup is locked, but it returns a `Span` that escapes the lock (`depsgraph_registry.cc`,
`get_all_registered_graphs()`), and each graph's ID hash is then read without a lock while the
main thread may be rebuilding it. One reachable path from `BKE_blendfile_append()`: when an
object is appended as a local *copy* (`LINK_APPEND_ACT_COPY_LOCAL`),
`BKE_libblock_relink_to_newid()` remaps the old/new pair and runs the object post-process, which
tags metaball basis objects with `DEG_id_tag_update()` (`lib_remap.cc:769` → `:382/391`). Any
IDTypeInfo callback or versioning function calling `DEG_id_tag_update()` (non-`_ex`) is another.

Extended grep for the next audit pass, over `blenkernel/intern/*.cc` (read callbacks, remap,
make-local) and `blenloader/intern/versioning_*.cc`:

```
G\.main|G_MAIN|DEG_id_tag_update\(|DEG_relations_tag_update\(|_from_global|CTX_
```

### Function-by-Function Verification

Line numbers in the Layer 1 and Layer 2 tables are those of the revision audited for v3.2;
re-anchor them against the implementation revision during the next audit pass.

#### Layer 1: BLO (Blend Loader)

| Function | Source | Global Access | Notes |
|---|---|---|---|
| `BLO_blendhandle_from_file` | readblenentry.cc:65 | ❌ None | Calls `blo_filedata_from_file()` — pure file I/O. v3.3: called by the link/append context, not by the loader directly |
| `BLO_library_link_begin` | readfile.cc:5303 | ❌ None | Calls `library_link_begin(params->bmain, fd, filepath)` |
| ↳ `library_link_begin` | readfile.cc:5228 | ❌ None | `fd->bmain = mainvar` (staging), iterates `mainvar` only |
| ↳ `blo_find_main_for_library_and_idname` | readfile.cc | ❌ None | Uses `BKE_main_blendfile_path(mainvar)` — reads `staging_main->filepath` (set by RequestLoad) |
| `BLO_library_link_named_part` | readfile.cc:5207 | ❌ None | Calls `link_named_part(mainl, fd, ...)` — all local |
| ↳ `link_named_part` | readfile.cc:5156 | ❌ None | `mainl` + `fd` only, reads bheads from file |
| ↳ `read_libblock` | readfile.cc | ❌ None | Allocates into `mainl` (staging split) |
| `BLO_library_link_end` | readfile.cc:5463 | ❌ None** | Calls `library_link_end(mainl, &fd, params->flag, reports)` |
| ↳ `library_link_end` | readfile.cc:5335 | ⚠️ **Only if FILE_RELPATH** | See readfile.cc:5681 (e7c5aac) above — **we never set this flag** |
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
| ↳ `libblock_remap_data_postprocess_object_update` | lib_remap.cc:364 (e7c5aac) | ⚠️ **Implicit G.main** | Reached through `BKE_libblock_relink_to_newid()` for copied-local objects; `DEG_id_tag_update()` for metaball basis objects (lib_remap.cc:382/391) |
| `BKE_blendfile_link_append_context_library_add` | BKE_blendfile_link_append.hh:240 (e7c5aac) | ❌ None | Returns `void`; **always steals** the given handle — pass `nullptr` |
| `BKE_blendfile_link_append_context_item_newid_get` | blendfile_link_append.cc:300 (e7c5aac) | ❌ None | Reads `item->new_id`; resolves requested roots by pointer |
| `BKE_blendfile_link_append_context_free` | blendfile_link_append.cc:144 | ❌ None | Frees context-local data; releases the handle it owns |
| `BKE_main_free` (error/cancel paths) | main.cc (e7c5aac) | ❌ None | `BKE_main_clear()` frees with `LIB_ID_FREE_NO_DEG_TAG` |

#### Layer 3: Infrastructure

| Function | Thread-Safe | Notes |
|---|---|---|
| `MEM_guardedalloc` (MEM_mallocN, MEM_freeN, etc.) | ✅ Yes | Thread-safe allocator with internal mutex |
| `CLG_log` (CLOG_DEBUG, CLOG_ERROR, etc.) | ✅ Yes | Thread-safe logging system |
| `BKE_reports_init / BKE_reports_free` | ✅ Yes | Per-instance (own mutex). The second argument of `BKE_reports_init` is a `ReportListFlags` mask: pass `RPT_STORE` |
| `BLI_task_pool_create_background_serial` | ✅ Yes | Dedicated pthread + `BLI_thread_queue` (mutex per push/pop) |

### Critical Safety Requirements (summary)

Six conditions MUST be met:

1. **`params->flag` must NOT include `FILE_RELPATH`** — prevents the only explicit global access
   (readfile.cc:5681, e7c5aac).
2. **`params->context.scene` must be `nullptr`** — prevents `loose_data_instantiate()`, which
   accesses Scene/ViewLayer (blendfile_link_append.cc:977, e7c5aac). Instantiation happens in
   stage 1.
3. **DO NOT call `context_init_done()` or `context_finalize()` on the loader thread** — both
   execute Python handlers through `BKE_callback_exec` (blendfile_link_append.cc:372 and 403,
   e7c5aac). The non-handler work of `finalize()` (node tree tagging,
   `BKE_main_ensure_invariants()`) is redone on the main thread in stage 1.
4. **Pass `nullptr` to `library_add()`** — the context opens the file and owns the handle.
5. **Never reach `DEG_id_tag_update()` (non-`_ex`) or any other implicit `G.main` access from
   the loader thread** — verified by TSAN on the checklist corpus, not by grep alone.
6. **Initialize the staging Main from G_MAIN** (next section).

Conditions 1-4 and 6 are enforced in `LoadThreadFunc` / `RequestLoad()`; condition 5 is the
TSAN blocker.

### `staging_main` Initialization Requirement

`RequestLoad()` initializes the staging Main from G_MAIN on the main thread:

```cpp
load->staging_main = BKE_main_new();
BKE_main_init_from_reference(*load->staging_main, *G_MAIN);  // filepath + working colorspace
```

- **Filepath.** Making appended IDs local rebases their relative paths (images, sounds,
  libraries) against `BKE_main_blendfile_path(staging)` (lib_id.cc:181,
  `lib_id_library_local_paths()`), so they end up relative to the game file. `BKE_main_merge()`
  then rebases from dir(staging) to dir(G_MAIN), a no-op. `library_link_begin` also resolves
  library paths through `BKE_main_blendfile_path(mainvar)`. v3.2 copied the filepath only.
- **Colorspace.** `BKE_main_init_from_reference()` is documented "for relative path and
  colorspace conversions" (BKE_main.hh:479-482), so colorspace conversions during append target
  the working space of the game file rather than the default one set by `BKE_main_new()`.

This is safe because `RequestLoad()` runs on the main thread, where G_MAIN may be read.

---

## Threading Model

### Loader Thread

```
┌────────────────────────────────────────────────────────────────────┐
│ BLI_task_pool_create_background_serial() — one per loader          │
├────────────────────────────────────────────────────────────────────┤
│ Created:   KX_BlendFileLoader constructor                          │
│ Push:      RequestLoad() — one task per load (FIFO)                │
│ Runs on:   ONE dedicated pthread (BLI_threadpool), not a TBB task  │
│ Freed:     Shutdown() — ~TaskPool joins the thread                 │
│            (background_task_pool_work_and_wait + threadpool_end)   │
└────────────────────────────────────────────────────────────────────┘
```

Why not `BLI_task_pool_create()` (v3.2), verified in `task_pool.cc`:

1. **Inline execution.** With one scheduler thread (`-t 1`, single-core machines)
   `use_threads == false` and `BLI_task_pool_push()` executes the task immediately
   ("Execute immediately") — the whole load would run inside `RequestLoad()`.
2. **Work stealing.** A TBB wait "can actually do work" (Blender's own comment in
   `tbb_task_pool_work_and_wait`). Any wait or `parallel_for` on the main thread (depsgraph
   evaluation, draw manager extraction) can pick up a queued multi-hundred-millisecond load task:
   the freeze this design exists to remove, intermittent and hard to reproduce.
3. **No join on free.** `~TaskPool` does nothing for `TASK_POOL_TBB`; destroying a
   `tbb::task_group` with pending work makes oneTBB throw `missing_wait` from a destructor
   (`std::terminate`). There is a window between `state.store(READY)` and the task returning,
   and v3.2's `CancelAll()` during an active load hits it deterministically.

Why not `BLI_task_pool_create_background()`: its constructor turns `TASK_POOL_BACKGROUND` into
`TASK_POOL_TBB` whenever threads are available. Only the `_serial` variant always uses a
dedicated thread.

Why serial: loads are I/O-bound and share one disk. Concurrent readers on the same device
interleave seeks and lose throughput; FIFO order also gives the predictable completion order
that batching relies on. Cost: one sleeping thread.

### Synchronization Model

```
┌────────────────────────────────────────────────────────────────────┐
│ std::atomic<BlendLoadState> — the only shared mutable word         │
├────────────────────────────────────────────────────────────────────┤
│ Loader:  CAS LOADING → READY (acq_rel) publishes all output fields │
│          release store ERROR / CANCELLED (terminal)                │
│ Main:    acquire load before reading any loader output             │
│          CAS LOADING → CANCEL_REQUESTED (acq_rel)                  │
│          relaxed stores for every transition after READY           │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ No mutex in the loader                                             │
├────────────────────────────────────────────────────────────────────┤
│ RequestLoad() (called from Python) and IntegratePendingLoads() run │
│ on the main thread; both assert BLI_thread_is_main().              │
│ Re-entrancy (a callback requesting a load) is handled by running   │
│ callbacks after all iteration, on a local vector.                  │
│ The only lock on the path is inside BLI_thread_queue (one push     │
│ per load, one pop per load), never per frame.                      │
└────────────────────────────────────────────────────────────────────┘
```

### Ownership by Phase

| State | Owner | May touch | shared_ptr holders |
|---|---|---|---|
| `LOADING` | Loader thread | `staging_main`, output fields, `progress` | Loader, task, Python handle(s) |
| `CANCEL_REQUESTED` | Loader thread | same | same |
| `READY` | Main thread | everything (loader thread is done) | Loader, Python handle(s); task ref may still be dropping |
| `INTEGRATING` | Main thread | `container`, `owned_uids` | Loader (`m_pending`, `m_batch`), handle(s) |
| `DONE` | Main thread | `container`, `owned_uids` (unload) | Loader (`m_loaded`), handle(s) |
| terminal | nobody | read-only queries | Python handle(s) |

### Lifetime Guarantees

- The queued task owns a `shared_ptr` copy (`new shared_ptr` as taskdata, deleted by
  `LoadTaskFree()` after the task runs), so the struct outlives the loader thread's last access
  independently of what the main thread does.
- If the task's reference is the last one, `~PendingBlendLoad` runs on the loader thread. It
  only frees plain memory and asserts that `on_finish` and `staging_main` are already released.
- `PyObject` references (`on_finish`) are released on the main thread only: in
  `ReapTerminalLoads()` (game loop, GIL held as for any Python controller) or in `Shutdown()`,
  which runs from `StopEngine()` before `ExitPython()` (`LA_Launcher.cpp:323` vs `:389`) and wraps
  its releases in `PyGILState_Ensure()/Release()` so it does not depend on the launcher mode
  (embedded mode already holds the GIL across `ExitEngine()`, `BL_KetsjiEmbedStart.cpp:208/444`).
- Python handles stay valid forever; after the engine stops they report the last state.

---

## Integration Points

### In `KX_KetsjiEngine::NextFrame()` — Once per Frame, Outside the Tick Loop

```cpp
bool KX_KetsjiEngine::NextFrame()
{
  m_logger.StartLog(tc_services);
  const FrameTimes times = GetFrameTimes();
  if (times.frames == 0) {
    m_logger.StartLog(tc_outside);
    return false;
  }

  /* [ASYNC_BLEND_LOAD] One integration stage per rendered frame. Inside the tick loop it would
   * run once per catch-up tick, i.e. several times in exactly the frames where the engine is
   * already behind. Logged separately so stage costs show up in the profiler. */
  m_logger.StartLog(tc_asyncload);
  m_blendFileLoader->IntegratePendingLoads(this);
  m_logger.StartLog(tc_services);

  for (unsigned short i = 0; i < times.frames; ++i) {
    m_frameTime += times.framestep;
    m_converter->MergeAsyncLoads();  // existing legacy, untouched
    // ... rest of the tick loop (logic, physics, scenegraph) ...
  }
}
```

### In `KX_KetsjiEngine`

```cpp
// Constructor:
m_blendFileLoader = std::make_unique<KX_BlendFileLoader>();

void KX_KetsjiEngine::StopEngine()
{
  if (m_bInitialized) {
    m_converter->FinalizeAsyncLoads();
    m_blendFileLoader->Shutdown();  // Before scenes go: join loader thread, drop PyObject refs.

    while (m_scenes->GetCount() > 0) {
      // ... existing scene removal ...
    }
    /* After scenes go: KX objects referencing the loaded Blender data no longer exist. */
    if (m_freeAsyncLoadsOnExit) {
      m_blendFileLoader->FreeLoadedIds();
    }
    m_rasterizer->Exit();
  }
}
```

`m_freeAsyncLoadsOnExit` is set by `LA_BlenderLauncher` to `!(gm.flag & GAME_USE_UNDO)`. Embedded
mode restores Main through memfile undo only when `GAME_USE_UNDO` is set
(`BL_KetsjiEmbedStart.cpp:217/422`); without it, merged IDs would remain in the user's editing
session and could be saved into their `.blend`. The blenderplayer frees Main at exit and leaves
the flag off, so exit does not pay for a deletion nobody observes.

### In `RequestLoad()` — Path Resolution + staging_main

```cpp
std::shared_ptr<PendingBlendLoad> KX_BlendFileLoader::RequestLoad(
    const std::string &filepath,
    std::vector<std::string> collections,
    std::vector<std::string> objects,
    PyObject *on_finish,
    KX_Scene *target_scene)
{
  BLI_assert(BLI_thread_is_main() && m_pool != nullptr);
  auto load = std::make_shared<PendingBlendLoad>();
  load->filepath = filepath;
  load->collection_names = std::move(collections);
  load->object_names = std::move(objects);
  load->target_scene = target_scene;
  load->scene_generation = target_scene->GetGeneration();
  Py_XINCREF(on_finish);
  load->on_finish = on_finish;
  m_pending.append(load);  // Every outcome, early errors included, is reaped and calls back.

  if (load->collection_names.empty() && load->object_names.empty()) {
    load->error_msg = "Nothing requested: pass at least one collection or object name";
    load->state.store(BlendLoadState::ERROR, std::memory_order_relaxed);
    return load;
  }

  /* '//' is relative to the game file: resolved here, where G_MAIN may be read. */
  char resolved[FILE_MAX];
  STRNCPY(resolved, filepath.c_str());
  BLI_path_abs(resolved, BKE_main_blendfile_path(G_MAIN));
  load->filepath_resolved = resolved;
  if (!BLI_exists(resolved)) {
    load->error_msg = "File not found: " + load->filepath_resolved;
    load->state.store(BlendLoadState::ERROR, std::memory_order_relaxed);
    return load;
  }

  /* Staging inherits filepath AND working colorspace from G_MAIN (BKE_main.hh:479-482).
   * Make-local rebases relative paths (images, sounds, libraries) against
   * BKE_main_blendfile_path(staging) (lib_id.cc:181, lib_id_library_local_paths()), so they end
   * up relative to the game file; BKE_main_merge() then rebases dir(staging) -> dir(G_MAIN),
   * which is a no-op. */
  load->staging_main = BKE_main_new();
  BKE_main_init_from_reference(*load->staging_main, *G_MAIN);

  /* The queue's internal mutex publishes every field written above to the loader thread. */
  BLI_task_pool_push(
      m_pool, LoadThreadFunc, new std::shared_ptr<PendingBlendLoad>(load), true, LoadTaskFree);
  return load;
}
```

### `Shutdown()` and `FreeLoadedIds()` — Engine Stop

```cpp
void KX_BlendFileLoader::Shutdown()
{
  BLI_assert(BLI_thread_is_main());
  /* 1. Queued and running loads stop at their next check (entry check or final CAS). */
  for (std::shared_ptr<PendingBlendLoad> &load : m_pending) {
    BlendLoadState expected = BlendLoadState::LOADING;
    load->state.compare_exchange_strong(
        expected, BlendLoadState::CANCEL_REQUESTED, std::memory_order_acq_rel);
  }
  /* 2. Join. Queued tasks exit at their entry check; a running link/append finishes its current
   *    file first (BLO has no cancellation point). Queued FreeMainTask jobs run as well. */
  BLI_task_pool_free(m_pool);
  m_pool = nullptr;

  /* 3. READY loads that never reached stage 1 still own their staging Main. */
  for (std::shared_ptr<PendingBlendLoad> &load : m_pending) {
    if (load->staging_main) {
      BKE_main_free(load->staging_main);
      load->staging_main = nullptr;
    }
    load->force_merge_ids.clear_and_shrink();
  }

  /* 4. Python references, under the GIL whatever the launcher mode. */
  PyGILState_STATE gil = PyGILState_Ensure();
  for (Vector<std::shared_ptr<PendingBlendLoad>> *list : {&m_pending, &m_loaded}) {
    for (std::shared_ptr<PendingBlendLoad> &load : *list) {
      Py_CLEAR(load->on_finish);
    }
  }
  PyGILState_Release(gil);
  /* INTEGRATING and DONE loads stay listed: FreeLoadedIds() may still need their ownership. */
}

void KX_BlendFileLoader::FreeLoadedIds()
{
  /* KX scenes are already destroyed: only Blender data is deleted. One session_uid pass over
   * G_MAIN for all loads together, one multi-delete. */
  Set<uint32_t> uids;
  for (Vector<std::shared_ptr<PendingBlendLoad>> *list : {&m_pending, &m_loaded}) {
    for (std::shared_ptr<PendingBlendLoad> &load : *list) {
      const BlendLoadState s = load->state.load(std::memory_order_relaxed);
      if (ELEM(s, BlendLoadState::INTEGRATING, BlendLoadState::DONE)) {
        for (const uint32_t uid : load->owned_uids) {
          uids.add(uid);
        }
      }
    }
  }
  Set<ID *> alive;
  ID *id;
  FOREACH_MAIN_ID_BEGIN (G_MAIN, id) {
    if (uids.contains(id->session_uid)) {
      alive.add_new(id);
    }
  }
  FOREACH_MAIN_ID_END;
  BKE_id_multi_delete(G_MAIN, alive);
  m_pending.clear();
  m_loaded.clear();
  m_batch.clear();
}
```

**Cancellation latency at shutdown:** bounded by the file being read at that moment. Queued
loads cost nothing; they exit at their entry check.

---

## Batch Conversion

### Problem (existing converter, verified)

- `BL_ConvertBlenderObjects()` with a single runtime object iterates **every base of the scene**
  (`SETLOOPER`, `BL_DataConversion.cpp:1296`) and filters with
  `blenderobject != runtime_converted_object` at ~15 sites.
- Each base does `converter->FindGameObject()`, a `std::map::operator[]`
  (`BL_SceneConverter.cpp:94-97`): a pointer-chasing red-black tree lookup that also **inserts**
  a null entry for every miss.
- `ConvertBlenderCollection()` and `ConvertBlenderObjectsList()` call it once per object.

Converting K objects into a scene of N bases therefore costs O(K · N · log N) node visits. For
N = 5000 and K = 500 that is ~2.5 M map lookups, against ~5.5 k for a single pass.

- For every object that is not in an active layer, the loop also runs
  `BKE_main_collection_sync_remap()` + `DEG_relations_tag_update()`
  (`BL_DataConversion.cpp:1330-1333`): a whole-Main cache rebuild **per hidden object**. This
  affects the initial scene conversion as well.

### Change

```diff
 void BL_ConvertBlenderObjects(blender::Main *maggie,
                               ...
-                              blender::Object *runtime_converted_object,
+                              const blender::Set<const blender::Object *> *runtime_batch,
                               bool alwaysUseExpandFraming,
                               bool libloading)
```

Mechanical replacement at every filter site:

```diff
-    if (runtime_converted_object && !converting_instance_col_at_runtime) {
-      if (blenderobject != runtime_converted_object) {
+    if (runtime_batch && !converting_instance_col_at_runtime) {
+      if (!runtime_batch->contains(blenderobject)) {
         continue;
       }
     }
```

- `converting_during_runtime = runtime_batch != nullptr`.
- `converting_instance_col_at_runtime` becomes "any object of the batch has an
  `instance_collection` and is not converted yet" — for a one-element batch this is exactly
  today's condition.
- The two places that use the single runtime object directly (its actuators, and the
  post-conversion `FindGameObject(runtime_converted_object)` block) loop over the batch.

Hoisted cache rebuild:

```diff
     if (hide) {
       blenderobject->visibility_flag |= OB_HIDE_VIEWPORT;
-      BKE_main_collection_sync_remap(maggie);
-      DEG_relations_tag_update(maggie);
+      need_collection_remap = true;  // Once per call, not once per hidden object.
     }
 ...
+  if (need_collection_remap) {
+    BKE_main_collection_sync_remap(maggie);
+    DEG_relations_tag_update(maggie);
+  }
```

Lookup without insertion:

```diff
 KX_GameObject *BL_SceneConverter::FindGameObject(blender::Object *for_blenderobject)
 {
-  return m_map_blender_to_gameobject[for_blenderobject];
+  const auto it = m_map_blender_to_gameobject.find(for_blenderobject);
+  return (it != m_map_blender_to_gameobject.end()) ? it->second : nullptr;
 }
```

`KX_Scene` entry points:

```cpp
void KX_Scene::ConvertBlenderObjectsBatch(const blender::Set<const blender::Object *> &batch)
{
  KX_KetsjiEngine *engine = KX_GetActiveEngine();
  blender::bContext *C = engine->GetContext();
  BL_ConvertBlenderObjects(CTX_data_main(C),
                           CTX_data_expect_evaluated_depsgraph(C),
                           this,
                           engine,
                           UseBullet,
                           engine->GetRasterizer(),
                           engine->GetCanvas(),
                           m_sceneConverter,
                           &batch,
                           false,
                           false);
}

void KX_Scene::ConvertBlenderObject(blender::Object *ob)
{
  ConvertBlenderObjectsBatch(blender::Set<const blender::Object *>{ob});  // Behavior unchanged.
}
```

The synchronous paths of `ConvertBlenderObjectsList()` and `ConvertBlenderCollection()` (the
existing Python API) can route through the batch as well and get the same speed-up. Their
`asynchronous == true` paths are not used by the loader and are left untouched by this
proposal (see the note in the checklist about a defect found there).

**Cost after the change:** one `SETLOOPER` pass with an O(1) set lookup per base, plus the
per-object conversion work: O(N + K) instead of O(K · N · log N).

---

## Unloading

### Semantics

- `handle.free()` on a `DONE` load removes everything the load created:
  - its KX objects, including replicas spawned from them (replicas share the Blender object);
  - logic-manager mesh and action entries;
  - scene-slot materials, meshes and interpolators;
  - its container collection and every owned ID that still exists.
- `Cancel()` on an `INTEGRATING` load (and a target scene that dies between the two stages) uses
  the same path; there are no KX objects yet.
- `handle.free()` on a load that is not `DONE` yet is a cancel.

### Algorithm

```cpp
void KX_BlendFileLoader::Unload(PendingBlendLoad &load)
{
  BLI_assert(BLI_thread_is_main());
  /* Resolve owned ids by session_uid, never through stored pointers: game Python (bpy) may have
   * deleted some of them since the load. One linear pass over G_MAIN: unload is not a per-frame
   * operation, and this is what makes it immune to dangling pointers. */
  Set<ID *> alive;
  ID *id;
  FOREACH_MAIN_ID_BEGIN (G_MAIN, id) {
    if (load.owned_uids.contains(id->session_uid)) {
      alive.add_new(id);
    }
  }
  FOREACH_MAIN_ID_END;

  if (load.state.load(std::memory_order_relaxed) == BlendLoadState::DONE) {
    /* KX side first: KX objects reference the Blender data deleted below. Shared with
     * FreeBlendFile(), which passes IS_TAGGED as the predicate. */
    KX_GetActiveEngine()->GetConverter()->RemoveConvertedData(
        [&alive](const ID *owned) { return alive.contains(const_cast<ID *>(owned)); });
  }

  /* Deleting the container also unlinks it from master_collection. */
  BKE_id_multi_delete(G_MAIN, alive);
  DEG_relations_tag_update(G_MAIN);
  load.container = nullptr;
  load.owned_uids.clear_and_shrink();
}

void KX_BlendFileLoader::Cancel(PendingBlendLoad &load)
{
  BLI_assert(BLI_thread_is_main());
  BlendLoadState expected = BlendLoadState::LOADING;
  if (load.state.compare_exchange_strong(
          expected, BlendLoadState::CANCEL_REQUESTED, std::memory_order_acq_rel))
  {
    return;  // The loader thread finishes it as CANCELLED (before any I/O if still queued).
  }
  switch (expected) {
    case BlendLoadState::READY:  // Loader thread is done: staging_main belongs to the main thread.
      ReleaseStagingAsync(load);
      load.state.store(BlendLoadState::CANCELLED, std::memory_order_relaxed);
      break;
    case BlendLoadState::INTEGRATING:  // Merged and linked hidden, not converted.
      Unload(load);
      load.state.store(BlendLoadState::CANCELLED, std::memory_order_relaxed);
      break;
    default:  // CANCEL_REQUESTED, DONE (use free()), or already terminal.
      break;
  }
}

void KX_BlendFileLoader::Free(PendingBlendLoad &load)
{
  if (load.state.load(std::memory_order_relaxed) != BlendLoadState::DONE) {
    Cancel(load);
    return;
  }
  Unload(load);
  load.state.store(BlendLoadState::FREED, std::memory_order_relaxed);
  m_loaded.remove_if(
      [&load](const std::shared_ptr<PendingBlendLoad> &p) { return p.get() == &load; });
}
```

### Shared KX Cleanup (`BL_Converter`)

The body of `FreeBlendFile()` that removes KX-side data (`BL_Converter.cpp:604-752`) moves
unchanged into `RemoveConvertedData()`, parameterized by a predicate instead of `IS_TAGGED`:

```cpp
void BL_Converter::RemoveConvertedData(blender::FunctionRef<bool(const blender::ID *)> is_owned);

// FreeBlendFile() keeps its tagging and then calls:
RemoveConvertedData([](const blender::ID *id) { return IS_TAGGED(id); });
```

Inside the moved body, `IS_TAGGED(x)` becomes `(x && is_owned(&x->id))` (or the cast used today).
The scene-removal branch never triggers for async loads, because they never own a Scene ID. The
predicate call costs a few nanoseconds per check on an operation that runs once per unload.

### Cost

| Part | Proportional to |
|---|---|
| session_uid pass | all IDs in G_MAIN |
| KX cleanup (existing `FreeBlendFile` logic) | all KX objects in all scenes + their meshes/materials |
| `BKE_id_multi_delete` | deleted IDs + remapping of their users across Main |
| Next render | one relations rebuild |

The unload spike is of the same order as a load's stage 1. If profiling shows it matters for
streaming, the same two-stage split applies: hide the container and remove KX data in frame U,
delete IDs in frame U+1 (see Roadmap).

---

## Python API

### `bge.logic.loadBlendAsync()`

```python
handle = bge.logic.loadBlendAsync(
    filepath,            # str: path to .blend (accepts '//')
    collections=[...],   # list[str]: ID_GR names (roots; children come with them)
    objects=[...],       # list[str]: ID_OB names (roots)
    on_finish=callback,  # callable(handle) | None: called once, on the main thread,
                         # in every terminal state ('done', 'error', 'cancelled')
    scene=None           # KX_Scene | None: target scene, defaults to the current scene
)
```

The `evaluate` parameter of v3.2 is removed. Conversion always reads data evaluated by the
render pass (stage split), which is both cheaper than v3.2's forced evaluation and correct:
without an evaluation, `base_flag` keeps the value saved in the source file
(`object_update.cc:311` is the only sync to the original object), and the converter may then
force the objects hidden.

### `bge.logic.setAsyncLoadBudget()` / `getAsyncLoadBudget()`

```python
bge.logic.setAsyncLoadBudget(max_objects_per_batch=256)  # >= 1; the first READY load is always admitted
bge.logic.getAsyncLoadBudget()  # -> int
```

The budget is expressed in objects because the stages are indivisible: a time budget can only
be checked after it has been exceeded. See [Batching and Budget](#batching-and-budget-replaces-phases-2--21).

### `KX_BlendLoadHandle` (returned object)

| Attribute | Type | Description |
|---|---|---|
| `.state` | `str` | `'loading'` \| `'integrating'` \| `'done'` \| `'error'` \| `'cancelled'` \| `'freed'` |
| `.progress` | `float` | 0.0-1.0 (always safe to read, see Phase 3) |
| `.error` | `str\|None` | Error message; read only when `state == 'error'` (never while the loader thread may write it) |
| `.filepath` | `str` | Path as requested |
| `.collections` | `list[str]` | Requested collection names |
| `.objects` | `list[KX_GameObject]` | Objects created by this load, replicas included; built on each access with one pass over the target scene's objects. Empty unless `'done'` |
| `.cancel()` | `method` | Request cancellation (see semantics) |
| `.free()` | `method` | Unload a `'done'` load; on any earlier state it cancels |

`'loading'` covers the internal `LOADING`, `CANCEL_REQUESTED` and `READY` states.

**Lifetime guarantee:** the handle is always safe to query, including after integration, error,
unload or engine stop. The underlying `PendingBlendLoad` is kept alive by the handle's
`shared_ptr`. `cancel()` and `free()` raise `RuntimeError` once the engine has stopped.

**Cancel semantics:**

| State when called | Effect |
|---|---|
| queued (not started) | Exits before any I/O |
| reading on the loader thread | Best-effort: the current file read completes first (BLO has no cancellation point), then the staging Main is freed on the loader thread |
| `READY` | Immediate; the staging Main is freed on the loader thread |
| `'integrating'` | Immediate unload on the main thread (no KX objects exist yet) |
| `'done'` | No effect; use `free()` |

**Names:** local IDs that collide with existing names are renamed on merge (`"Cube"` →
`"Cube.001"`), as with Blender's Append. Use `handle.objects` instead of name lookups.

### Usage Example

```python
import bge

def on_zone_finished(handle):
    if handle.state != 'done':
        print(f'Zone load {handle.state}: {handle.error}')
        return
    for obj in handle.objects:          # no scene-wide name search
        if obj.get('spawn'):
            obj.worldPosition = [0, 0, 0]

def on_trigger_enter(cont):
    own = cont.owner
    if cont.sensors['Near'].positive and 'zone_handle' not in own:
        own['zone_handle'] = bge.logic.loadBlendAsync(
            '//levels/zone_02.blend',
            collections=['Terrain_Z2', 'Enemies_Z2', 'Props_Z2'],
            on_finish=on_zone_finished)

def on_trigger_leave(cont):
    own = cont.owner
    handle = own.get('zone_handle')
    if handle and not cont.sensors['Near'].positive:
        handle.free()                   # unload ('done') or cancel (still loading)
        del own['zone_handle']

def update_loading_bar(cont):
    handle = cont.owner.get('zone_handle')
    if handle and handle.state in ('loading', 'integrating'):
        bar_width = handle.progress * 200  # pixels
```

---

## Class Diagram

```
┌──────────────────────────┐
│  KX_KetsjiEngine         │
├──────────────────────────┤
│ - m_blendFileLoader      │───────┐ owns (unique_ptr)
│ - m_freeAsyncLoadsOnExit │       │
│ + GetBlendFileLoader()   │       │
└──────────────────────────┘       ▼
                    ┌───────────────────────────────────┐
                    │  KX_BlendFileLoader               │
                    ├───────────────────────────────────┤
                    │ - m_pool (background serial)      │── one dedicated pthread
                    │ - m_pending   vector<shared_ptr>  │──┐
                    │ - m_loaded    vector<shared_ptr>  │──┤
                    │ - m_batch     vector<shared_ptr>  │──┤
                    │ - m_stage (IDLE | AWAIT_EVAL)     │  │
                    │ - m_batchScene, m_batchGeneration │  │
                    │ - m_maxObjectsPerBatch            │  │
                    │ + RequestLoad()                   │  │
                    │ + IntegratePendingLoads()         │  │
                    │ + Cancel() / Free()               │  │
                    │ + Shutdown() / FreeLoadedIds()    │  │
                    │ # LoadThreadFunc() [static]       │  │
                    │ # StageMergeAndLinkHidden()       │  │
                    │ # StageConvertAndReveal()         │  │
                    │ # Unload()                        │  │
                    └───────────────────────────────────┘  │
                                                           ▼
                                            ┌──────────────────────────┐
                                            │ PendingBlendLoad         │
                                            ├──────────────────────────┤
                                            │ input:  filepath(s),     │
                                            │   names, target_scene,   │
                                            │   scene_generation,      │
                                            │   on_finish (PyObject*)  │
                                            │ loader: staging_main,    │
                                            │   root_collections,      │
                                            │   root_objects,          │
                                            │   force_merge_ids,       │
                                            │   owned_uids,            │
                                            │   num_objects, error_msg │
                                            │ main:   container        │
                                            │ sync:   state, progress  │
                                            └──────────────────────────┘
                                                ▲                   ▲
                                     shared_ptr │                   │ shared_ptr
                                                │                   │
                                    ┌───────────┴──────────────┐  ┌─┴─────────────────────────┐
                                    │ KX_PyBlendLoadHandle     │  │ Queued loader task        │
                                    ├──────────────────────────┤  ├───────────────────────────┤
                                    │ - m_load (shared_ptr)    │  │ taskdata = new shared_ptr │
                                    │ + state, progress, error │  │ deleted by LoadTaskFree() │
                                    │ + filepath, collections  │  └───────────────────────────┘
                                    │ + objects                │
                                    │ + cancel(), free()       │
                                    └──────────────────────────┘

┌──────────────────────────┐        ┌──────────────────────────────────────────┐
│ KX_Scene                 │        │ BL_Converter                             │
├──────────────────────────┤        ├──────────────────────────────────────────┤
│ + GetGeneration()        │        │ + RemoveConvertedData(is_owned)          │
│ + ConvertBlenderObjects- │        │   (extracted from FreeBlendFile)         │
│   Batch(Set<Object*>)    │        └──────────────────────────────────────────┘
└──────────────────────────┘
```

---

## Call Sequence

### Nominal Case (No Error)

```
[Python]  bge.logic.loadBlendAsync('//level.blend', collections=['Enemies'], on_finish=cb)
    │
    └──► [C++] gLoadBlendAsync()
            ├─ parse args, scene = current scene unless given
            └──► KX_BlendFileLoader::RequestLoad()
                    ├─ make_shared<PendingBlendLoad>, Py_XINCREF(cb), m_pending.append()
                    ├─ BLI_path_abs() + BLI_exists()                       (main thread)
                    ├─ BKE_main_new() + BKE_main_init_from_reference()     (main thread)
                    ├─ BLI_task_pool_push(m_pool, LoadThreadFunc,
                    │                     new shared_ptr(load), true, LoadTaskFree)
                    └─ return shared_ptr → KX_PyBlendLoadHandle → Python

[Loader thread]  LoadThreadFunc()
    ├─ entry cancel check
    ├─ BKE_reports_init(&reports, RPT_STORE)
    ├─ context_new(&params{bmain = staging, APPEND_RECURSIVE, no FILE_RELPATH, scene = null})
    ├─ library_add(lapp, path, nullptr)              (context opens and owns the handle)
    ├─ item_add("Enemies", ID_GR) + library_index_enable(item, 0)
    ├─ BKE_blendfile_link() / BKE_blendfile_append()  (staging only; no init_done/finalize)
    ├─ root_collections = { item_newid_get(item) }
    ├─ context_free()
    ├─ snapshot: force_merge_ids, owned_uids, num_objects
    └─ CAS LOADING → READY (acq_rel)

[Main thread — frame N]  NextFrame() → IntegratePendingLoads()
    ├─ GatherReadyBatch(): READY + live scene (pointer, then generation) → m_batch
    └─ StageMergeAndLinkHidden()
          ├─ BKE_main_merge(G_MAIN, &force_merge_ids, &staging_main)  → staging_main = nullptr
          ├─ container = BKE_collection_add(master, "AsyncLoad"); root → container
          ├─ BKE_ntree_update_tag_all + BKE_main_ensure_invariants(loaded trees)
          ├─ BKE_main_collection_sync() + BKE_view_layer_synced_ensure()
          ├─ container LayerCollection |= LAYER_COLLECTION_HIDE
          ├─ DEG_relations_tag_update(G_MAIN)
          └─ INTEGRATING
    tick loop (logic, physics)
    Render(): BKE_scene_graph_update_tagged() → relations rebuild + eval (hidden, not drawn)

[Main thread — frame N+1]  NextFrame() → IntegratePendingLoads()
    ├─ StageConvertAndReveal()
    │     ├─ FindLiveScene()
    │     ├─ scene->ReinitBlenderContextVariables()
    │     ├─ BKE_scene_graph_evaluated_ensure()          (O(1) after render N)
    │     ├─ scene->ConvertBlenderObjectsBatch(objects)  (one SETLOOPER pass)
    │     ├─ container LayerCollection &= ~LAYER_COLLECTION_HIDE
    │     ├─ DEG_id_tag_update_ex(scene, ID_RECALC_BASE_FLAGS)
    │     └─ DONE
    └─ ReapTerminalLoads(): m_pending → m_loaded, then cb(handle)
    tick loop: new objects take part in logic and physics
    Render(): first draw of the new objects

[Python, later]  handle.free()
    └──► KX_BlendFileLoader::Free() → Unload()
            ├─ session_uid pass over G_MAIN → alive owned ids
            ├─ BL_Converter::RemoveConvertedData(is_owned)
            ├─ BKE_id_multi_delete(G_MAIN, alive)   (container included)
            └─ DEG_relations_tag_update(G_MAIN) → FREED
```

---

## Pre-Implementation Checklist

### Open Items (v3.3)

#### BLOCKER: TSAN Run on a Dedicated Corpus

The static audit is not sufficient (see [BLO Thread Safety Audit](#blo-thread-safety-audit)):
reachable BKE code takes `G.main` implicitly through `DEG_id_tag_update()`. Run the loader under
TSAN while the main thread rebuilds and evaluates depsgraphs (load during a heavy scene), with:

- metaball objects (remap post-process path, `lib_remap.cc:382/391`);
- node groups, shader and geometry node trees;
- files saved by older Blender/UPBGE versions (versioning code runs on the loader thread);
- packed images and sounds;
- library chains (the loaded file links from other files, including from the game file itself).

Any report in this run blocks the merge.

#### MEASURE: Stage Costs and Default Budget

With the `tc_asyncload` profiler category, measure stage 1, stage 2 and the render passes of
frames N and N+1 on representative assets. Set the default `max_objects_per_batch` from these
numbers. If the first draw of new content (shader compilation, VBO extraction, texture upload)
dominates frame N+1, add a third stage that reveals in frame N+2; this is one more case in the
stage switch, at the cost of one tick where converted objects are invisible.

#### VERIFY During Implementation

- [ ] The hidden container is not drawn by any render engine used in game mode (EEVEE and
      `eevee_game`) — visual test.
- [ ] The hoisted collection remap in `BL_ConvertBlenderObjects()` passes the existing regression
      scenes (the initial scene conversion path changes as well).
- [ ] `BKE_blendfile_link_append_context_item_newid_get()` returns the local ID after
      `BKE_blendfile_append()` (debug assert on `!ID_IS_LINKED`).

#### Asset Policy

Save streamed `.blend` files with the UPBGE version of the runtime. Loading is faster and
versioning code — the least audited part of the loader thread's call graph — never runs.

#### Side Finding (separate fix, outside this proposal)

`KX_Scene::ConvertBlenderObjectsList(..., asynchronous = true)` contains
`while (!task.objectslist.size() != 0)`: with an empty list it calls `back()` on an empty
vector and then `delete`s a `blender::Object`. Undefined behavior.

### Status of the v3.2 Items

The review against UPBGE `e7c5aac` found defects in v3.2 that are fixed in this version; the
full list is in [Changes in v3.3](#changes-in-v33).

### ~~BLOCKER 1: TSAN Validation of `BLO_library_link_begin`~~ → REOPENED ⚠️

The v3.2 conclusion relied on a grep that cannot see implicit `G.main` access. TSAN is a blocker
again (see Open Items).

### ~~BLOCKER 2: Name-based lookup post-merge~~ → SUPERSEDED ✅

The v3.2 premise was wrong: `BKE_main_merge()` never produced `"Enemies.001"`. Two local IDs with
the same name *match* (`main.cc`, `are_ids_from_different_mains_matching()`), so the loaded ID
was discarded and its users remapped to the unrelated existing one. v3.3 force-merges every
local ID (renamed on collision, Blender's append semantics) and resolves requested roots by
pointer with `BKE_blendfile_link_append_context_item_newid_get()`. No tags, no name lookup.

### ~~BLOCKER 3: scene_generation validation~~ → RESOLVED ✅ (validation changed)

The generation counter stays. v3.2 compared it through `scene->GetGeneration()` on a possibly
destroyed scene (use-after-free read). v3.3 compares the raw pointer against
`engine->CurrentScenes()` first and reads the generation only on a match (`FindLiveScene()`),
before stage 1 and again before stage 2.

### ~~BLOCKER 4: Depsgraph phase safety~~ → SUPERSEDED ✅

No forced evaluation. Stage 1 tags relations; the existing render-pass update evaluates;
stage 2 converts. The loader no longer calls `CTX_data_depsgraph_on_load()`: it returns the
depsgraph of the context scene, which is the last rendered scene, not the target scene.

### ~~BLOCKER 5: Collection linking post-merge~~ → SUPERSEDED ✅

Only requested roots are linked, into one hidden container per load. v3.2 linked every loaded
collection, including child collections (duplicated in the hierarchy) and instance collections
(made visible at the origin and converted as game objects). `BKE_main_collection_sync_remap()`
is no longer called by the loader: it is meant for pointer remapping and rebuilds caches for the
whole Main.

### ~~VERIFY: `BKE_main_merge` Signature~~ → VERIFIED ✅ (overload used)

```cpp
// BKE_main.hh:545
void BKE_main_merge(Main *bmain_dst,
                    Set<ID *> *force_merge_src_ids,
                    Main **r_bmain_src,
                    MainMergeReport &reports);
```

`*r_bmain_src` is always freed and set to `nullptr`. IDs in `force_merge_src_ids` are always
moved; `BKE_libblock_management_main_add()` renames them on collision.

### ~~VERIFY: `ConvertBlenderCollection` Method Name~~ → VERIFIED ✅ (not used)

The second parameter of `KX_Scene::ConvertBlenderCollection()` is `asynchronous`, not
"libloading". With `true` it pushes `BL_ConvertBlenderObjects()` to a TBB task and immediately
calls `BLI_task_pool_work_and_wait()` (`KX_Scene.cpp:1154-1183`). The loader does not use it:

- **Not asynchronous.** The main thread blocks until the conversion ends (or executes the task
  itself inside the wait), so the frame pays the same cost plus the task overhead.
- **Not safe off the main thread.** The conversion instantiates Python components and custom
  objects (`PyObject_Call` on user classes, `BL_DataConversion.cpp:709/992`, reached from `:1744`)
  while the main thread holds the GIL inside the wait. It also tags G_MAIN depsgraphs
  (`:1330-1333`) and inserts into KX scene lists and the Bullet world.
- **Same algorithmic cost.** It still calls `BL_ConvertBlenderObjects()` once per object, so the
  O(K · N · log N) scene walk of [Batch Conversion](#batch-conversion) remains.

The loader uses `ConvertBlenderObjectsBatch()`, which is the same converter with the
single-object filter generalized to a set.

### ~~VERIFY: `KX_Scene::GetGeneration()` Availability~~ → RESOLVED ✅

KX_Scene does not have a generation counter; it is added (~5 lines):

```cpp
// In KX_Scene.h:
private:
    static std::atomic<uint64_t> s_generation_counter;  // Global monotonic counter
    uint64_t m_generation;
public:
    uint64_t GetGeneration() const { return m_generation; }
// In KX_Scene.cpp (global scope):
    std::atomic<uint64_t> KX_Scene::s_generation_counter{0};
// In KX_Scene constructor:
    m_generation = ++s_generation_counter;
```

### ~~VERIFY: G_MAIN->filepath in blenderplayer/standalone~~ → VERIFIED ✅

**Tested in all 5 execution modes:**

| Mode | Maggie1 (m_maggie) | G_MAIN | Match? |
|---|---|---|---|
| Embedded (editor) | `C:\...\Untitled.blend` | `C:\...\Untitled.blend` | ✅ |
| Blenderplayer inside (from editor) | `C:\...\Untitled.blend~` | `C:\...\Untitled.blend~` | ✅ |
| Blenderplayer outside (external) | `E:\...\bin\Release\Untitled.blend` | `E:\...\bin\Release\Untitled.blend` | ✅ |
| Runtime (.exe export) | `E:\Test\Test\Test.exe.blend` | `E:\Test\Test\Test.exe.blend` | ✅ |
| Runtime + linked blend | Changes to linked.blend | Changes to linked.blend | ✅⚠️ |

**Note on runtime + linked blend:** G_MAIN changes when another .blend is loaded. This is safe:
`RequestLoad()` copies G_MAIN's filepath and colorspace into the staging Main once
(`BKE_main_init_from_reference()`), and the loader thread only uses that copy.

---

## Performance Model

v3.2 listed per-step milliseconds that had not been measured; several were wrong in kind
(scene-proportional work presented as content-proportional). v3.3 states the cost structure and
leaves the numbers to the `tc_asyncload` measurement (checklist).

### Before (Blocking)

```
Frame with level load:    hundreds of ms (FROZEN)
├─ Full .blend read
├─ Sync conversion
└─ Merge
```

### After (Async, v3.3)

| Phase | Thread / frame | Proportional to |
|---|---|---|
| `RequestLoad()` | main, any frame | O(1) + one `stat()` (`BLI_exists`) + `BKE_main_new()` |
| Read, versioning, append, ownership snapshot | loader thread | file size and content — **off the main thread** |
| Stage 1: `BKE_main_merge()` | main, frame N | all IDs in G_MAIN (destination name map) + loaded IDs; it also clears G_MAIN's namemap, so the next ID creation or rename rebuilds it |
| Stage 1: node tree invariants | main, frame N | loaded node trees |
| Stage 1: container, links, hide | main, frame N | requested roots |
| Relations rebuild | render pass, frame N | whole scene graph, **once per batch** |
| Evaluation of new objects | render pass, frame N | loaded content (modifiers, geometry nodes) |
| Stage 2: evaluated-ensure | main, frame N+1 | O(1) when render N evaluated |
| Stage 2: batch conversion | main, frame N+1 | scene bases (one pass) + loaded objects (meshes, materials, Bullet shapes) |
| Stage 2: reveal | main, frame N+1 | base-flag re-evaluation, no rebuild |
| First draw | render pass, frame N+1 | loaded content: shader compilation, VBO extraction, texture decode and upload |
| Unload | main, one frame | all IDs in G_MAIN + all KX objects + deleted IDs; one relations rebuild |

Properties that hold regardless of the measured numbers:

- No phase is executed twice. v3.2 forced an evaluation and then let the render pass run again;
  per-object batches repeated the evaluation call per batch.
- The relations rebuild and the conversion pass are paid once per **batch**, not once per load.
- The two heaviest main-thread groups (merge + rebuild + evaluation, and conversion) never share
  a frame.
- Integration runs once per rendered frame, never once per catch-up tick.
- The loader thread cannot be stolen into the main thread (no TBB task).

### Costs Outside the Loader's Control

Listed so that they are measured, not assumed away:

- **Shader compilation** for materials not seen before (first draw, frame N+1).
- **Bullet shapes** for triangle meshes (`btBvhTriangleMeshShape` build) during conversion.
- **Texture decode** (disk read + PNG/EXR decompression) happens lazily on the main thread at the
  first `BKE_image_acquire_ibuf()`; Phase 4 currently covers only the GPU upload (see Roadmap).

---

## Batching and Budget (replaces Phases 2 + 2.1)

v3.2's per-object time slicing is removed. It did not spread the cost: every object was linked
in step 3, so the first evaluation already processed every new ID node; each batch repeated the
evaluation call; collections were converted again at the end; objects were visible without logic
between frames; and the time budget was checked only after a whole batch had run.

### Rules

1. **One batch in flight.** Stage 1 of batch k+1 never runs in the same frame as stage 2 of
   batch k. Throughput is one batch every two frames, of variable size.
2. **Coalescing.** A batch takes every `READY` load of the same target scene, in FIFO order,
   while the cumulative `num_objects` fits in `max_objects_per_batch`. With the serial loader
   thread and requests issued in bursts (entering a zone that needs many small files), several
   loads are `READY` together; integrating them as one batch pays one relations rebuild and one
   conversion pass instead of one per load.
3. **Budget in objects.** The stages are indivisible: a time budget is known only after it has
   been exceeded. The object count predicts both the evaluation and the conversion cost before
   any work starts.
4. **No starvation.** The first `READY` load of a batch is always admitted, even if it alone
   exceeds the budget.
5. **One scene per batch.** Stage 1 and stage 2 work on one view layer and one depsgraph. A
   `READY` load for another scene waits for the next batch.

### Python API

```python
bge.logic.setAsyncLoadBudget(max_objects_per_batch=256)
bge.logic.getAsyncLoadBudget()  # -> int
```

The initial default is a placeholder until the measurement in the checklist is done.

### Not Coalesced: The Merges

Each load still performs its own `BKE_main_merge()`, and each merge builds a name map over all
IDs in G_MAIN. If profiling shows this dominating with many small loads, the merges can be
chained (merge the staging Mains together first, then once into G_MAIN). That change interacts
with the "linked from the destination file" handling of `BKE_main_merge()` and is deferred until
the numbers justify it (see Roadmap).

---

## Progress Reporting (Phase 3)

`PendingBlendLoad::progress` is a `std::atomic<float>` written with relaxed stores: a reader
may see a slightly stale value, never a torn one. Cost: one store per stage.

| Value | Written by | Point |
|---|---|---|
| 0.0 | — | Queued |
| 0.1 | loader thread | Task started (after the entry cancel check) |
| 0.3 | loader thread | `BKE_blendfile_link()` returned |
| 0.45 | loader thread | `BKE_blendfile_append()` returned |
| 0.5 | loader thread | `READY` published |
| 0.75 | main thread | Stage 1 done (`INTEGRATING`) |
| 1.0 | main thread | Stage 2 done (`DONE`) |

BLO exposes no progress callback for linking, so the loader-thread values mark completed
calls, not bytes read. The value does not move in `ERROR`, `CANCELLED` or `FREED`.

```python
def update_loading_bar(cont):
    handle = cont.owner.get('load_handle')
    if handle:
        bar_width = handle.progress * 200  # pixels
```

---

## Future Roadmap

### Deferred: Legacy System Cleanup (Phase 1.1)

**Action (post-implementation):** remove the legacy async loading system:
- `BL_Converter::MergeAsyncLoads()` / `FinalizeAsyncLoads()` / `AddScenesToMergeQueue()`
- `async_convert()` static function
- `KX_LibLoadStatus` (replaced by `KX_BlendLoadHandle`)
- `BL_Converter::m_mergequeue`, `m_status_map`, `ThreadInfo`
- the `m_converter->MergeAsyncLoads()` call in `KX_KetsjiEngine::NextFrame()`
- the `m_converter->FinalizeAsyncLoads()` call in `KX_KetsjiEngine::StopEngine()`
- the `asynchronous == true` paths of `KX_Scene::ConvertBlenderCollection()` and
  `ConvertBlenderObjectsList()` (conversion on a TBB task with `work_and_wait`)

**Why deferred:** the legacy code does not interfere with the new system. It can be removed
once the new system is validated.

### Deferred: Texture Streaming (Phase 4)

**Goal:** large textures (4K, 8K) stall the GPU upload during the first draw.

**Proposed solution:**
- `KX_TextureStreamer` queue for deferred `GPU_texture_create_from_image`.
- Objects use a placeholder texture until the real texture is uploaded.
- Per-frame upload budget (e.g. 2 textures or 4 MB).

### Candidate: Texture Decode on the Loader Thread (Phase 4b)

Image decoding (disk read + PNG/EXR decompression) currently happens lazily on the main thread
at the first `BKE_image_acquire_ibuf()`. Acquiring and releasing the image buffers of the
staging images on the loader thread, before `READY`, would remove that cost from the first-draw
frame. Requires the same audit as the rest of the loader thread (image path resolution and
color management must not reach `G.main`), plus a check that the image cache limiter does not
evict the buffers before use.

### Candidate: Optional Third Stage (Reveal in N+2)

If the measurement shows that the first draw dominates frame N+1, reveal one frame later than
the conversion (see checklist).

### Candidate: Two-Stage Unload

If the unload spike matters for streaming, hide the container and remove KX data in frame U,
then delete the IDs in frame U+1.

### Candidate: Chained Merges

Merge the staging Mains of a batch together, then once into G_MAIN, so the G_MAIN name map is
built once per batch instead of once per load. Deferred until profiling shows the per-load merge
dominating (see Batching and Budget).

### Candidate: Converter Maps

`BL_SceneConverter` uses `std::map` keyed by pointers (node-based, pointer chasing). A
`blender::Map` would make every lookup O(1) with better locality. Independent of this proposal.

---

## Executive Summary

| Metric | Value |
|---|---|
| **New files** | 4 (all in `gameengine/Ketsji/`) |
| **New code lines** | ~970 (headers + implementation + Python handle) |
| **Changes to existing files** | 13 files, all in `source/gameengine/`: ~140 lines written (~40 mechanical) + ~150 moved |
| **Blender APIs used** | 100% public (BKE/BLO/BLI/DEG) |
| **Modifications to Blender** | 0 (zero) |
| **Threading model** | One dedicated loader thread (`BLI_task_pool_create_background_serial`), FIFO |
| **Thread safety** | Static audit + **TSAN blocker** (implicit `G.main` access exists in reachable code) |
| **Lifetime management** | `shared_ptr` held by loader, Python handle and queued task |
| **Locks on the hot path** | None (main-thread-only containers, atomic state) |
| **Integration** | Two stages per batch: merge + hidden link (frame N), convert + reveal (frame N+1) |
| **Depsgraph evaluation** | Reused from the existing render pass; never forced, never duplicated |
| **Merge collision handling** | Every local ID force-merged (renamed on collision, Blender append semantics) |
| **Batching** | Coalesced `READY` loads per scene, budget in objects, one batch in flight |
| **Conversion** | One `SETLOOPER` pass per batch (was one per object) |
| **Unload** | `handle.free()`, session_uid-validated, KX cleanup shared with `FreeBlendFile()` |
| **Progress reporting** | atomic float, coarse stages 0.0-1.0 |
| **Technical complexity** | Medium-High (threading, atomics, depsgraph, converter) |
| **Performance impact** | High positive (no freeze; no repeated scene-proportional work) |
| **Regression risk** | Medium-Low (new code isolated; converter change affects initial conversion too) |
| **Blockers** | 1 — TSAN run on the test corpus |
| **Estimated impl. time** | 7-9 days (core + batch conversion + unload + testing) |

---

## Changes in v3.3

Review of v3.2 against UPBGE master `e7c5aac`. Every row was checked in source.

| # | Issue in v3.2 | Fix in v3.3 | Evidence |
|---|---|---|---|
| C1 | `BKE_main_merge()` matches local IDs by name: loaded data colliding with existing names ("Cube", "Material") was silently replaced by unrelated G_MAIN data | Force-merge every local ID; roots by pointer | `main.cc` `are_ids_from_different_mains_matching()`; `BKE_main.hh:545` |
| C2 | `ConvertBlenderCollection(co, true)` meant "asynchronous": conversion on a TBB task with `work_and_wait`, Python components instantiated off the main thread without the GIL | `ConvertBlenderObjectsBatch()` | `KX_Scene.cpp:1154`; `BL_DataConversion.cpp:1744` |
| C3 | `BKE_reports_init(&reports, RPT_ERROR)`: flag mask without `RPT_STORE`, errors never stored or detected | `RPT_STORE`; per-item `newid_get()` validation | `BKE_report.hh` `ReportListFlags`; `report.cc` |
| C4 | `library_add()` returns `void` (did not compile) and steals the handle (double close) | Pass `nullptr`; index 0 | `BKE_blendfile_link_append.hh:230-242` |
| T1 | `BLI_task_pool_create()` (TBB): inline execution with one thread, main-thread work stealing, no join on free | `BLI_task_pool_create_background_serial()` | `task_pool.cc` constructor, `tbb_task_pool_run`, `~TaskPool` |
| T2 | Audit grep blind to implicit `G.main` (`DEG_id_tag_update()`), reachable from append | TSAN is a blocker; extended grep | `depsgraph_tag.cc:844-846`; `lib_remap.cc:382/391` |
| T3 | Skipping `finalize()` also skipped node tree tagging and `BKE_main_ensure_invariants()` | Redone in stage 1 for loaded trees | `blendfile_link_append.cc:375-404` |
| I1 | ABA check dereferenced a possibly destroyed scene | Live-list pointer match, then generation | — |
| I2 | `CTX_data_depsgraph_on_load()` used the context scene (last rendered), also read by conversion | `ReinitBlenderContextVariables()` on the target scene | `context.cc:1791`; `KX_Scene.cpp:497`; `BL_DataConversion.cpp:370` |
| I3 | Every loaded collection linked to the master collection, instance collections included | Roots only, in a per-load container | — |
| I4 | `BKE_main_collection_sync_remap()` (whole-Main cache rebuild) after linking | Removed; lazy `BKE_main_collection_sync()` | `layer.cc` |
| I5 | Integration inside the tick loop: repeated per catch-up tick | Once per `NextFrame()`, before the loop | `KX_KetsjiEngine.cpp:483-486` |
| P1 | Conversion O(K · N · log N): one `SETLOOPER` per object, `std::map::operator[]` inserting on miss; whole-Main remap per hidden object | Batch filter set, `find()`, hoisted remap | `BL_DataConversion.cpp:1296/1330`; `BL_SceneConverter.cpp:94-97` |
| P2 | Phase 2 per-object slicing did not slice (all IDs evaluated at once; re-conversion of collections; visible unconverted objects) | Two-stage pipeline by phase + hidden link | `deg_builder.cc:70-73`; `layer.cc:1127-1130` |
| P3 | `evaluate=false` still paid the rebuild (merge tags relations) and converted with a stale `base_flag` | Parameter removed | `lib_remap.cc` post-process; `object_update.cc:311` |
| L1 | No unload: loaded IDs merged into G_MAIN could never be freed | `handle.free()` + `Unload()` | `BL_Converter.cpp:571` (`FreeBlendFile` model) |
| L2 | Embedded mode without `GAME_USE_UNDO` left loaded IDs in the user's session | `FreeLoadedIds()` at stop | `BL_KetsjiEmbedStart.cpp:217/422` |
| L3 | Callback declared as `std::function` but called as `on_ready_pyobj`/`GetProxy()`; never called on error; run during iteration | `PyObject *on_finish`, every terminal state, after iteration | — |
| L4 | Final `store(READY)` could overwrite a racing cancel | CAS `LOADING → READY` | — |
| L5 | `staging_main` initialized with the filepath only | `BKE_main_init_from_reference()` (filepath + working colorspace) | `BKE_main.hh:479-482` |
| D1 | Patch integrated in `LogicEndFrame(this)` with a `KX_Scene*`; file and line counts inconsistent; "6-step" vs 7; `-fno-exceptions` claim false; `catch (bad_alloc)` after a partial merge | Patch, counts and text corrected; try/catch removed | — |

---

## Changes from v1

| Issue | v1 | v2 | v2.2 | v3.1 | v3.2 | v3.3 |
|---|---|---|---|---|---|---|
| BLO thread safety | "Verify later" | TSAN spike required | Source audit: VERIFIED | Downgraded to "by static audit, TSAN recommended" | — | **TSAN is a blocker again (implicit `G.main`)** |
| Handle lifetime | `unique_ptr` + raw | `shared_ptr` shared | — | — | — | + queued task holds a `shared_ptr` |
| Worker safety | "Verify later" | scene=nullptr | + NO FILE_RELPATH | — | — | + `library_add(nullptr)`, `RPT_STORE`, finalize work moved to main thread |
| Path resolution | Unclear thread | Main thread | + BLI_exists check | — | — | — |
| staging_main | In worker | In worker | **Main thread** | — | — | **`BKE_main_init_from_reference()` (filepath + colorspace)** |
| Multi-scene | Not addressed | target_scene | ABA: uint64_t monotonic | **`std::atomic<uint64_t>` monotonic counter in KX_Scene** | **Confirmed: `std::atomic<uint64_t>` (was incorrectly labeled in v3.1)** | Live-list pointer match before reading the generation |
| Integration point | LogicEndFrame | LogicEndFrame | **Top of NextFrame** | — | — | **Once per `NextFrame()`, outside the tick loop** |
| Post-merge pipeline | Not addressed | BKE_main_merge only | 6-step pipeline | **7-step: Pre-clear tags in G_MAIN** | — | **Two stages: merge + hidden link / convert + reveal** |
| Collision ID logic | Not addressed | Not addressed | Pointer collection | **ID_TAG_DOIT (safer)** | — | **Force-merge every local ID; roots by pointer; no tags** |
| Depsgraph | Not addressed | Not addressed | Forced eval + raw mode | — | — | **No forced eval; render pass reused; `evaluate` removed** |
| Texture Streaming | Not addressed | Not addressed | Phase 4 | **Deferred to Roadmap** | — | + decode on loader thread (Phase 4b candidate) |
| Modified files count | — | — | — | ~~5~~ (error) | **4 (corrected)** | **13, all in `source/gameengine/`** |
| Threading primitive | — | — | — | — | `BLI_task_pool` (TBB) per load | **One `BLI_task_pool_create_background_serial()` per loader** |
| Unload | — | — | — | — | — | **`handle.free()` + `FreeLoadedIds()` at stop** |
| Blockers | Multiple unknowns | 1 (TSAN) | 0 | **0 — ready to implement** | **0** | **1 (TSAN)** |

---

## References

- **Design document:** `UPBGE_AsyncBlendLoading.docx`
- **Architecture v1:** `UPBGE_AsyncBlendLoading_v1.md`
- **Blender PR #151907:** Introduction of `BKE_main_merge`
- **Source audit (BKE layer):** `blendfile_link_append.cc` — confirmed `params->bmain` pattern, zero `G_MAIN` access
- **Source audit (BLO layer):** `readfile.cc` — confirmed `library_link_begin`, `link_named_part`, `library_link_end` all use `mainvar`/`params->bmain`. Single explicit `BKE_main_blendfile_path_from_global()` gated behind `FILE_RELPATH` (line 5361 in the v3.2 revision, 5681 in `e7c5aac`)
- **v3.3 review baseline:** UPBGE master `e7c5aac` (2026-10-12): `main.cc` (`BKE_main_merge`, `BKE_main_init_from_reference`), `blendfile_link_append.cc`, `lib_id.cc`, `lib_remap.cc`, `report.cc`, `layer.cc`, `collection.cc`, `task_pool.cc`, `depsgraph_tag.cc`, `depsgraph_registry.cc`, `deg_builder.cc`, `object_update.cc`, `scene.cc`, `context.cc`, and in `source/gameengine/`: `KX_KetsjiEngine.cpp`, `KX_Scene.cpp`, `BL_DataConversion.cpp`, `BL_SceneConverter.cpp`, `BL_Converter.cpp`, `BL_KetsjiEmbedStart.cpp`, `LA_Launcher.cpp`
- **Source audit (BLO entry):** `readblenentry.cc` — `BLO_blendhandle_from_file` is pure file I/O, no globals
- **Frame loop analysis:** `KX_KetsjiEngine.cpp` — `NextFrame()`: `MergeAsyncLoads()` is inside the tick loop (`e7c5aac`: lines 483-486); v3.3 integrates once, before the loop. Per-scene depsgraph update in the render pass: `KX_Scene.cpp:727`
- **Legacy system analysis:** `BL_Converter.cpp` — `async_convert` broken (runs depsgraph on worker thread), `MergeAsyncLoads` operates at KX_Scene level not Blender Main level
- **APIs used:**
  - `source/blender/blenloader/BLO_readfile.hh` (BLO_blendhandle_*, BLO_library_link_*)
  - `source/blender/blenkernel/BKE_blendfile_link_append.hh` (BKE_blendfile_link, BKE_blendfile_append)
  - `source/blender/blenkernel/BKE_main.hh` (BKE_main_merge, BKE_main_new, BKE_main_init_from_reference, BKE_main_free)
  - `source/blender/blenkernel/BKE_main_invariants.hh` (BKE_main_ensure_invariants)
  - `source/blender/blenkernel/BKE_collection.hh` (BKE_collection_add, BKE_collection_child_add_no_sync, BKE_collection_object_add, BKE_collection_has_object_recursive, BKE_main_collection_sync)
  - `source/blender/blenkernel/BKE_layer.hh` (BKE_layer_collection_first_from_scene_collection, BKE_view_layer_need_resync_tag, BKE_view_layer_synced_ensure)
  - `source/blender/blenkernel/BKE_lib_id.hh` (BKE_id_multi_delete)
  - `source/blender/blenkernel/BKE_node_tree_update.hh` (BKE_ntree_update_tag_all)
  - `source/blender/blenkernel/BKE_report.hh` (BKE_reports_init, BKE_reports_contain, BKE_reports_string)
  - `source/blender/blenkernel/BKE_scene.hh` (BKE_scene_graph_evaluated_ensure)
  - `source/blender/depsgraph/DEG_depsgraph.hh` (DEG_id_tag_update_ex, DEG_relations_tag_update)
  - `source/blender/blenlib/BLI_task_c.hh` (BLI_task_pool_create_background_serial, BLI_task_pool_push, BLI_task_pool_free)

---

**Maintainer:** UPBGE Team
**Version:** 3.3
**Date:** 2026-10-12
**Reviewed by:** Claude (Architecture Review + BLO Source Audit) + Gemini pro; v3.3 review by Claude against UPBGE `e7c5aac`
**Note:** v3.3 fixes the defects listed in [Changes in v3.3](#changes-in-v33) and adopts: dedicated serial loader thread, two-stage batched integration, removal of `evaluate`, batch conversion, and unload in this version.
