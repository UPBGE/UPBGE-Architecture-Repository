# UPBGE Agent-Friendly Engine — Technical Plan v3.4

**Status:** Proposal, decisions partially closed (see section 4). v3.1 applied the review feedback (section 0.1); v3.2 narrowed the target environment to local desktops (section 0.2); v3.3 applied the second review and brought mouse injection into scope (section 0.3); v3.4 closes the open questions of section 16 (section 0.4)
**Scope:** Small, additive tooling. Engine-side code lives only under `source/gameengine/`. No upstream Blender changes are planned (see the gate in section 14).
**Primary goal:** Let an external AI agent build, run, test and iterate on a UPBGE game autonomously, with **zero measurable cost when the feature is unused** and **no LLM dependency in the engine**.

---

## 0. Changes versus v2

| # | Change | Reason |
|---|---|---|
| 1 | Project goal widened from "optimize a project" to "build a game autonomously": edit → run → observe → verify → iterate | Product vision |
| 2 | **Modify plane is now first-class**, implemented as headless `blender -b file.blend --python edit.py` (Option A) | `bpy` already covers logic bricks, components and runtime properties. No Blender changes needed |
| 3 | **Run plane** defined: player as a separate, killable process with structured result and crash-safe event log | The agent must know whether the game works, hangs or crashes |
| 4 | **Verify plane** added: frame-indexed input injection at the input-device level and Python assertion callbacks at chosen frames | "It runs" is not "it works" |
| 5 | Profiling and benchmark kept, but moved after the closed loop | Priority follows the new vision. Performance rules remain unchanged |
| 6 | Explicit gate for touching Blender code | Policy: only when strictly necessary |
| 7 | Second POC: build a small game from a spec with hidden tests | Measures the actual vision |

### 0.1 Changes v3 → v3.1 (review feedback applied)

| # | Change | Section |
|---|---|---|
| 1 | Snapshot covers `.blend` plus script folder; a failed batch restores scripts | 5.1, 5.3 |
| 2 | Exit-code table, status precedence, tolerant parsing of truncated JSONL | 6.2 |
| 3 | Headless GL: Xvfb + Mesa llvmpipe is the supported CI path; software GL is invalid for benchmarks | 6.5, 9.6 |
| 4 | Determinism bounded to same machine and build; `PYTHONHASHSEED=0`; Python `random` seeded by the scenario bootstrap; engine commit in the environment block | 6.4, 9.6 |
| 5 | Assertion execution point fixed: after logic and physics, before render | 8.2 |
| 6 | `CM_FrameSample` corrected to exactly 128 B; scope table validated as a tree | 9.2 |
| 7 | Error ring: fixed inline text, identity compared on hash match, no heap after init | 7.1 |
| 8 | `--fail-fast` and `agent.abort()` | 6.1, 8.2 |
| 9 | `Import` operation family | 5.2 |
| 10 | Engine commit pinned for the capability matrix | 4, 10 |
| 11 | In-engine `--max-seconds` replaced by `--heartbeat` plus external kill (an in-thread watchdog cannot interrupt a hung frame) | 6.1, 6.2 |
| 12 | Minor: `--fixed-hz`, `scene=` parameters, per-iteration run directories, late input registration error, `save_on_error` wording, Python GC stats | various |

Not adopted: `gc.disable()` by default. GC statistics are recorded instead (section 6.4).

### 0.2 Changes v3.1 → v3.2 (scope clarification: indie developers on local machines with a display)

| # | Change | Section |
|---|---|---|
| 1 | Headless-GL / CI requirement removed. C8 redefined; Xvfb + llvmpipe and software-renderer rejection dropped (supersedes 0.1 item 3) | 4, 6.5, 10, 13, 15 |
| 2 | New section on target users and environment | 1.1 |
| 3 | Local desktop requirements: fixed window geometry flags, no focus stealing (to verify), Windows process and path handling | 6.1, 6.5 |
| 4 | Benchmark noise on a machine in use: load sampling, `noisy_environment` flag, GPU adapter recorded | 9.6 |
| 5 | Phase 0 and Phase 1 rows updated | 10 |
| 6 | Open question on platform support order | 16 |

### 0.3 Changes v3.2 → v3.3 (second review, mouse injection in scope)

| # | Change | Section |
|---|---|---|
| 1 | Mouse injection in scope now: relative and absolute events, plus a **virtual pointer** in agent mode so the real OS cursor is never moved or warped. D7 closed. Event struct grows to 16 B | 4, 6.5, 8.1, 10 |
| 2 | Script policy: agent-managed scripts are external files only; helper prevents internal text datablocks from shadowing them | 5.2, 10 |
| 3 | `bpy.data` first; `bpy.ops` only through one wrapper with `temp_override`; Phase 0 produces an operation matrix | 5.2, 10 |
| 4 | Pre-flight validation (`ast.parse` / `compile`, referenced-module check) before launching Blender | 5.1 |
| 5 | `CM_ErrorRecord.file` holds a project-relative path (tail kept), 96 B | 7.1, 9.6 |
| 6 | Windows: `SetErrorMode` in the launcher, log file sharing note, heartbeat as one fixed-size write | 6.1, 6.2, 6.5 |
| 7 | Benchmarks: thermal soak, interleaved A-B-A-B trials, drift check | 9.6 |
| 8 | Input recording and `repro.json` on failure | 8.1 |
| 9 | Optional screen-space positions in `inspect_*` | 7 |
| 10 | Determinism self-check: `agent.state_hash()` and an empirical repeat-run audit in Phase 0 | 6.4, 10 |

### 0.4 Changes v3.3 → v3.4 (open questions resolved)

| # | Change | Section |
|---|---|---|
| 1 | D2 closed: top-level stage numbers come from `KX_TimeCategoryLogger`; the flat scope table covers only what it cannot | 9.2 |
| 2 | D3 closed: GPU timer queries are added to the Blender GPU module as the first candidate exception to the gate, conditional on Phase 0 finding no reusable API. Other backends report unavailable | 9.3, 14 |
| 3 | `WITH_AGENT_API` compile-time option (default ON) plus the runtime flag | 2, 4 |
| 4 | Benchmark numbers only from the standalone player; embedded runtime out of scope | 1, 6.1, 9.6 |
| 5 | Windows, Linux and macOS supported from day one, with a definition of "supported" | 1.1, 6.5, 10 |
| 6 | Repository layout: `source/gameengine/Agent/` (C++ and Python), install rule in the gameengine CMake, launcher runs with Blender's bundled Python | 3.2, 10 |
| 7 | C13 pin rule: HEAD of the main branch when Phase 0 starts | 4, 10 |
| 8 | Section 16 now lists only items answered by Phase 0 audits | 16 |

---

## 1. Goals and non-goals

