# KX_SaveSystem — Architecture

**Revision 2** · 2026-10-22 · Supersedes revision 1

The save system lives in `source/gameengine`. The only Blender-side change is the Game actuator extension in §13: two DNA defines, RNA enum items and properties, and one UI line. No DNA struct layout changes.

---

## 0. What changed from revision 1

| Area | Revision 1 | Revision 2 | Reason |
|---|---|---|---|
| Container | `nlohmann::json` + msgpack-c | `nlohmann::json` only, using its built-in `to_msgpack` / `from_msgpack` | One library instead of two. nlohmann 3.11.3 is already in the tree (`extern/json`, CMake target `bf::extern::json`). |
| Compression | zlib level 6 | zstd level 3 with a content checksum | 12× faster at the same ratio (§2). zstd is already a required Blender dependency. |
| Integrity | Adler32 on the uncompressed data | zstd frame checksum (XXH64) | Verified by zstd during decompression. Adler32 was also redundant with the checksum inside the zlib stream. |
| Encryption | ChaCha8, encrypted *before* compression, nonce fixed at zero | None. One flag bit and 12 nonce bytes stay reserved in the header. | Ciphertext cannot be compressed, so rev 1's compression did nothing. The fixed nonce reused the keystream. Godot, Unity and Unreal do not encrypt saves by default. |
| Threading | Fully synchronous on the main thread | Snapshot on the main thread; compression, write and fsync on a serial background pool | fsync latency is unbounded and cannot be time-sliced (§5.6). |
| Files per slot | `save.msgpack` + `metadata.json` + `thumbnail.png` (+ debug `save.json`) | One `<slot>.upsav` file | Two files cannot be replaced atomically together. Listing a slot now reads a few hundred bytes. |
| Atomic write | `std::ofstream` + `flush()` + `BLI_rename` | `BLI_open` → write → fsync / `_commit` → `BLI_rename_overwrite` → directory fsync | `BLI_rename` refuses an existing target, so in rev 1 every overwrite failed. `flush()` is not durable. `ofstream` breaks on non-ASCII paths on Windows. |
| Save location | `//saves/` (next to the .blend) | Per-user data directory | Program Files, signed macOS bundles and APKs are read-only. |
| Migrations | Exact `"1.0.0"→"2.0.0"` string pairs, each copying the whole tree | Integer `schema_version`; migrations run v→v+1 in a flat loop, in place | String pairs do not chain, and lexicographic order puts `"1.10"` before `"1.9"`. |
| Lifetime | Process-wide singleton | Member of `KX_KetsjiEngine` | The embedded player restarts the engine within one Blender session. A singleton would keep callbacks from the previous run. |
| Errors | Global `GetLastErrorType()` | Returned per call, plus a per-slot job status | Global error state cannot work once jobs complete on another thread. |
| Partial save/load | `partial_keys` | Removed | With one compressed stream, a partial load still decompresses and parses everything. A partial save either loses the other keys or must load, merge and rewrite the file. |
| Scene state | Not handled | Built-in scene snapshot stored as flat SoA arrays (§8) | This is the part a game cannot do well from script. |
| Debug JSON | Written next to every save | Offline dump tool | Avoids doubling the I/O of every save. |
| Logic bricks | Save/load modes on the Action actuator; a "Save Exists" property sensor | *Save Slot* / *Load Slot* modes on the Game actuator (§13). Existence checks use the in-memory slot table. | The Action actuator plays animations; the Game actuator already hosts the globalDict save/load modes. A sensor that checks the file system would `stat()` every tick. |
| Big-endian path | Byte swapping | `static_assert` little-endian | Every supported target (x86-64, ARM64) is little-endian. |

---

## 1. Goals and non-goals

**Goals**

1. **No frame hitch caused by I/O.** The main thread pays only for the snapshot. Compression, file writes and fsync never run on it.
2. **Python-free core.** Everything except the `bge.save` module builds without `WITH_PYTHON` (needed for Android).
3. **Crash safety.** A power loss or crash at any point leaves either the previous save or the new one, never a torn file.
4. **Bounded load on untrusted files.** A malformed or malicious save fails with an error code. It can never crash the game or force a huge allocation. Saves are often shared between players.
5. **Game-engine-only changes.** No modifications outside `source/gameengine`, except the small Game actuator extension in §13.

**Non-goals**

- Protecting saves from a determined player. The format is not encrypted. Integrity checks detect corruption, not deliberate edits.
- Bit-exact physics resumption. Bullet's contact cache, sleep state and solver warm-start data are not saved. Restored bodies resume from their transform and velocity.
- Cloud sync and save versioning (several versions per slot). Both can be built on top later.

---

## 2. Measurements that drive the design

**Compression.** The test payload is 1.26 MiB, structured like the §8 snapshot: 10,000 objects as SoA arrays plus three properties each, plus a small key-value user section. Timings are single-threaded on a 2.1 GHz Xeon vCPU, best of 20 runs. Expect phones to be slower in absolute terms; the ratios between codecs carry over.

| Codec | Compress | Throughput | Ratio | Decompress |
|---|---|---|---|---|
| zlib-1 | 12.8 ms | 101 MB/s | 3.57× | — |
| zlib-6 (rev 1 default) | 45.3 ms | 28 MB/s | 4.02× | 3.55 ms |
| zstd-1 | 2.85 ms | 453 MB/s | 3.88× | — |
| **zstd-3 (rev 2 default)** | **3.75 ms** | **344 MB/s** | **4.07×** | **1.19 ms** |

Rev 1 estimated 500 KB raw for 10,000 objects. With transforms, velocities, state and three properties per object, the real figure is about 1.26 MiB.

**Parser recursion.** nlohmann's msgpack reader is recursive. A file made of nested one-element arrays overflows the stack at:

| Stack size | Nesting that crashes | File size that crashes |
|---|---|---|
| 1 MiB (Windows main-thread default) | ~6,300 levels | ~6.3 KB |
| 8 MiB | ~52,000 levels | ~52 KB |

Passing `allow_exceptions=false` does not help: the overflow happens before any error path is reached. §6.6 adds a depth guard.

---

## 3. Overview

```
MAIN THREAD (frame N, logic step)                    BACKGROUND SERIAL POOL
──────────────────────────────────                   ──────────────────────
bge.save.save() / KX_SaveSystem::Save()
  ├─ user data   → SaveData (json tree)
  ├─ scene       → SceneSnapshot (SoA byte blobs)
  └─ metadata    → small json map
        │
MAIN THREAD (frame N, end of frame, after EndFrame)
  ├─ thumbnail readback (same point as FlushScreenshots)
  ├─ json::to_msgpack(payload) → reused byte buffer
  └─ BLI_task_pool_push(job) ───────────────────────► zstd payload (reused CCtx)
                                                     downscale + zstd thumbnail
                                                     assemble header|meta|thumb|payload
                                                     write tmp → fsync → rename → dir fsync
                                                     job->state.store(DONE, release)
MAIN THREAD (frame N+k, NextFrame)                           │
  └─ Poll(): job->state.load(acquire) ◄──────────────────────┘
       update slot table, recycle buffers, expose status
```

---

## 4. Ownership and lifetime

- `KX_SaveSystem` is a plain member of `KX_KetsjiEngine`, reached through `KX_GetActiveEngine()->GetSaveSystem()`. It has no static state.
- It is created with the engine and drained in `KX_KetsjiEngine::StopEngine()`. The drain sits next to the existing `m_converter->FinalizeAsyncLoads()`, which is the same pattern already used for async LibLoad.
- The embedded player loops on `RESTART_GAME` / `START_OTHER_GAME` (`BL_KetsjiEmbedStart.cpp`). Each run therefore starts with an empty migration table, an empty slot table and no pending jobs.

---

## 5. Threading model

### 5.1 Main-thread work (the only part that costs frame time)

| Step | When | Cost |
|---|---|---|
| Scene capture (§8.5) | At the `Save()` call, during logic | One pass over `GetObjectList()`; dominated by pointer chasing |
| User data to tree | At the `Save()` call | Proportional to the number of nodes in the user tree |
| Thumbnail readback | End of the same frame, where `RAS_ICanvas::FlushScreenshots()` already runs, via `RAS_Rasterizer::MakeScreenshot()` | One synchronous GPU→CPU read. It can be disabled per save. |
| `json::to_msgpack` | End of frame, just before the job is pushed | Linear in the tree. Scene arrays are `json::binary` nodes, so each one costs a single bulk copy. |

The scene is captured during the logic step that asked for the save, so it reflects that step. The thumbnail shows the frame rendered from that same state.

### 5.2 Worker work

Jobs run on a pool created with `BLI_task_pool_create_background_serial()`. Each job:

1. Compresses the payload with zstd level 3 (`ZSTD_c_checksumFlag = 1`, content size on). It writes straight into the output buffer at its final offset. The output buffer is sized with `ZSTD_compressBound`, so the payload is never copied a second time.
2. Downscales the thumbnail to at most 320 px wide with a box filter, then compresses it with zstd.
3. Fills in the header and metadata in front of the payload, giving one contiguous buffer and one `write()`.
4. Performs the durable atomic write described in §7.2.
5. Stores the result code, then does `state.store(Done, std::memory_order_release)`.

The serial pool means only one job runs at a time. The pool therefore owns one `ZSTD_CCtx` and its scratch buffers outright, with no locking.

### 5.3 Hand-off

- A job is a heap struct holding its byte buffers (moved in), the slot name, and `std::atomic<uint8_t> state`.
- Between `BLI_task_pool_push` and the release store, only the worker touches the job. Before the push and after the acquire load, only the main thread does. There is no mutex anywhere.
- `KX_KetsjiEngine::NextFrame()` calls `KX_SaveSystem::Poll()` once per frame. Poll does one acquire load per job still in flight, which in practice means zero to two jobs.
- When a job finishes, its buffers go back to a two-entry free list. In steady state, saving allocates no new I/O buffers.

### 5.4 Ordering

The serial pool runs jobs in submission order. Two saves to the same slot therefore complete in order, and the last one requested is the one on disk. Pending jobs are not coalesced; saves are rare enough that it would add complexity for no measurable gain.

### 5.5 Shutdown

`StopEngine()` calls `BLI_task_pool_work_and_wait()` and then `Poll()` once. A save requested on the frame the game quits is completed, never dropped.

### 5.6 Why a thread is justified here

The CPU part (3.75 ms of zstd-3 for 1.26 MiB) could arguably stay on the main thread. The fsync cannot. It blocks on the storage device, its latency has no upper bound, and it cannot be split across frames. On Android flash or a busy HDD it routinely takes tens of milliseconds.

The synchronisation cost is one release store and one acquire load per save. Unreal's `AsyncSaveGameToSlot` uses the same split: serialise on the game thread, write on a worker.

### 5.7 Load path

`Load()` reads and decompresses synchronously on the main thread:

- Applying scene state has to happen on the main thread anyway.
- Reading about 300 KiB and decompressing it in roughly 1.2 ms is acceptable during a loading screen.

Scene application is deferred to the next frame boundary (§8.6). An asynchronous prefetch is listed in §15.

---

## 6. File format

### 6.1 Layout

```
offset 0                  header_size            +meta_size          +thumb_size
┌────────────────────────┬──────────────────────┬───────────────────┬──────────────────────┐
│ SaveFileHeader (48 B)  │ Metadata             │ Thumbnail         │ Payload              │
│                        │ msgpack, uncompressed│ zstd frame, opt.  │ zstd frame + XXH64   │
└────────────────────────┴──────────────────────┴───────────────────┴──────────────────────┘
```