### Goals
1. The agent edits `.blend` projects (objects, meshes, materials, physics, logic bricks, components, properties) without a human in the editor.
2. The agent launches the player, sees whether it works, stops it, edits, relaunches, iterating until a stated game spec passes its tests.
3. The agent can drive the game (input) and verify behavior (assertions, state, screenshots).
4. Errors, stats and profiles are available as bounded, structured data.
5. Benchmarks are repeatable and noise-aware so performance work is real, not imagined.

### Non-goals
- Any LLM, embedding, vector store or planner inside the engine.
- Engine-side transactions or rollback.
- Replacing existing `bge.*` / `bpy` APIs.
- Always-on instrumentation.
- Multithreading of engine state.
- Asset generation (textures, meshes beyond primitives and procedural data) in the first versions.
- Headless-GL, CI or server deployment (not required for the target audience).
- Support for the embedded (in-Blender viewport) runtime: the agent API targets the standalone player only (C20).

### 1.1 Target users and environment

- Indie game developers running the agent on **their own computers**, in a graphical desktop session, with a real GPU.
- The player never has to run without graphics. (`blender -b` for edit batches needs no GL because it only runs `bpy`.)
- The agent **shares the machine with the developer**, which drives window handling, input isolation and benchmark noise rules (sections 6.5 and 9.6).
- Desktop platforms: **Windows, Linux and macOS, all supported from day one** (C21; definition in section 6.5). Windows is expected to be the most common.

---

## 2. Hard constraints

| Constraint | Rule |
|---|---|
| Upstream policy | New engine code only in `source/gameengine/`. Blender code only through the gate in section 14. |
| Cost when off | One predicted branch (or a pointer/count compare) per instrumented site. No clock reads, allocation or string work. |
| Cost when on | Bounded and documented. Profiling and benchmark runs are never the same run. |
| Allocation | None on the frame path. Buffers sized when the feature is enabled. |
| Hot data | Flat arrays, cache-line aligned, sized for L2. No `std::map`, `std::string`, or virtual dispatch on recorded paths. |
| Threads | Engine state stays single-threaded. No server thread inside the player (a Python thread would also contend for the GIL). |
| Python | Agent API is a cold path: called on demand or at selected frames, never per frame. |
| Build | `WITH_AGENT_API` CMake option, default ON (C19). When OFF, instrumentation macros compile to nothing and `--agent` does not exist (for example for builds used to ship games). When ON, nothing runs until `--agent` is passed. |

---

## 3. Architecture

```
                         External agent (any LLM / framework)
                                        │
        ┌───────────────┬───────────────┼───────────────┬───────────────┐
        │               │               │               │               │
     MODIFY            RUN           OBSERVE          VERIFY         MEASURE
  blender -b       player process   bge.agent        input inject    profile /
  edit.py          (separate,       inspect,         + assertions    benchmark /
  (bpy)            killable)        errors, stats,   + screenshot    compare
                                    screenshot       diff
        │               │               │               │               │
        └───────────────┴───────────────┴───────────────┴───────────────┘
                                        │
                                      UPBGE
```

### 3.1 The autonomous loop

```
spec ─► agent writes edit.py ─► snapshot .blend ─► blender -b (apply batch)
          ▲                                                 │
          │                                     edit_result.json (what changed, bpy errors)
          │                                                 ▼
          │                           upbge player --agent scenario.py --frames N
          │                                                 │
          │                       run_result.json + run.events.jsonl + screenshots
          │                                                 ▼
          └──────── agent decides: fix / extend / revert snapshot / done
```

All planes communicate through **files and process exit codes**. No sockets, no shared memory, no engine threads.

### 3.2 Repository layout and tooling (C22)

```
source/gameengine/Agent/
├── CMakeLists.txt            # WITH_AGENT_API option, install rule for python/
├── (C++ sources)             # error ring, profiler, input injection, virtual pointer, bge.agent module
└── python/
    ├── upbge_agent_edit/     # helper used inside `blender -b` (section 5)
    └── upbge_agent_run/      # launcher: spawn, heartbeat, kill, classify (section 6)
```