Listing a slot reads `header_size + meta_size` bytes. A load menu reads the thumbnail on demand at a known offset. Only `Load()` touches the payload.

### 6.2 Header

```cpp
#include <bit>
#include <cstddef>
#include <cstdint>

// All supported targets are little-endian; the header is written and read verbatim.
static_assert(std::endian::native == std::endian::little, "save format assumes little-endian");

enum SaveFileFlag : uint32_t {
  SAVE_FLAG_PAYLOAD_ZSTD = 1u << 0, /* Always set by v1 writers. */
  SAVE_FLAG_ENCRYPTED = 1u << 1,    /* Reserved: never written by v1, rejected by v1 readers. */
  SAVE_FLAG_HAS_SCENE = 1u << 2,    /* Payload contains a "scene" section. */
};

struct SaveFileHeader {
  char magic[4];            // "UPBG" as raw bytes so it reads correctly in a hex dump
  uint16_t format_version;  // 1; bumped only for incompatible layout changes
  uint16_t header_size;     // newer writers may append fields; readers skip what they don't know
  uint64_t raw_size;        // decompressed payload size: one exact allocation, capped before use
  uint32_t flags;           // SaveFileFlag; unknown bits are rejected
  uint32_t schema_version;  // game-defined integer that drives the v -> v+1 migration loop
  uint32_t meta_size;       // bytes of the metadata block right after the header
  uint32_t thumb_size;      // bytes of the compressed thumbnail; 0 means none
  uint32_t payload_size;    // bytes of the compressed payload; file must end exactly here
  uint8_t nonce[12];        // reserved for SAVE_FLAG_ENCRYPTED; zero in v1
};
// raw_size sits at offset 8 so it is naturally aligned without padding words.
static_assert(sizeof(SaveFileHeader) == 48, "on-disk layout must not depend on compiler padding");
static_assert(offsetof(SaveFileHeader, raw_size) == 8, "64-bit field must stay naturally aligned");
static_assert(offsetof(SaveFileHeader, nonce) == 36, "nonce offset is part of the format");
```

The 12-byte nonce and `SAVE_FLAG_ENCRYPTED` exist so that an opt-in encryption mode can be added later without changing `format_version`. Such a mode would compress first, then encrypt, use a random nonce per save and add a MAC.

### 6.3 Metadata block

A msgpack map. It stays uncompressed so that listing slots never touches zstd.

| Key | Type | Source |
|---|---|---|
| `engine` | str | UPBGE version string |
| `game_version` | str | `SetGameVersion()` (informational only; migrations use `schema_version`) |
| `timestamp` | int | Unix seconds at the time of the `Save()` call |
| `scene` | str | Name of the active scene |
| `thumb_w`, `thumb_h` | int | Thumbnail size; absent when there is no thumbnail |
| `custom` | map | Game-provided key-value pairs (chapter, playtime, difficulty…) |

### 6.4 Payload (after decompression)

A msgpack map `{ "user": <SaveData tree>, "scene": <snapshot section, §8.4> }`. The `scene` key is present only when `SAVE_FLAG_HAS_SCENE` is set.

### 6.5 Validation order on read (cheap checks first)

1. File size ≥ 48, `magic == "UPBG"`, `format_version == 1`, `48 ≤ header_size ≤ file size`.
2. No unknown flag bits. `SAVE_FLAG_ENCRYPTED` → `SaveError::Unsupported`. `SAVE_FLAG_PAYLOAD_ZSTD` must be set.
3. `header_size + meta_size + thumb_size + payload_size == file size`, computed in 64 bits. A mismatch means the file is truncated or has trailing bytes → `Corrupted`.
4. Caps checked before any allocation:
   - `meta_size ≤ 64 KiB`
   - `thumb_size ≤ 4 MiB`
   - `raw_size ≤ max_payload` (default 256 MiB, configurable)
5. `schema_version ≤ current` → otherwise `VersionTooNew`. A save is never downgraded.
6. *Slot listing stops here and parses only the metadata.*
7. `ZSTD_getFrameContentSize(payload) == raw_size`, then `ZSTD_decompressDCtx` must return exactly `raw_size`. zstd verifies the XXH64 content checksum itself.
8. Depth guard (§6.6), then `json::from_msgpack(first, last, /*strict*/ true, /*allow_exceptions*/ false)`. If the result `is_discarded()` → `Corrupted`.
9. Scene section bounds checks (§8.4) before any of it is applied.

### 6.6 Depth guard

Before `from_msgpack`, run one `json::sax_parse(..., input_format_t::msgpack)` pass with a handler whose `start_object` / `start_array` return `false` once depth exceeds 64. The reader stops before it recurses deeper.

This was tested: 1,000,000 nesting levels under a 1 MiB stack are rejected cleanly. The guard is one extra linear pass with no DOM allocation. It covers both the metadata and the payload.

### 6.7 Slot names and file names

- Slot names must match `[A-Za-z0-9_-]{1,63}`. Anything else → `InvalidSlotName`. This closes path traversal (`"../../x"`). Windows device names (`CON`, `PRN`, `AUX`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9`, any case) are rejected explicitly on every platform, so a save made on Linux still loads on Windows. The 63-character limit makes every valid name fit in the Game actuator's existing `filename[64]` field (§13).
- Files are `<root>/<slot>.upsav`. The temporary file is `<root>/<slot>.upsav.tmp`.
- The first slot scan deletes any leftover `.tmp` file. Such a file only exists if a write was interrupted, and the previous `.upsav` is still intact.

---

## 7. Storage location and durable writes

### 7.1 Save root

| Platform | Default root | Built from |
|---|---|---|
| Windows | `%APPDATA%\<Game>\saves` | `BLI_getenv("APPDATA")` |
| macOS | `~/Library/Application Support/<Game>/saves` | `BLI_dir_home()` |
| Linux | `$XDG_DATA_HOME/<Game>/saves`, otherwise `~/.local/share/<Game>/saves` | `BLI_getenv`, `BLI_dir_home()` |
| Android | App-internal files directory | Supplied by the Android launcher (§15) |

- `<Game>` defaults to the main .blend file name without its extension. `SetGameName()` overrides it.
- `SetSavePath()` overrides the whole root. A `//` prefix is still accepted for development and is resolved with `BLI_path_abs(path, KX_GetMainPath().c_str())`, the same way `KX_BlenderCanvas::MakeScreenShot` already resolves it.
- Every path handed to BLI file functions is absolute, because `BLI_fopen` asserts that its path is not relative.
- The directory is created with `BLI_dir_create_recursive()` on first use.

### 7.2 Write sequence (worker thread)

```cpp
// Data must reach the device before the rename: otherwise a crash can leave the renamed
// file pointing at blocks that were never written (zero-length or garbage save).
int fd = BLI_open(tmp_path, O_WRONLY | O_CREAT | O_TRUNC | O_BINARY, 0644);
write_all(fd, buf.data(), buf.size());  // loops on short writes and EINTR
flush_to_device(fd);                    // POSIX: fsync(fd). Windows: _commit(fd) (FlushFileBuffers)
close(fd);
BLI_rename_overwrite(tmp_path, final_path);  // POSIX rename(); Windows MoveFileExW(REPLACE_EXISTING)
fsync_directory(root);                       // POSIX only: makes the directory entry change durable
```

Checked against the current source:

- `BLI_rename` **fails when the target exists**; it follows Windows `rename` semantics. Rev 1 would have failed on every save after the first one. `BLI_rename_overwrite` is the correct call.
- On Windows, `BLI_rename_overwrite` ends in `MoveFileExW(..., MOVEFILE_REPLACE_EXISTING)` without `MOVEFILE_WRITE_THROUGH`. Durability of the file contents therefore comes from the `_commit` before the rename.
- On Windows the rename also fails if another process has the target open (indexers, antivirus). The worker retries up to 3 times, sleeping 50 ms between attempts. That sleep is on the worker only.
- `BLI_open` converts UTF-8 to UTF-16 on Windows, so a profile path such as `C:\Users\José\…` works. `std::ofstream` does not, which is why it was removed.

---

## 8. Scene snapshot

### 8.1 What is captured

One scene per save: the scene that issued the request, meaning the current scene of the calling script or the scene that owns the actuator. Overlay and background scenes (HUDs, for example) are not captured in v1.

| Data | Source |
|---|---|
| Identity | Blender object name plus library index (originals); template name (replicas, `KX_GameObject::IsReplica()`) |
| Hierarchy | `GetParent()` |
| Local transform | `NodeGetLocalPosition()`, `NodeGetLocalOrientation()` stored as a quaternion via `getRotation()`, `NodeGetLocalScaling()` |
| Velocities | `GetLinearVelocity(false)`, `GetAngularVelocity(false)` (world space) |
| Flags | `GetVisible()`, `IsDynamicsSuspended()` |
| Logic state mask | `SCA_IObject::GetState()` |
| Game properties | int, float (including timers), bool, string |
| Lifespan | The `::timebomb` property set by `KX_Scene::ApplyLifespan` (captured like any other property) |
| Active actions | `GetActionName(layer)`, `GetActionFrame(layer)` and the playback parameters (§14) |
| Python attributes (`WITH_PYTHON`) | `m_attr_dict`: values that are not `EXP_Value`, such as lists, dicts and object references |

### 8.2 Not captured in v1

- Objects in the inactive list. They are templates.
- Objects parked in an object pool, if `KX_ObjectPool` lands. Pool occupancy is rebuilt rather than saved.
- Bullet internals: contacts, sleep and deactivation state, constraint impulses.
- `KX_PythonComponent` instance state (§15).
- `EXP_Value` vector properties (`VECTOR2/3/4`). They are skipped with a warning.
- Camera and lamp parameters beyond their transform.

### 8.3 Identity

- **Originals** (converted from the .blend) are keyed by `(library index, Blender object name)`. Names are unique within a library. The snapshot stores the list of library file paths that were loaded.
- **Replicas** are keyed by their template's Blender object name (`GetBlenderObject()`) and their position in the table. They are recreated, not matched.
- At capture, the table is ordered parents-before-children: a DFS from the roots, O(N). Restore can then create objects in table order and every parent already exists.

### 8.4 Layout

Each array is allocated as a `std::vector<uint8_t>` of exact size and filled through `memcpy` stores. Those compile to plain moves and avoid strict-aliasing UB. The array is then moved into the tree with `json::binary(std::move(blob))`. This was tested: the move transfers the pointer, with no copy. The json tree for the scene therefore has a few dozen nodes regardless of object count, so the per-node cost of A1 does not scale with the scene.

| Key | Element | Count | Notes |
|---|---|---|---|
| `v` | int | 1 | Snapshot layout version; engine-owned, separate from `schema_version` |
| `name` | str | 1 | Scene name |
| `libs` | array of str | L | Library file paths that must be loaded for the restore |
| `strings` | bin | — | NUL-terminated UTF-8 pool; every string field is a byte offset into it |
| `o_lib` | u16 | N | Library index, or `0xFFFF` for the main file |
| `o_name` | u32 | N | String offset |
| `o_kind` | u8 | N | 0 = original, 1 = replica, 2 = implicit child of a replica (§8.6) |
| `o_parent` | i32 | N | Table index, or −1 |
| `o_flags` | u8 | N | bit0 visible, bit1 dynamics suspended |
| `o_state` | u32 | N | Logic state mask |
| `x_pos` | f32×3 | N | Local position |
| `x_rot` | f32×4 | N | Local orientation, quaternion (w, x, y, z) |
| `x_scl` | f32×3 | N | Local scale |
| `p_lin` | f32×3 | N | Linear velocity, world space; zero for non-dynamic objects |
| `p_ang` | f32×3 | N | Angular velocity, world space |
| `pr_obj`, `pr_name`, `pr_type`, `pr_val` | u32, u32, u8, u64 | P | Property table. `pr_val` holds the int64, double bits, bool, or a string offset. |
| `a_obj`, `a_layer`, `a_name`, `a_frame`, `a_start`, `a_end`, `a_speed`, `a_weight`, `a_blendin`, `a_mode`, `a_blendmode`, `a_priority` | … | A | Active action layers |
| `py` | map | — | `WITH_PYTHON` only: `{ "<object index>": <converted m_attr_dict> }` |

**Size.** About 80 bytes per object before properties. 10,000 objects take about 800 KB raw, and about 1.26 MiB with three properties each (§2).

**Quaternion vs matrix.** Storing the quaternion halves the orientation bytes (16 vs 36). The round trip is exact to float epsilon, not bit-exact. Positions and scales are stored bit-exact.

**Bounds checks on load**, before anything is applied:

- every blob size equals count × stride
- `o_parent < N` and parents precede their children
- every string offset is inside the pool, and the pool ends with NUL
- `pr_obj` and `a_obj` are below N
- enum fields are in range

### 8.5 Capture cost

There is one pass over `GetObjectList()`:

- Writes go sequentially into the SoA arrays.
- Reads chase pointers: object → SG_Node → physics controller, with virtual calls into Bullet for the velocities.
- **Estimate: on the order of a few hundred ns per object, so a few ms per 10,000 objects. It must be measured.** It is the first profiling target, since it runs on the main thread.

Property iteration must not use the existing index API. `EXP_Value::GetProperty(int)` walks `m_properties` (a `std::map`) from the beginning on every call, so iterating with it is O(n²). `GetPropertyNames()` allocates one string per property. §14 adds a single-pass visitor instead.

### 8.6 Restore algorithm

`Load()` with `apply_scene = true` returns the user data immediately. Scene application is queued and runs at the start of the next `NextFrame()`, before logic. No logic brick or Python iteration is active at that point.

1. **Pre-check.** The scene must be active and every entry in `libs` must be loaded. Otherwise → `SceneMismatch`, and nothing is touched.
2. **Remove current replicas.** Remove every replica in the scene with `KX_Scene::RemoveObject`, the immediate path. `DelayedRemoveObject` is not used, so old and new replicas never coexist for a physics step.
3. **Originals.**
   - Build a hash map `(lib, name) → object` from the scene's originals and apply the saved state to each one found.
   - Originals that are present now but absent from the snapshot were ended before the save; remove them.
4. **Replicas.** In table order:
   - Look up the template by name in `GetInactiveList()`, falling back to the active list.
   - Call `AddReplicaObject(template, nullptr, lifespan_frames)`.
   - Compute `lifespan_frames = timebomb_seconds × 60`, because `ApplyLifespan` multiplies by 1/60.
   - A replica of a collection instance spawns its children itself. Those children are stored with `o_kind = 2` and are matched by (parent index, name) instead of being added again.
5. **Per-object state.**
   - `NodeSetLocalPosition/Orientation/Scale`. For root objects these already push into the physics controller.
   - `SetParent` / `RemoveParent`.
   - `setLinearVelocity` / `setAngularVelocity` in world space.
   - `SuspendDynamics` / `RestoreDynamics` on the controller.
   - `SetState`.
   - `SetVisible(v, false)`.
6. **Properties.**
   - If a property of the same type exists, update it in place with `EXP_Value::SetValue`. This is required for timers: `SCA_TimeEventManager` holds an `AddRef`'d pointer to the timer value, and `EXP_Value::SetProperty` replaces the object. Replacing it would leave the manager advancing an orphan while the visible property stays frozen.
   - Otherwise, create the property with `SetProperty`.
7. **Actions.** `PlayAction` with the stored parameters, then `SetActionFrame`.
8. **Python attributes.** Restored into `m_attr_dict` through the normal item-assignment path.

Python references to replicas removed in step 2 become invalid through the existing proxy invalidation. Scripts must re-acquire object references after a load; this is documented in the API.

### 8.7 Object references in user data

A `KX_GameObject` found in user data (`globalDict`, `m_attr_dict`) is stored as a msgpack ext value: `json::binary` with subtype 1 holding a 4-byte table index. This was tested: it encodes as `fixext4` and round-trips with its subtype. After the restore, the reference resolves to the recreated object. If the save has no scene section, it loads as `None`.

---

## 9. Data model

- `SaveData` wraps `nlohmann::json`. `json.hpp` is a single header of about 25k lines and Blender ships no `json_fwd.hpp`, so `SaveData.h` is included only by save-system translation units and the Python binding.
- **Getters never throw.** nlohmann's `value()` throws `type_error.302` when the stored type differs (tested). Getters check `is_number_integer()`, `is_string()` and so on first, and return the default on a mismatch.
- **Float width.** `to_msgpack` already emits float32 when a double is exactly representable as one (tested: `100.0` takes 5 bytes, `0.1` takes 9). There is no need for a separate float API.
- **Serialisation** appends into a reused `std::vector<uint8_t>` (tested: capacity is kept).
- **Parsing** uses `from_msgpack(first, last, true, false)` followed by an `is_discarded()` check (tested on garbage and truncated input).

**Python → SaveData conversion**

| Python | Stored as | Loads back as |
|---|---|---|
| `dict` with `str` keys | map | `dict` |
| `dict` with non-`str` keys | — | `TypeError`. Keys are not silently stringified. |
| `list`, `tuple` | array | `list` (tuple identity is lost; documented) |
| `str`, `bool`, `None`, `float` | same | same |
| `int` | int64 | `int`; outside the int64 range → `OverflowError` |
| `bytes` | bin | `bytes` |
| `mathutils.Vector/Matrix/Quaternion/Euler/Color` | array of float | `list` |
| `KX_GameObject` | ext reference (§8.7) | the object, or `None` |
| anything else | — | `TypeError` naming the key path |

Conversion depth is capped at 64. A dict that contains itself would otherwise recurse until the stack overflows.

**Existing globalDict saving.** `bge.logic.saveGlobalDict()` / `loadGlobalDict()` (marshal into `.bgeconf`) stay unchanged. Marshal preserves Python types exactly but cannot be read from C++. `bge.save` is the portable path.

---

## 10. Migrations

- `schema_version` is a game-defined integer stored in the header. The game registers one function per version step.
- A function registered for version `v` upgrades a tree from `v` to `v + 1`, in place:

```cpp
using SaveMigrationFn = void (*)(SaveData &user);  // mutate in place: no tree copies per step

// Applied in a flat loop: one step per version, no graph search, no string comparisons.
for (uint32_t v = file_version; v < current_version; v++) {
  if (migrations[v] == nullptr) {
    return SaveError::VersionGap;
  }
  migrations[v](user);
}
```

- Game migrations only see the `user` section. The `scene` section is versioned by the engine through its own `v` key.
- In Python, `bge.save.registerMigration(from_version, fn)` takes an `fn(dict) -> None` that mutates the dict in place.

---

## 11. Slot table

- **Scan.** The first query scans the save root once with `BLI_filelist_dir_contents()`. For each `.upsav` it reads the header and metadata (steps 1–6 of §6.5) and deletes any leftover `.tmp`.
- **Updates.** The table changes when a save completes (in `Poll()`) and on `Delete()`. `Exists()` and `List()` are memory lookups with no syscalls, so calling them every frame is free. This replaces rev 1's "Save Exists" property sensor, which would have done a `stat()` per tick.
- **Thumbnails** are read on demand: `thumb_size` bytes at offset `header_size + meta_size`, then decompressed.

---

## 12. API

### 12.1 C++

```cpp
enum class SaveError : uint8_t {
  None, NotFound, Io, Corrupted, Unsupported, VersionTooNew, VersionGap,
  SceneMismatch, InvalidSlotName, TooLarge,
};
enum class SaveStatus : uint8_t { Idle, Pending, Ok, Failed };

struct SaveRequest {
  SaveData user;           // moved in; the caller's tree is not copied
  SaveData custom_meta;    // goes to the metadata "custom" map
  bool include_scene = true;
  bool thumbnail = true;
};

struct SaveSlotInfo {
  std::string slot;
  int64_t timestamp;
  uint32_t schema_version;
  bool has_thumbnail;
  SaveData meta;           // parsed metadata block
};

class KX_SaveSystem {
 public:
  /* Configuration. */
  void SetSavePath(const std::string &path);
  void SetGameName(const std::string &name);
  void SetGameVersion(const std::string &version);
  void SetSchemaVersion(uint32_t version);
  void RegisterMigration(uint32_t from_version, SaveMigrationFn fn);

  /* Returns immediately; I/O completes on the background pool. */
  SaveError Save(const std::string &slot, SaveRequest &&request);
  SaveStatus GetStatus(const std::string &slot, SaveError *r_error = nullptr) const;

  /* Synchronous read; scene application is deferred to the next frame boundary. */
  SaveError Load(const std::string &slot, SaveData &r_user, bool apply_scene);

  bool Exists(const std::string &slot) const;
  const std::vector<SaveSlotInfo> &List();
  SaveError Delete(const std::string &slot);
  SaveError ReadThumbnail(const std::string &slot, int &r_w, int &r_h, std::vector<uint8_t> &r_rgba);

  /* Engine hooks (KX_KetsjiEngine only). */
  void EndFrame(RAS_Rasterizer *rasty, const RAS_Rect &viewport);  // thumbnail readback + job push
  void BeginFrame();  // Poll() + apply a pending scene restore
  void Shutdown();    // BLI_task_pool_work_and_wait + final Poll()
};
```

### 12.2 Python: `bge.save`

The module is registered with `addSubModule(..., "bge.save")`, next to `bge.constraints` in `KX_PythonInit.cpp`.

```python
import bge.save as save
import bge.logic as logic

save.setSchemaVersion(3)
save.registerMigration(2, lambda d: d.setdefault("difficulty", "normal"))

# Returns immediately.
save.save("slot1", logic.globalDict, scene=True, thumbnail=True,
          meta={"chapter": "Castle", "playtime": 5412.0})

state, message = save.status("slot1")    # "idle" | "pending" | "ok" | "error"

if save.exists("autosave"):
    data = save.load("autosave", scene=True)  # scene is applied before the next logic step
    logic.globalDict.update(data)

for info in save.list():                      # memory lookup, no file access after the first scan
    print(info["slot"], info["timestamp"], info["meta"].get("chapter"))

w, h, rgba = save.thumbnail("slot1")          # raw RGBA8 bytes
save.delete("slot1")
```

---

## 13. Logic bricks: Game actuator *Save Slot* / *Load Slot*

Logic bricks get two new modes on the existing Game actuator. The slot name lives in the actuator's existing `filename[64]` field, and the thumbnail toggle in its existing, so far unused `flag` field. No DNA struct changes, so there is no versioning code and no SDNA layout change.

### 13.1 Behaviour

| Mode | On a positive pulse |
|---|---|
| *Save Slot* | Calls `Save(slot)` for the actuator's scene: scene snapshot, metadata, and a thumbnail unless *Thumbnail* is off. The user section is `bge.logic.globalDict` in builds `WITH_PYTHON`, and empty otherwise. Returns immediately (§5). |
| *Load Slot* | Calls `Load(slot, apply_scene = true)`. With Python, the contents of `globalDict` are replaced with `PyDict_Clear` + `PyDict_Update`, exactly like the existing *Load bge.logic.globalDict* mode. The scene is restored before the next logic step (§8.6). |

- **Works without Python** (Android). The scene snapshot carries the game state.
- **Errors** (invalid slot name, slot not found, corrupted file) are reported with `CM_Error`. Scripts can read the result with `bge.save.status(slot)`.
- **Repeated pulses.** Each positive pulse queues one save, and the serial pool keeps them in order (§5.4). As with other one-shot actuators, drive it from a positive-edge sensor.
- **No completion sensor in v1.** It would need a new DNA sensor struct.

### 13.2 Blender side

`makesdna/DNA_actuator_types.h`, after `ACT_GAME_SCREENSHOT 6`:

```c
#define ACT_GAME_SAVESLOT 7
#define ACT_GAME_LOADSLOT 8

/* gameactuator->flag (field existed but was unused) */
#define ACT_GAME_NO_THUMBNAIL (1 << 0)
```

The new values are appended, so existing files keep their meaning. `flag` is zero in existing files, which means *Thumbnail* is on by default. That is why the bit is stored negated.

`makesrna/intern/rna_actuator.cc`, in `rna_def_game_actuator()`:

```cpp
/* Appended to prop_type_items. */
{ACT_GAME_SAVESLOT, "SAVESLOT", 0, "Save Slot", "Save the scene state and bge.logic.globalDict to a save slot"},
{ACT_GAME_LOADSLOT, "LOADSLOT", 0, "Load Slot", "Load a save slot and restore the scene state"},

/* Same DNA field as "filename", exposed as a plain string: a slot is a name, not a path,
 * so the file browser button that PROP_FILEPATH adds would be misleading. */
prop = RNA_def_property(srna, "slot_name", PROP_STRING, PROP_NONE);
RNA_def_property_string_sdna(prop, nullptr, "filename");
RNA_def_property_ui_text(prop, "Slot", "Save slot name: letters, digits, '_' and '-', up to 63 characters");
RNA_def_property_update(prop, NC_LOGIC, nullptr);

prop = RNA_def_property(srna, "use_thumbnail", PROP_BOOLEAN, PROP_NONE);
RNA_def_property_boolean_negative_sdna(prop, nullptr, "flag", ACT_GAME_NO_THUMBNAIL);
RNA_def_property_ui_text(prop, "Thumbnail", "Store a thumbnail of the current frame in the save");
RNA_def_property_update(prop, NC_LOGIC, nullptr);
```

`editors/space_logic/logic_window.cc`, in `draw_actuator_game()`:

```cpp
const int mode = RNA_enum_get(ptr, "mode");
if (ELEM(mode, ACT_GAME_LOAD, ACT_GAME_SCREENSHOT)) {
  layout->prop(ptr, "filename", UI_ITEM_NONE, std::nullopt, ICON_NONE);
}
else if (ELEM(mode, ACT_GAME_SAVESLOT, ACT_GAME_LOADSLOT)) {
  layout->prop(ptr, "slot_name", UI_ITEM_NONE, std::nullopt, ICON_NONE);
  if (mode == ACT_GAME_SAVESLOT) {
    layout->prop(ptr, "use_thumbnail", UI_ITEM_NONE, std::nullopt, ICON_NONE);
  }
}
```

### 13.3 Gameengine side

- `SCA_GameActuator.h`: add `KX_GAME_SAVESLOT` and `KX_GAME_LOADSLOT` before `KX_GAME_MAX`. The Python `mode` attribute accepts the range `[KX_GAME_NODEF + 1, KX_GAME_MAX - 1]`, so it covers the new modes with no further change. Add a `bool thumbnail` constructor parameter.
- `BL_ConvertActuators.cpp`: two cases mapping `ACT_GAME_SAVESLOT` and `ACT_GAME_LOADSLOT` to the new modes. They read `gameact->filename` and `gameact->flag`.
- `SCA_GameActuator::Update()`: two cases that call `m_ketsjiengine->GetSaveSystem()`.

---

## 14. Integration points

**New files** (`source/gameengine/Ketsji/`)

| File | Contents |
|---|---|
| `KX_SaveSystem.h/.cpp` | Public API, slot table, jobs, engine hooks |
| `KX_SaveData.h/.cpp` | `SaveData` wrapper with non-throwing getters |
| `KX_SaveFile.h/.cpp` | Header, validation, depth guard, durable write, zstd |
| `KX_SceneSnapshot.h/.cpp` | Capture, bounds checks, restore |
| `KX_PySaveBinding.h/.cpp` | `bge.save` (`WITH_PYTHON` only) |

**Modified gameengine files**

| File | Change |
|---|---|
| `KX_KetsjiEngine.h/.cpp` | Member. `BeginFrame()` hook in `NextFrame()`. `EndFrame()` hook next to `m_canvas->FlushScreenshots()`. `Shutdown()` in `StopEngine()`. |
| `KX_PythonInit.cpp` | `addSubModule` for `bge.save` |
| `BL_Action.h` | Const getters for `m_startframe`, `m_endframe`, `m_speed`, `m_layer_weight`, `m_blendin`, `m_playmode`, `m_blendmode`, `m_priority`. These are private today, and only the frame and name are exposed. |
| `EXP_Value.h` | `template<class F> void ForEachProperty(F &&f)`: one pass over `m_properties` with no string allocations (replaces the O(n²) index loop) |
| `Ketsji/CMakeLists.txt` | `PRIVATE bf::extern::json`, `PRIVATE bf::dependencies::zstd` (both targets already exist) |
| `GameLogic/SCA_GameActuator.h/.cpp` | Two modes, `thumbnail` flag, two cases in `Update()` (§13.3) |
| `Converter/BL_ConvertActuators.cpp` | Two `ACT_GAME_*` cases (§13.3) |

**Blender side** (§13.2), with no DNA struct layout changes:

| File | Change |
|---|---|
| `makesdna/DNA_actuator_types.h` | `ACT_GAME_SAVESLOT`, `ACT_GAME_LOADSLOT`, `ACT_GAME_NO_THUMBNAIL` |
| `makesrna/intern/rna_actuator.cc` | Two enum items, `slot_name` and `use_thumbnail` properties |
| `editors/space_logic/logic_window.cc` | Show the two properties for the new modes |

---

## 15. Risks and open questions

1. **Capture cost.** The §8.5 estimate is unmeasured and runs on the main thread. Profile it with 10k objects before committing to an autosave frequency.
2. **Thumbnail readback** is a synchronous GPU stall. Measure it at 1080p and 4K; if it is too slow, default `thumbnail` to off for autosaves.
3. **`KX_PythonComponent` state.** It could use optional `onSave() -> dict` / `onLoad(dict)` hooks per component, stored under `scene.py`. To be decided.
4. **Collection-instance replicas.** Matching children by (parent, name) fails if two children of one instance share a name. They cannot within one library, but this is worth a test.
5. **Android save root.** It needs the path from the Android launcher (`getFilesDir`). It is unresolved until the port defines how the launcher passes it.
6. **Async load prefetch.** A read + decompress job on the same serial pool for loading screens that animate. It is not needed for v1.
7. **Physics fidelity.** Restored bodies wake with their saved velocity but without contact history. Stacks of resting bodies may jitter for a frame or two. Document it, or suspend dynamics for the first restored frame.

---

## 16. Verification references

The API claims above were checked against UPBGE `6ae4cf77` (2026-10-22). The format claims were checked against compiled tests using `extern/json` (nlohmann 3.11.3).

| Claim | Location |
|---|---|
| `BLI_rename` refuses an existing target; `BLI_rename_overwrite` replaces it | `blenlib/BLI_fileops.hh:60-99` |
| Windows overwrite uses `MoveFileExW(REPLACE_EXISTING)` without write-through | `intern/utfconv/utf_winfunc.cc:86` |
| `BLI_fopen` asserts a non-relative path; `BLI_open` exists | `blenlib/intern/fileops_c.cc:569-574`, `BLI_fileops.hh:324-326` |
| Background serial task pool | `blenlib/BLI_task_c.hh:77-88` |
| nlohmann 3.11.3 in tree, target `bf::extern::json` | `extern/json/include/json.hpp:68`, `extern/json/CMakeLists.txt:7` |
| zstd required on all platforms, target `bf::dependencies::zstd` | `build_files/cmake/platform/dependency_targets.cmake:105-111` |
| C++20 (`std::endian`) | `CMakeLists.txt:2389` |
| Engine restart loop | `BlenderRoutines/BL_KetsjiEmbedStart.cpp:250-387` |
| `StopEngine` drains async loads; `FlushScreenshots` at end of frame | `Ketsji/KX_KetsjiEngine.cpp:1068-1071`, `:307` |
| Screenshot readback via `RAS_Rasterizer::MakeScreenshot` | `Rasterizer/RAS_ICanvas.cpp:153-156` |
| `AddReplicaObject(obj, loc, lifespan)`; lifespan stored as frames × 1/60 in `::timebomb` | `Ketsji/KX_Scene.h:400-402`, `KX_Scene.cpp:1424-1435` |
| `RemoveObject` is the immediate removal path | `Ketsji/KX_Scene.cpp:2106` |
| `IsReplica`, transform, velocity, parent, action, visibility APIs | `Ketsji/KX_GameObject.h:165-710` |
| `m_attr_dict` holds non-`EXP_Value` Python attributes | `Ketsji/KX_GameObject.h:216-226` |
| `SetProperty` replaces the value object; `GetProperty(int)` is a linear walk | `Expressions/intern/Value.cpp:150-166`, `:243-253` |
| Timer values are `AddRef`'d by the time manager | `GameLogic/SCA_TimeEventManager.cpp:87-91` |
| `BL_Action` playback parameters are private | `Ketsji/BL_Action.h:56-75` |
| `saveGlobalDict` uses marshal | `Ketsji/KX_PythonInit.cpp:2378-2381` |
| `bge.*` submodule registration | `Ketsji/KX_PythonInit.cpp:1838-1844` |
| Game actuator modes and the `filename` field | `GameLogic/SCA_GameActuator.h:48-57`, `Converter/BL_ConvertActuators.cpp:701-731` |
| `bGameActuator` layout: `filename[64]`, `flag` unused elsewhere; `ACT_GAME_*` defines end at 6 | `makesdna/DNA_actuator_types.h:202-207`, `:546-553` |
| Game actuator RNA enum; `filename` is `PROP_FILEPATH` | `makesrna/intern/rna_actuator.cc:1909-1945` |
| `RNA_def_property_string_sdna`, `RNA_def_property_boolean_negative_sdna` | `makesrna/RNA_define.hh:391`, `:413` |
| Game actuator UI | `editors/space_logic/logic_window.cc:2025-2029` |
| Python `mode` range is `[KX_GAME_NODEF + 1, KX_GAME_MAX - 1]` | `GameLogic/SCA_GameActuator.cpp:196-198` |
| *Load globalDict* replaces contents with `PyDict_Clear` + `PyDict_Update` | `Ketsji/KX_PythonInit.cpp:2464-2465` |
| `json::binary(std::move(v))` does not copy; float32 narrowing; no-throw parse; ext subtype round-trip; stack overflow depth; SAX depth guard | Compiled tests against `extern/json` (§2, §6.6, §8.7, §9) |