- Everything is versioned with the engine, so the pinned engine commit (C13) covers helper, launcher and engine together.
- An install rule in the gameengine CMake places `python/` in the release layout. Phase 0 confirms this is sufficient without touching Blender's install CMake; if it is not, that is a gate case (section 14).
- The launcher runs with the Python interpreter distributed with Blender, so developers do not need a system Python (its path in each platform's release layout is confirmed in Phase 0). Agent frameworks call the launcher as a CLI command.
- The GPU timer-query API (C18) is the only file outside `source/gameengine/` currently planned.

---

## 4. Decisions

### Closed

| ID | Decision |
|---|---|
| C1 | Edit plane uses headless Blender per batch (`blender -b file.blend --python edit.py`). Persistent `bpy` worker deferred until latency proves a problem. |
| C2 | `bpy` covers logic bricks, components and runtime properties for the pinned engine commit (C13). No Blender changes for editing. |
| C3 | Player runs as a separate process. Agent starts and kills it. |
| C4 | Input injection at the input-device level, frame-indexed events (section 8.1). |
| C5 | Gameplay assertions are Python callbacks registered for specific frames (section 8.2). No declarative DSL. |
| C6 | Engine-side transactions dropped. Rollback is restoring the project snapshot (`.blend` plus script folder). |
| C7 | No server thread, no Python server. Control via CLI flags and files. |
| C8 | Target is indie developers on their own machines with a display and a real GPU. No headless-GL or CI support is required. |
| C9 | Exit codes: 0 OK, 10 ASSERT_FAIL, 11 ERROR, 12 LOAD_FAIL. Anything else is CRASH. Part of the versioned schema. |
| C10 | Snapshot per iteration = `.blend` plus script folder. Git is optional and only for text files. |
| C11 | Assertion callbacks run once per frame, after logic and physics, before render. |
| C12 | No in-engine watchdog. The engine writes a heartbeat; the launcher kills the process on a stale heartbeat or total timeout. |
| C13 | The capability matrix and C2 are pinned to a specific engine commit: the HEAD of the main branch on the day Phase 0 starts, recorded by Phase 0 and moved only deliberately. |
| C14 | Mouse injection is in scope: relative and absolute events, and a virtual pointer in agent mode so the real OS cursor is never moved or warped. |
| C15 | Agent-managed scripts are external files only; the helper prevents internal text datablocks from shadowing them. |
| C16 | `bpy.data` first; `bpy.ops` only through one wrapper with `temp_override` and structured errors. |
| C17 | D2 closed: top-level stage numbers come from `KX_TimeCategoryLogger`; the flat scope table covers only what it does not (scripts, sub-stages). |
| C18 | D3 closed: GPU timer queries are added to the Blender GPU module (additive API, GL implementation, other backends report unavailable), pending the Phase 0 check for a reusable existing facility (gate, section 14). |
| C19 | `WITH_AGENT_API` CMake option (default ON, can be turned OFF for builds used to ship games) plus the runtime `--agent` flag. |
| C20 | Benchmark numbers come only from the standalone player. The embedded runtime is not supported by the agent API. |
| C21 | Windows, Linux and macOS are supported from day one. "Supported" is defined in section 6.5. |
| C22 | Helper and launcher live in `source/gameengine/Agent/` (C++ and `python/`), installed by a rule in the gameengine CMake. |

### Still open (options in sections 9 and 10)

| ID | Question |
|---|---|
| D4 | Python error capture location |
| D6 | Pause/step semantics |

---

## 5. Modify plane

### 5.1 Execution model

```
blender -b project.blend --python edit.py -- --batch batch.json --out edit_result.json
```

- One launch applies a **batch** of operations. Startup and load cost is amortized per batch.
- The agent may write raw `bpy`, but should prefer the helper module below for validated operations.
- Before each batch the agent snapshots the `.blend` **and the project script folder** (for example `snapshots/0007/`). Revert restores both. Git may track the text files for readable diffs, but it is not the rollback mechanism, because a binary `.blend` bloats a repository.
- **Pre-flight (before launching Blender):** the helper runs `ast.parse` / `compile` on every script it wrote or the agent supplied, and checks that every module referenced by components and Python controllers exists in the script folder. Syntax and missing-module errors are reported in milliseconds instead of after a Blender start. An external linter (for example Ruff) is used only if it is already installed; it is not bundled.

### 5.2 Helper module (`upbge_agent_edit`, pure Python)

Lives alongside the engine sources (for example `source/gameengine/Agent/python/`) and is added to `sys.path` by `edit.py`. It is **not** a Blender change and has **no runtime cost** in the player.

Design rules:
- **Idempotent by name**: `ensure_object("Player", ...)` creates or updates. Re-running a batch after a failure is safe.
- **Validated**: unknown properties or missing referenced objects produce a structured error before mutating anything.
- **Structured result**: every operation returns `{op, ok, changed: [...], error}`.
- **Raw escape hatch**: `raw(code)` runs arbitrary `bpy`, with exceptions captured into the result.
- **Deterministic ordering**: operations apply in list order, and the result echoes that order.
- **Scripts are external only (C15).** Agent-managed scripts live as files in the script folder and load as regular modules. Internal text datablocks (`bpy.data.texts`) are not used for them. On every batch the helper checks that no internal text datablock shadows or duplicates an agent-managed script; if one exists it is synchronized from the file or reported as an error. Phase 0 verifies the module resolution order between internal texts and the script folder.
- **`bpy.data` first (C16).** Operations use `bpy.data`, `mathutils` and object/collection attributes wherever a data-level API exists. `bpy.ops` is used only where none does (expected for logic bricks, components and importers, to be confirmed in Phase 0), always through a single wrapper that applies `context.temp_override(...)` and converts poll failures (`RuntimeError`) into structured errors.

Operation families (initial):

| Family | Examples |
|---|---|
| Scene/world | create scene, set gravity, fps/tickrate, physics engine, vsync |
| Objects | primitive meshes, empties, cameras, lights, transform, parenting, collections |
| Mesh data | simple procedural meshes from vertex/face arrays |
| Materials | basic principled values, color, texture from existing file path |
| Physics | rigid body type, collision shape, mass, friction, character settings |
| Properties | game properties on objects |
| Logic bricks | add sensor/controller/actuator, set parameters, link bricks |
| Components | attach Python component, set its arguments |
| Import | glTF/FBX/OBJ and other formats via `bpy.ops.import_scene.*` (importer add-on must be enabled in the headless session) |
| Scripts | write/update `.py` files in the project's script folder |
| Save | save, optional snapshot |

### 5.3 Edit result schema

```json
{
  "schema_version": 1,
  "ok": false,
  "ops": [
    {"op": "ensure_object", "name": "Player", "ok": true, "changed": ["created"]},
    {"op": "add_sensor", "object": "Player", "ok": false, "error": "unknown sensor type 'Keybord'"}
  ],
  "saved": false
}
```

On the first failed operation the batch stops, the `.blend` is **not saved**, and the script folder is restored from the pre-batch snapshot, so the project stays consistent with its snapshot. `save_on_error=true` keeps and saves the partial state (debugging only).

### 5.4 Cost

None inside the player. Fixed cost per batch is Blender startup plus file load. Mitigations: batch many operations, keep scenes lean while iterating. If latency becomes the bottleneck, introduce a persistent `bpy` worker (reads JSON commands from stdin in a blocking loop, no threads) behind the same helper API.

---

## 6. Run plane

### 6.1 Launch

```
upbge --agent scenario.py --frames 600 --fixed-hz 60 --seed 1 \
      --out runs/0003/run_result.json --events runs/0003/run.events.jsonl \
      --heartbeat runs/0003/heartbeat --fail-fast [--no-vsync] [--uncapped]
```

| Flag | Purpose |
|---|---|
| `--agent <script>` | Scenario script (input, assertions, screenshots) executed by the player |
| `--frames N` | Stop after N logic frames |
| `--fixed-hz` | Deterministic integer logic rate (for example 60), ignoring wall clock |
| `--seed` | Seeds the engine RNG only. Python's `random` is seeded by the scenario bootstrap (section 6.4) |
| `--heartbeat` | File updated with the current frame counter every N frames (default 30), written as one small fixed-size write. The launcher uses it to detect a hung frame and tolerates a partial read |
| `--fail-fast` | End the run at the first failed assertion or error. `frames_run` reports the frames actually executed |
| `--window-size WxH`, `--window-pos X,Y` | Fixed window geometry, so screenshots and visual diffs are comparable and the window stays out of the developer's way |
| `--out` | Final structured result |
| `--events` | **Incremental** event log (errors, assertions), flushed as they happen |

Flags are parsed in `LA_SystemCommandLine` / launcher code. The flags are supported by the standalone player only (C20). Each iteration uses its own directory (`runs/0003/`) holding the result, event log, heartbeat and screenshots, so concurrent or successive runs never overwrite each other.

### 6.2 Crash and hang safety

A crashed process cannot write its final JSON. Therefore:

- Errors and assertion results are appended to `run.events.jsonl` and flushed when they occur (cold path, error path only). After a crash the agent still has everything up to the crash. The last line may be truncated mid-write, so the parser must tolerate and discard an incomplete final line.
- The agent-side launcher monitors the `--heartbeat` file and enforces a total wall-clock timeout. A stale heartbeat means a single frame is hung (for example a `while True` in a script). An in-engine watchdog cannot interrupt that: it would run on the same thread, and an extra thread breaks the threading rule (C7). The launcher kills the whole process tree (process groups on POSIX, Job Objects on Windows). On Windows the launcher also calls `SetErrorMode(SEM_NOGPFAULTERRORBOX | SEM_FAILCRITICALERRORS)` before creating the player process, which the child inherits, so a crash does not leave a Windows Error Reporting dialog keeping the process alive. `CREATE_NO_WINDOW` is not used: the player is a GUI application.
- Status classification is done by the agent-side launcher from exit code, presence of `run_result.json` and the event log:

| Status | Exit code | Meaning |
|---|---|---|
| `OK` | 0 | Frames reached (or clean abort), no errors, assertions passed |
| `ASSERT_FAIL` | 10 | At least one assertion failed |
| `ERROR` | 11 | Python or engine errors were recorded |
| `LOAD_FAIL` | 12 | Project failed to load (missing file, bad data). The engine exits with this code itself |
| `CRASH` | any other code, or a signal / Windows crash status | No final result written |
| `TIMEOUT` | assigned by the launcher | Killed by the launcher (total timeout or stale heartbeat). The engine never emits it |

Precedence: if both errors and failed assertions occurred, the exit code is 11 (`ERROR`, since errors often explain failures) and the result lists both. `LOAD_FAIL` is only possible before frame 0. The exit-code table is part of `schema_version`: changing it bumps the version.

### 6.3 Run result schema

```json
{
  "schema_version": 1,
  "status": "ASSERT_FAIL",
  "exit_code": 10,
  "engine_commit": "<git hash>",
  "frames_run": 600,
  "exit_reason": "frames_reached",
  "errors": [ { "severity": "ERROR", "exception": "AttributeError", "file": "player.py",
                "line": 82, "function": "update", "object": "Player",
                "first_frame": 12, "last_frame": 600, "count": 589, "message": "..." } ],
  "assertions": [ { "name": "player_moves_right", "frame": 120, "passed": false,
                    "message": "x=0.00 expected > 1.0" } ],
  "stats_last": { "frame_time_ms": 6.9, "objects": 42, "draw_calls": 31 },
  "screenshots": ["shots/f0120.png"],
  "log_tail": ["..."]
}
```

### 6.4 Determinism contract

Reproducibility is guaranteed **on the same machine and the same engine build**, not across platforms or builds. Bullet's results depend on build, platform and solver configuration, so cross-machine equality is not promised.

| Source of variance | Handling |
|---|---|
| Wall clock | `--fixed-hz` ignores it |
| Engine RNG | `--seed` |
| Python `random` | The scenario bootstrap calls `random.seed()` with the same seed. Documented for game scripts |
| Hash randomization | Launcher sets `PYTHONHASHSEED=0` in the environment **before** the interpreter starts |
| Physics | Single-threaded solver for verification runs; same build and machine |
| Input | Frame-indexed events (section 8.1) |
| Python GC | Not disabled (it is part of real game cost). GC counters (collections, time if available) are recorded in the result so spikes can be attributed |
| Engine version | Engine commit stored in every result and in the environment block |
| Self-check | `agent.state_hash()` (cold path) returns a hash of object transforms and physics state. Running the same scenario twice and comparing hashes at the same frames verifies reproducibility on the machine in use |

### 6.5 Local desktop environment

- **Window behavior:** the player opens at a fixed geometry (`--window-size`, `--window-pos`) and, where the OS allows, without taking focus from what the developer is doing. The window is **not minimized**: a minimized or fully occluded render can change vsync and timing behavior. Phase 0 verifies what the windowing layer actually permits.
- **Input isolation:** input is injected at the input-device level (section 8.1), never through OS-level events, so the agent does not interfere with the developer's own keyboard and mouse. The virtual pointer (C14) also stops engine-initiated cursor warps from moving the real cursor.
- **Shared machine load:** the launcher samples system CPU and GPU load before and during benchmark runs and records it. Results under high load are flagged `noisy_environment: true` (section 9.6).
- **Visual diffs** are only comparable between runs on the same GPU and driver version.
- **Windows specifics:** Job Objects for process-tree kill, `SetErrorMode` (section 6.2), paths with spaces and long paths, and antivirus latency on `blender -b` startup (measured in Phase 0). Log files written by the engine use the C standard library in append mode with `fflush` after each event: a flushed write survives a process crash (`fsync` is not needed) and stays readable by the launcher while the game runs. Raw `CreateFile` without sharing flags must be avoided.
- **GPU selection:** on machines with integrated and dedicated GPUs, the adapter used by each run is recorded, since it changes results completely.
- **Definition of "supported" (C21):** the code compiles on the platform and the Phase 1 exit criterion passes there. A capability that a platform or GPU backend cannot provide degrades explicitly (`available: false` in the result) instead of being silently absent.
- **Process control:** Job Objects and `SetErrorMode` on Windows; process groups on Linux and macOS.
- **Tick source:** `CM_ReadTicks` is implemented per architecture: `rdtsc` or `QueryPerformanceCounter` on x86, the architectural counter on ARM64. The counter on Apple Silicon is reported to run at 24 MHz (about 42 ns per tick; verify in Phase 0), which is adequate for stage-level scopes but coarse for tiny ones, so scopes shorter than a few ticks are not reported individually.
- **GPU timing:** may be `available: false` on backends without a timer-query implementation (for example Metal until one is added).
- **Test matrix:** the Phase 1 exit criterion is run on all three platforms with a real GPU.

---

## 7. Observe plane (`bge.agent`, cold path)

```python
bge.agent.project_info()
bge.agent.list_scenes()
bge.agent.inspect_scene(scene=None, limit=50, offset=0, fields=None, sort=None)
bge.agent.inspect_object(name, scene=None, fields=None)
bge.agent.inspect_logic(name, scene=None)       # Phase 5
bge.agent.stats()
bge.agent.errors(since_frame=0, severity=None, limit=50)
bge.agent.screenshot(path=None, camera=None, resolution=None)
```

Rules:
- Every payload has `schema_version` and unit-suffixed keys (`_ms`, `_mb`).
- Size-bounded: `inspect_scene` returns counts plus a page of summaries, never a full dump.
- Object lookup by name: verify whether `EXP_ListValue` lookup is linear. If so it is documented as O(n), acceptable on a cold path, forbidden in loops.
- Counters are plain `uint32_t` increments collected in one 64-byte struct. Memory use is queried on demand only.
- `inspect_object` and `inspect_scene` entries may include `screen`: `{"x": px, "y": px, "on_screen": bool}` for a chosen camera (default: the active camera), requested through `fields=["screen"]` and off by default. It uses the camera's existing screen-position projection and lets a multimodal agent relate a screenshot to objects without image analysis. No overlay is drawn into the image.

### 7.1 Error capture (decision D4)

| Option | Cost | Reliability | Notes |
|---|---|---|---|
| **A. Central report function at existing `PyErr_Print` sites** | Error path only | High | One helper called from `EXP_PythonCallBack.cpp`, `SCA_PythonController.cpp`, `KX_PythonComponent.cpp`, `KX_PythonProxy.cpp`. Fetch and normalize the exception, hash `(file, line, type)`, update a fixed ring entry, append to the events file on first occurrence. Original console printing is preserved. |
| B. Replace `sys.stderr` and parse text | Every print | Fragile | Rejected. |
| C. `sys.excepthook` | None | Never fires | Engine catches at call sites. Rejected. |

Recommendation: A. A script that fails every frame costs one hash probe and one increment per frame after the first occurrence.

```cpp
// All text is stored inline in fixed-size buffers, so no error path allocates after
// init, including eviction. The identity fields (file, line, exception) stay in the
// record so a hash match is confirmed with a compare: two distinct errors that
// collide on the 64-bit key are never merged silently.
struct CM_ErrorRecord {
  uint64_t key;            // hash(file, line, exception type); 0 = empty slot
  uint32_t line;
  uint32_t count;
  uint32_t first_frame;
  uint32_t last_frame;
  uint8_t  severity;
  uint8_t  truncated;      // set if the message did not fit
  char     file[96];       // project-relative path; tail kept if truncated
  char     exception[32];
  char     message[192];  // first occurrence only
};
// ~350 B x 64 records, about 22 KB. Overflow evicts the least recently seen entry
// and sets an "overflowed" flag in the report. Full stack traces and absolute paths
// are written to the events file on first occurrence and are not kept in the ring.
```

---

## 8. Verify plane

### 8.1 Input injection (decision C4)

**Where:** at the input-device level (`DEV_InputDevice`, `SCA_IInputDevice`), through the same path real events take. Keyboard sensors, mouse sensors and `bge.logic.keyboard` / `bge.logic.mouse` then observe injected input exactly like real input, with no logic changes.

**Format:** a flat array of POD events with a cursor, sorted by frame.

```cpp
// 16 bytes (four per cache line). Sorted by frame, consumed with a single cursor.
// No allocation after load.
struct CM_InjectedEvent {
  uint32_t frame;      // logic frame at which the event is applied
  uint16_t device;     // keyboard, mouse button, mouse motion, joystick button, ...
  uint16_t code;       // key code / button id / axis (X or Y) for mouse motion
  uint8_t  state;      // press / release; for motion: 0 = relative, 1 = absolute
  uint8_t  pad[3];
  int32_t  value;      // motion amount in pixels (relative) or position (absolute)
};
```

Applied at the start of the logic frame, before the input managers evaluate sensors:

```cpp
// With no scenario loaded, count == 0 and this is a single compare per frame.
while (cursor < count && events[cursor].frame <= current_frame) {
  ApplyEvent(events[cursor++]);
}
```

**Scenario API (player side):**

```python
from bge import agent
agent.input.press("W", frame=10)
agent.input.release("W", frame=90)
agent.input.tap("SPACE", frame=30, hold_frames=3)
agent.input.mouse_move(dx=12, dy=-4, frame=40)     # relative motion, in pixels
agent.input.mouse_set(x=640, y=360, frame=40)      # absolute position, in pixels
agent.input.mouse_press("LEFT", frame=50)
agent.input.mouse_release("LEFT", frame=52)
```

Events are accumulated in the scenario script, then compiled into the sorted array before frame 0 (or loaded from a recorded file). Registering an event whose frame has already passed raises an explicit `ScenarioError`, recorded as an error. All input is registered before frame 0 unless a dynamic mode is added later.

**Recording and repro:** every run writes the compiled input events to `runs/NNNN/input.json`. When a run ends in failure, the launcher also writes `repro.json` with the events up to the failing frame, the seed, the fixed rate, the engine commit, the snapshot id of the `.blend` and script folder, and the scenario path. Replaying it through the normal `--agent` flags reproduces the failure at the same frame. Minimizing the event list (delta debugging) is done agent-side.

**Properties:**
- With a fixed timestep, events indexed by frame make runs deterministic and reproducible.
- A recorded playthrough is a regression test.
- **Mouse (virtual pointer, C14):** in agent mode the canvas reads and writes a **virtual pointer position** instead of the operating system's cursor. Injected relative and absolute motion update that virtual state, and engine calls that set or recenter the cursor (for example the Mouse actuator's look mode) write to it without warping the real cursor, so the developer's own mouse is never touched. The mouse-focus sensor and `bge.logic.mouse.position` read the same position, so ray-cast focus and mouse look need no special cases. Outside agent mode the real-pointer path is unchanged; the cost is one flag check on the few per-frame pointer queries.

**Phase 0 must verify** how the device updates key state (just-activated / active / just-released) so injected events produce identical state transitions to real ones, and where in the frame order injection must happen. It must also verify where the pointer position is read and written (canvas layer, mouse manager, Mouse actuator recentering, mouse-focus sensor), so the virtual pointer replaces every path that would otherwise touch the real cursor.

### 8.2 Gameplay assertions (decision C5)

Plain Python callbacks registered for specific frames. No declarative DSL: Python already expresses any condition.

**Execution point (C11):** callbacks run once per logic frame **after logic and physics have completed and before render**. An assertion at frame N therefore sees the state produced by frame N's logic and physics. Input events for frame N are applied before the input managers evaluate sensors (section 8.1), so input and assertion for the same frame are ordered: input → logic/physics → assertions → render.

Callbacks and `expect_prop` take an optional `scene=` argument; the default is the scene active at registration.

```python
from bge import agent, logic

scene = logic.getCurrentScene()

@agent.at_frame(120)
def player_moves_right():
    x = scene.objects["Player"].worldPosition.x
    agent.expect(x > 1.0, f"x={x:.2f}, expected > 1.0")

@agent.at_end()
def coins_collected():
    agent.expect_prop("GameState", "coins", equals=5)
```

| Function | Purpose |
|---|---|
| `agent.at_frame(n)` | Run callback once at frame n |
| `agent.every(n, start=0, stop=None)` | Periodic check (explicit opt-in, not default) |
| `agent.at_end()` | Run at scenario end |
| `agent.expect(cond, msg)` | Record pass/fail |
| `agent.expect_prop(obj, prop, equals/lt/gt=..., scene=None)` | Common object-property check |
| `agent.abort(reason)` | End the run now. Status is derived from recorded assertions and errors; `exit_reason` is `abort` |
| `agent.screenshot_at(n, name)` | Capture for multimodal inspection |

**Cost:** callbacks live in a frame-sorted list with a cursor. Frames with no registered callback pay one compare. An exception inside a callback becomes a failed assertion with a traceback, not an engine error.

### 8.3 Visual verification

- `agent.screenshot_at(frame)` reuses the existing screenshot path (canvas). Readback stalls the pipeline, so it is used only for verification, never inside benchmark or profile windows.
- `agent.image_diff(a, b)` returns max abs diff, mean abs diff and PSNR.
- Deterministic only with fixed dt, fixed seed and identical resolution.
- GPU-side frame hashing is deferred until readback cost is measured to matter.

### 8.4 Test design guidance for agents

An agent that writes both the game and the only test of it can pass trivially. For the POC, a **held-out test** (written by the project owner, hidden from the agent) decides success. In practice the agent also writes its own assertions for self-checking, but the verdict uses the hidden suite.

---

## 9. Measure plane (profiling, benchmark, compare)

Unchanged in intent from v2. Moves to Phase 4 because the vision prioritizes the closed loop.

### 9.1 Profiling rules
1. Off by default. Instrumentation compiles to `if (active)`.
2. Flat scope table with an enum; self time derived offline from a static parent table.
3. All statistics are computed at report time from preallocated arrays.
4. Granularity floor: engine stage, scene, optionally Python script. Never per vertex, bone or triangle.
5. Benchmark runs record frame time only. Scope profiling is a separate diagnostic run (observer effect).

### 9.2 Profiler data structure (decision D2)

| Option | Cost on | Cost off | Granularity | Notes |
|---|---|---|---|---|
| **A. New flat scope table** | ~50 ns/scope | 1 branch | Stage, scene, script | Enum IDs, `uint64_t ticks[N]`, static parent table |
| B. Extend `KX_TimeCategoryLogger` | Existing | Existing | Stage only | Verify internals in Phase 0. Fine for ~10 categories at frame granularity, not for per-script scopes |
| C. Optional Tracy zones (`WITH_TRACY`) | Zero if not compiled | Zero | Excellent, includes GPU | External dependency; capture-file output needs an exporter for agents |

**Decision (C17):** B supplies the top-level stage numbers, so they match what UPBGE already shows; A covers only what B cannot (scripts, sub-stages). C can be added later behind a compile flag using the same macro.

The parent table must be a **tree** (every scope has exactly one parent), otherwise derived self time is ambiguous. A compile-time check validates this. A scope reachable from two callers is modeled as two scopes, as `PythonControllers` and `PythonComponents` already are.

```cpp
// Scope IDs are dense small integers, indexed without hashing. Keep Count <= 30 so a
// frame sample (below) stays within two cache lines.
enum class CM_Scope : uint8_t {
  Frame, Logic, Sensors, Controllers, PythonControllers, PythonComponents,
  Actuators, Physics, Animation, SceneGraph, Culling, Render, Network, Count
};

struct alignas(64) CM_ProfileState {
  bool active = false;                        // the only read when profiling is off
  uint64_t ticks[(int)CM_Scope::Count] = {};  // inclusive ticks, current frame
};
extern CM_ProfileState g_cm_profile;

// Inclusive timing. Self time is derived offline from the parent table, so the
// recording path never maintains a scope stack.
struct CM_ScopeTimer {
  CM_Scope id;
  uint64_t t0;
  explicit CM_ScopeTimer(CM_Scope s) : id(s), t0(g_cm_profile.active ? CM_ReadTicks() : 0) {}
  ~CM_ScopeTimer() { if (t0) g_cm_profile.ticks[(int)id] += CM_ReadTicks() - t0; }
};
```

```cpp
// One sample per frame. 30 floats + frame_ms + frame_index = 128 B = exactly 2 cache
// lines, so a report scan streams linearly and a record touches exactly two lines.
struct alignas(64) CM_FrameSample {
  float ms[30];
  float frame_ms;
  uint32_t frame_index;
};
// Ring capacity is a power of two (wrap by mask). Allocated only on profile.start().
```

### 9.3 GPU timing (decision D3)

| Option | Stall | Notes |
|---|---|---|
| **A. GL timer queries, ring of 3–4, read N frames late** | None | Standard technique. Results delayed a few frames |
| B. `glFinish` brackets | Full pipeline stall | Rejected: destroys CPU/GPU overlap and changes what is measured |
| C. CPU submit time only | None | Fallback, reported as `gpu.available = false` |

**Decision (C18):** timer queries are added to the Blender GPU module so they are backend-agnostic. This is the first candidate exception under the gate (section 14). Shape: a small additive API (begin/end/poll over a ring of queries) with a GL backend implementation; other backends (for example Metal, Vulkan) report "unavailable", which surfaces as `gpu.available = false`. Before any code, Phase 0 checks whether the GPU module or the draw manager already exposes a reusable timer facility; if one fits, nothing in Blender is touched. No query objects exist until `profile.start()`, so the cost outside agent mode is zero. Granularity is per pass (shadows, geometry, lighting, post), never per draw call; on tile-based GPUs timestamps are typically available per pass or encoder only. CPU and GPU times overlap and are never summed.

### 9.4 Per-script Python cost (decision D5)

| Option | Overhead | Notes |
|---|---|---|
| **A. Clock pair around the call site in C++** | ~50 ns/call | Diagnostic runs, indexed by an interned script ID |
| B. `cProfile` / `setprofile` | 2x or worse | Dedicated deep dive only, report flagged `distorted: true` |

### 9.5 Pause and step (decision D6)

| Option | Behavior | Risk |
|---|---|---|
| A. Suspend scenes | Logic and physics stop, rendering continues | Clock, input, network, 2D filters may keep running |
| **B. Timescale 0 plus launcher-driven fixed step** | Deterministic: advance exactly N frames at fixed dt | Launcher must own loop iteration |

Recommendation: B for stepping and verification, A for interactive pause. Phase 0 documents what each actually freezes.

### 9.6 Benchmark methodology

- Vsync off, cap lifted, fixed dt, fixed seed, fixed resolution. Warmup discarded.
- Environment block stored with every result (GPU, driver, renderer string, OS, resolution, build type, physics engine, **engine commit**). `compare` refuses mismatches.
- Run parameters stored with every result (seed, fixed rate, resolution, frames, warmup, trials). `compare` also refuses mismatches here, not only on the environment.
- GPU adapter (integrated vs dedicated, name, driver) is part of the environment block. `compare` refuses mismatches.
- Benchmark numbers come only from the standalone player (C20).
- System load is sampled before and during the run. Results measured under high load carry `noisy_environment: true`, and `compare` downgrades them to `INCONCLUSIVE`.
- Benchmarks should be run with the machine otherwise idle. The launcher reports this requirement; it does not try to enforce it.
- **Thermal soak:** before the first measured trial the launcher renders the scene without measuring for a configurable time (default about 20 s, `--soak-seconds`), so CPU and GPU boost state is not measured while cold. This is separate from the in-engine warmup (shader compilation, caches).
- **Interleaved trials:** when comparing two project versions, the launcher alternates processes A-B-A-B instead of running all baseline trials and then all candidate trials. Clock boost and temperature drift would otherwise bias the later group (a cold baseline against a hot candidate looks like a regression). Each trial is its own process, so interleaving needs no engine support.
- **Drift check:** if trial medians trend monotonically within a group, or inter-trial variance exceeds a threshold, the result is marked `INCONCLUSIVE`.
- Samples in a preallocated `float` array. Percentiles computed at the end with `nth_element` on a copy.
- At least 5 trials per configuration. Compare sets of trial medians: `significant = true` only if the ranges do not overlap and the change is at least 1%. Otherwise `INCONCLUSIVE`, and the agent must not keep the change.
- Goodhart guard: environment match, visual diff, and object/light/triangle counts included in `compare` so lowering resolution or deleting content is flagged.

```python
bge.agent.profile.start(scopes="engine")   # "engine" | "engine+python" | "full"
bge.agent.profile.stop()
bge.agent.profile.frame()
bge.agent.profile.report(top=10, min_ms=0.05)
bge.agent.benchmark(frames=300, warmup=100, trials=5)
bge.agent.compare(baseline, candidate, visual=None)   # IMPROVED | REGRESSED | INCONCLUSIVE | INVALID
```

Budgets (estimates, to be measured in Phase 0): scope guard off ≈ one predicted branch; on ≈ 50 ns; frame ring 128 KB; error ring ≈ 22 KB; benchmark buffer `frames × 4 B`.

---

## 10. Phases

### Phase 0 — Audit (no code)

| Area | Where | Question |
|---|---|---|
| Input path | `DEV_InputDevice.*`, `SCA_IInputDevice.*`, `SCA_KeyboardManager.*`, `SCA_MouseManager.*` | How are key states transitioned per frame? Where must injection occur? |
| Main loop | `LA_Launcher.cpp`, `KX_KetsjiEngine.cpp` | Safe points to inject input, run frame-indexed callbacks, sample frame time |
| Clock | `CM_Clock.*`, engine timescale/tickrate fields | Force fixed dt and ignore wall clock |
| CLI | `LA_SystemCommandLine.*`, `GPG_ghost.cpp` | Where `--agent` and related flags are parsed in the standalone player (embedded runtime is out of scope, C20) |
| Python surface | `KX_PythonInit.cpp`, `KX_PythonInitTypes.*` | Register `bge.agent`; reuse existing screenshot/profile functions |
| Error sites | `EXP_PythonCallBack.cpp`, `SCA_PythonController.cpp`, `KX_PythonComponent.cpp`, `KX_PythonProxy.cpp` | All `PyErr_Print` sites |
| Screenshot | `KX_BlenderCanvas.*`, `RAS_ICanvas.*` | When it executes, readback cost |
| Engine version pin | Repository | Record the HEAD of the main branch as the pinned engine commit when Phase 0 starts (C13); it moves only deliberately |
| `bpy` operation matrix | Headless script, pinned commit | For every operation the helper needs: `bpy.data` path or `bpy.ops` only, and whether `poll()` passes headless (with `temp_override`). Covers logic bricks, components, runtime properties, importers, render/game settings and physics |
| Script resolution | Player load path, `bpy.data.texts` | Resolution order between internal text datablocks and the external script folder; behavior on name collision |
| Edit batch startup | Measure `blender -b` cold/warm load on a mid-size `.blend`, on Windows with antivirus active | Real fixed cost per edit batch |
| Stage timing | `KX_TimeCategoryLogger.*`, `KX_TimeLogger.*` | Internal containers and categories; cost of reading them every frame; can they feed stage numbers directly (C17)? |
| GPU timer facility | `source/blender/gpu`, draw manager profiling (read only) | Does a reusable timer-query facility already exist? If yes, no Blender change (C18, gate) |
| Install layout | Gameengine CMake, release package layout per platform | Does an install rule inside gameengine place `python/` correctly? Where is Blender's bundled Python in each layout? |
| Pause semantics | `KX_Scene.*`, `SCA_LogicManager.*` | What freezes under suspend vs timescale 0 |
| Render stats | `RAS_Rasterizer.*`, `RAS_BucketManager.*`, `eevee_game/` | Existing counters |
| Object lookup | `EXP_BaseListValue.*` | Linear name lookup? |
| Clock cost | Microbenchmark per OS | Cost of `CM_ReadTicks` |
| Window behavior | `GPG_ghost.cpp`, `GPG_Canvas.*`, windowing layer (read only) | Can the player open at fixed geometry without taking focus? Is rendering unaffected when partially occluded? |
| Pointer path | `RAS_ICanvas.*`, `KX_BlenderCanvas.*`, `GPG_Canvas.*`, `SCA_MouseManager.*`, `SCA_MouseActuator.*`, `SCA_MouseFocusSensor.*` | Where is the pointer read and written? What does mouse-look recentering call? Can a virtual pointer replace it in agent mode? |
| Determinism audit | Target platforms | Same scenario run 20 times with a state hash every N frames: verify reproducibility empirically before promising it |

Exit criterion: table "capability → exists / needs C++ / needs Python only", the **exact hook sites** (input injection, pointer virtualization, assertion callbacks, frame-time sampling, error reporting, heartbeat), the pinned engine commit, measured clock cost, edit-batch startup cost and the window-behavior result.

### Phase 1 — Run and observe
- CLI flags (`--fixed-hz`, `--seed`, `--frames`, `--out`, `--events`, `--heartbeat`, `--fail-fast`), exit-code table, `PYTHONHASHSEED`, Python `random` seeding in the scenario bootstrap, per-iteration run directories.
- `bge.agent`: `project_info`, `list_scenes`, `inspect_scene`, `inspect_object`, `stats`, `errors`, `screenshot`.
- Error ring and event log (D4-A).
- `WITH_AGENT_API` CMake option and the `source/gameengine/Agent/` skeleton with its install rule (C19, C22).
- Agent-side launcher script (heartbeat monitor, total timeout, process-tree kill, status classification from exit code).
- Window options (`--window-size`, `--window-pos`) and Windows process-tree handling tested on a desktop session.
- Exit criterion: a script runs N deterministic frames and returns a valid `run_result.json`, on Windows, Linux and macOS (C21). Idle overhead with the feature compiled in but unused is not measurable above noise.

### Phase 2 — Edit and verify
- `upbge_agent_edit` helper module (including `Import`) and `edit.py` runner with snapshots of `.blend` plus scripts.
- Input injection (keyboard, mouse buttons, mouse motion with the virtual pointer) and `at_frame` / `expect` assertions.
- Exit criterion: the agent can create a scene with a controllable object, press keys, move the virtual mouse (mouse look) and assert that it moved and turned, with no human interaction and without touching the developer's real cursor.

### Phase 3 — Closed-loop POC (section 12.1)

### Phase 4 — Measure
- Profile, benchmark, compare, GPU pass timing through the GPU-module timer query (C18).
- Exit criterion: profiling on stays within the documented budget on the POC scene; profiling off is indistinguishable from baseline.

### Phase 5 — Logic introspection and adapters
- `inspect_logic` (read only), optional MCP/CLI adapter as an external process calling the same files/CLI interface.

---

## 11. Metrics

| Group | Metrics |
|---|---|
| Loop | Iterations to pass hidden tests, edit failures, crashes, timeouts, reverts, wall-clock per iteration |
| Agent | Context consumed per iteration, response sizes of `inspect_*` |
| Verification | Assertion pass rate, false passes (hidden test fails while own test passes) |
| Performance | Bottleneck identification rate, improvement, false-positive keeps, regressions |
| Engine overhead | Frame time with feature unused vs baseline build; profiling-on overhead |

---

## 12. Proofs of concept

### 12.1 POC-1: build a small game from a spec

Spec given to the agent (example): *"Top-down scene. The player moves with WASD. Five coins are placed in the scene; touching a coin removes it and increments `coins`. When all five are collected, a door opens. Reaching the door ends the game with `won = True`."*

- The agent has only: the edit helper, the run launcher, and `bge.agent`.
- A **hidden test** (project owner's scenario with recorded input) decides success.
- Success criteria: hidden test passes; no errors in the run; no hand editing; iterations and failures recorded.
- Variants: tighter spec, ambiguous spec (does the agent ask or assume), spec that requires fixing a planted bug in an existing project.

### 12.2 POC-2: performance with planted bottlenecks

| Scenario | Planted cause | Expected diagnosis |
|---|---|---|
| S1 | Python controller with an O(n²) loop over 2,000 objects | Python scope dominates, script ranked first |
| S2 | 10,000 moving objects with heavy per-object logic bricks | Logic scope dominates |
| S3 | Many shadow-casting lights | GPU shadows dominate |
| S4 | Thousands of unique materials | Draw calls and material switches dominate |
| S5 | Large static scene, culling ineffective | Culling and visible counts dominate |
| S6 | Balanced control scene | Agent reports no clear bottleneck and changes nothing |

Success: correct diagnosis per scenario, change kept only on `IMPROVED` with visual gate passing, no keeps on `INCONCLUSIVE`.

---

## 13. Risks

| Risk | Mitigation |
|---|---|
| Agent passes its own weak tests | Hidden test suite decides success |
| Hung or crashing player | Engine watchdog, external kill, incremental event log |
| Headless edit latency | Batching; persistent `bpy` worker behind the same helper API if needed |
| Binary `.blend` makes diffs unreadable | Snapshots per batch; edit results are structured JSON |
| Nondeterminism breaks verification | Contract bounded to same machine and build (6.4); fixed rate, seed, `PYTHONHASHSEED`; input indexed by frame; Phase 0 audits input and clock paths |
| A single hung frame cannot be interrupted from inside the engine | Heartbeat plus external kill (C12) |
| The agent run disturbs the developer (focus, GPU load), or the developer disturbs the run | Device-level input injection, fixed window geometry, load sampling with the `noisy_environment` flag |
| Dual-GPU laptops give different results per run | Adapter recorded in the environment block; `compare` refuses mismatches |
| Windows process and path edge cases (orphan processes, spaces, long paths, antivirus latency) | Job Objects, path handling tests, startup cost measured in Phase 0 |
| Capability matrix drifts as the engine changes | Pinned engine commit (C13), re-audited when the pin moves |
| Internal text datablocks shadow agent-written external scripts | Script policy C15 and a per-batch check in the helper |
| `bpy.ops` fails in a background context | Single wrapper with `temp_override` and structured errors (C16); operation matrix in Phase 0 |
| Engine-initiated cursor warps move the developer's real cursor | Virtual pointer in agent mode (C14) |
| Instrumentation perturbs results | Off by default; separate diagnostic and benchmark runs |
| Benchmark noise yields fake wins | Trials, significance rule, environment block |
| Agent games the metric | Visual gate, count checks, environment match |
| Agent-written Python is arbitrary code | Run in a working directory dedicated to the agent; sandboxing is the caller's responsibility, not an engine feature |
| Output too large for agent context | Pagination, field selection, byte budgets, summaries first |
| API drift breaks agents | `schema_version` in every payload, additive changes only |
| Embedded vs standalone runtime differences | Phase 0 verifies both launch paths and documents unsupported combinations |

---

## 14. Gate for touching Blender code

Blender code outside `gameengine/` and `eevee_game/` is changed only if **all** hold:

1. A specific agent capability is required and the Phase 0 audit shows it cannot be achieved through `bpy`, the CLI, file edits or the `gameengine/` module.
2. The change is **additive** (a new RNA property or operator), not a modification of existing behavior.
3. The diff is minimal and isolated in its own commit, with the audit finding referenced.
4. It has no cost on any existing code path.

Current state: **one candidate identified**: GPU timer queries in the Blender GPU module (C18). It passes the gate only if Phase 0 confirms that no existing facility fits, and only in the form described in section 9.3 (additive, isolated, a single commit, no cost outside agent mode). The audit may add items; each one must pass this gate individually.

---

## 15. Deferred / dropped

Dropped: engine transactions, Python server thread, `sys.stderr` parsing, `glFinish` GPU timing, declarative assertion DSL.
Deferred: persistent `bpy` worker, interactive socket control, MCP adapter, GPU-side frame hashing, Tracy integration, headless GL / CI support (not required for the target audience), asset generation, semantic project graph.

---

## 16. Open questions

All questions raised in earlier versions are closed (sections 0.3 and 0.4). What remains is answered by Phase 0 audits, not by decision:

1. Does the GPU module or the draw manager already expose a reusable timer-query facility (C18)?
2. Does an install rule inside the gameengine CMake suffice for the release layout, and where is Blender's bundled Python on each platform (C22)?
3. What exactly freezes under scene suspend versus timescale 0 (D6)?
4. Can the player open at fixed geometry without taking focus on each OS?
5. Is the pointer path fully virtualizable (C14)?
6. Which edit operations exist only as `bpy.ops`, and do they pass `poll()` headless (C16)?

---

## 17. Guiding principle

UPBGE stays unintelligent and becomes **editable, runnable, observable, verifiable, measurable and repeatable**. The cost of those properties must be invisible when unused and bounded when used. The intelligence lives outside the engine.
