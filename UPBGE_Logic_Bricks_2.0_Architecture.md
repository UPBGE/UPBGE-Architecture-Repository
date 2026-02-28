# UPBGE — Logic Bricks 2.0
## Architecture Document

**Target System:** `source/blender/editors/space_logic/`  
**Language:** C++ / DNA / RNA

---

## 1. Executive Summary

This document defines the architectural overhaul of the Blender Logic Bricks Editor for UPBGE. The objective is to modernize the existing Logic Stack paradigm into a robust, organized IDE-like panel while maintaining full backward compatibility with existing `.blend` files.

The new design retains the familiar three-column layout (Sensors / Controllers / Actuators) but adds high-level organization through **Logic Layers (Tabs)** and collapsible **Groups (Meta-Bricks)**. A single shared `View2D` ensures synchronized vertical scrolling across all three columns. Asset reusability is addressed through a **dual IO system** (binary for in-session operations, JSON for file-based export/import). Real-time **Path Highlighting** with UPBGE-branded theming provides debugging support.

The editor operates exclusively in edit mode. It does **not** redraw during runtime execution. These are mutually exclusive states, which eliminates all threading concerns in the editor.

---

## 2. Viewport & Interaction Architecture

### 2.1. Shared View2D — Single Scroll Region

The three columns (Sensors, Controllers, Actuators) share a single `View2D` instance. There is one vertical scrollbar on the right. Scrolling moves all three columns simultaneously with no synchronization logic required — they are simply different X zones within the same region.

```cpp
/* space_logic.cc -> logic_new() */
region->v2d.scroll   = V2D_SCROLL_RIGHT;
region->v2d.keepzoom = V2D_LOCK_ZOOM_X | V2D_LOCK_ZOOM_Y | V2D_LIMITZOOM;
region->v2d.align    = V2D_ALIGN_NO_NEG_X | V2D_ALIGN_NO_POS_Y;

/* Total scrollable height = tallest column, not the sum of all columns */
region->v2d.tot.ymin = -MAX3(height_sensors, height_controllers, height_actuators);
region->v2d.tot.ymax = 0;
```

### 2.2. Dynamic Column Layout Engine

Columns are calculated from the available region width at draw time. When the Properties panel is open, the width is divided into 4 equal columns; otherwise into 3. The **last column always absorbs the integer remainder** to prevent pixel gaps from float truncation.

```cpp
/* logic_window.cc -> logic_buttons() */
bool properties_open = (region_properties->flag & RGN_FLAG_HIDDEN) == 0;
int  num_columns     = properties_open ? 4 : 3;
int  w_col           = total_width / num_columns;

int x_sens = 0,         w_sens = w_col;
int x_cont = w_col,     w_cont = w_col;
int x_act  = w_col * 2, w_act  = properties_open ? w_col
                                : (total_width - w_sens - w_cont); /* absorbs residual */
int x_prop = properties_open ? (w_col * 3) : -1;
int w_prop = properties_open ? (total_width - w_col * 3) : 0;
```

### 2.3. Brick Ordering & Drag-and-Drop

Bricks within each column maintain explicit list order. Two mechanisms are provided for reordering:

- **Up/Down arrow buttons** on each brick header (existing behavior, preserved).
- **Drag-and-drop within the same column:** click and hold for 300ms on the brick header to enter drag mode. A semi-transparent ghost follows the cursor. A horizontal insertion line indicates the drop target. On release, `LOGIC_OT_brick_move` executes with source and destination indices. Cross-column dragging is not permitted.

### 2.4. Cable Routing

Bézier curves are retained for all connections. Manhattan routing (90-degree only) was considered and rejected — Bézier is more readable for variable-length connections and is already implemented. No change required.

---

## 3. Data Structure Extensions (DNA)

### 3.1. Logic Layers — `logic_mask`

Each brick carries a 32-bit bitmask indicating which layers it belongs to. Bit 0 = Layer 0, Bit 1 = Layer 1, etc. Up to 32 layers. The type is explicit `uint32_t` for portability across platforms.

```cpp
/* DNA_object_types.h — applied to bSensor, bController, bActuator */
uint32_t logic_mask;  /* Bit N = visible in Layer N. Default: 0 (Layer 0) */
```

```cpp
/* DNA_space_types.h */
typedef struct SpaceLogic {
    /* ... existing fields ... */
    int  active_layer_index;   /* Current visible tab (0–31)  */
    char layer_names[32][64];  /* Custom names per tab        */
} SpaceLogic;
```

> **Migration note:** Default value of `logic_mask = 0` means Layer 0, which is correct. No `do_versions` migration is needed — existing projects load with all bricks on Layer 0 and remain fully visible.

### 3.2. Logic Groups — `bLogicGroup`

Groups allow bricks to be encapsulated in collapsible containers. Each group is identified by a `uint32_t` ID generated from a per-object monotonically increasing counter, guaranteeing uniqueness within an object for the lifetime of the project.

```cpp
/* DNA_object_types.h — new struct */
typedef struct bLogicGroup {
    struct bLogicGroup *next, *prev;
    char     name[64];    /* Group display name              */
    uint32_t id;          /* Unique ID — never reused        */
    short    flag;        /* LOGIC_GROUP_COLLAPSED           */
    short    _pad;
    float    color[3];    /* Header accent color             */
} bLogicGroup;

/* Added to Object struct */
ListBase logic_groups;           /* List of bLogicGroup          */
uint32_t logic_group_id_counter; /* Increments on each creation  */
```

```cpp
/* DNA_sensor_types.h — same pattern for bController, bActuator */
uint32_t group_id;  /* Links to bLogicGroup->id. 0 = ungrouped. Default: 0 */
```

> **Migration note:** `group_id` default of `0` means ungrouped. No `do_versions` needed — existing bricks load as ungrouped.

---

## 4. Layer Cache System

Filtering bricks by layer on every draw call is O(n) over all bricks in the object. For large projects this is acceptable given that the editor only redraws on UI events, not every frame. However, a dirty-flag cache is implemented to avoid any redundant iteration.

### 4.1. Cache Structure

```cpp
/* DNA_space_types.h */
typedef struct LogicLayerCache {
    ListBase sensors;      /* LinkData pointers — no ownership */
    ListBase controllers;
    ListBase actuators;
} LogicLayerCache;

typedef struct SpaceLogic {
    /* ... existing fields ... */
    LogicLayerCache layer_cache;
    bool            cache_dirty;
    Object         *cache_owner;  /* Invalidate on active object change */
} SpaceLogic;
```

### 4.2. Cache Rebuild

```cpp
static void logic_cache_rebuild(SpaceLogic *slogic, Object *ob)
{
    BLI_listbase_clear(&slogic->layer_cache.sensors);
    BLI_listbase_clear(&slogic->layer_cache.controllers);
    BLI_listbase_clear(&slogic->layer_cache.actuators);

    uint32_t mask = (1u << slogic->active_layer_index);

    LISTBASE_FOREACH(bSensor *, s, &ob->sensors) {
        if (s->logic_mask & mask)
            BLI_addtail(&slogic->layer_cache.sensors, BLI_genericNodeN(s));
    }
    LISTBASE_FOREACH(bController *, c, &ob->controllers) {
        if (c->logic_mask & mask)
            BLI_addtail(&slogic->layer_cache.controllers, BLI_genericNodeN(c));
    }
    LISTBASE_FOREACH(bActuator *, a, &ob->actuators) {
        if (a->logic_mask & mask)
            BLI_addtail(&slogic->layer_cache.actuators, BLI_genericNodeN(a));
    }

    slogic->cache_dirty = false;
    slogic->cache_owner = ob;
}
```

### 4.3. Cache Invalidation Events

| Event | Where Triggered |
|---|---|
| Active layer tab changed | `LOGIC_OT_layer_set` |
| Brick added | `LOGIC_OT_sensor_add` / `controller_add` / `actuator_add` |
| Brick deleted | `LOGIC_OT_sensor_remove` / `controller_remove` / `actuator_remove` |
| Brick layer mask changed | `LOGIC_OT_brick_layer_assign` |
| Brick moved to/from group | `LOGIC_OT_brick_move` |
| Active object changed | `logic_buttons()` detects `cache_owner != ob` |

### 4.4. Draw Loop Usage

```cpp
static void logic_buttons(const bContext *C, ARegion *region)
{
    SpaceLogic *slogic = CTX_wm_space_logic(C);
    Object     *ob     = CTX_data_active_object(C);

    if (slogic->cache_dirty || slogic->cache_owner != ob)
        logic_cache_rebuild(slogic, ob);

    /* Draw only cached visible bricks — never iterates ob->sensors directly */
    LISTBASE_FOREACH(LinkData *, link, &slogic->layer_cache.sensors) {
        draw_sensor(x_sens, &y_cursor, w_sens, (bSensor *)link->data);
    }
    /* Same for controllers and actuators */
}
```

---

## 5. Asset Management (IO System)

Two IO backends share a single intermediate data representation. The serialization layer is backend-agnostic; only the writer/reader changes.

### 5.1. Dual Backend Strategy

| Operation | Format | Rationale |
|---|---|---|
| Copy / Paste within session | Binary (in-memory) | Zero parse overhead, no string allocation |
| Export to file | JSON | Human-readable, debuggable, shareable across projects |
| Import from file | JSON | Same as export |

### 5.2. Export Scope

Export is available at two granularities:

- **Global Export:** serializes all logic bricks of the selected object across all layers and groups.
- **Layer Export:** serializes only the bricks visible in the currently active layer (those matching the `active_layer_index` bitmask). The exported JSON includes the layer index as metadata.

### 5.3. JSON Format

```json
{
  "asset_type": "LOGIC_GROUP",
  "name": "PlayerJump",
  "export_scope": "LAYER",
  "layer_index": 2,
  "sensors":     [ "..." ],
  "controllers": [ "..." ],
  "actuators":   [ "..." ]
}
```

`export_scope` is either `"GLOBAL"` or `"LAYER"`. `layer_index` is only present when scope is `"LAYER"`.

### 5.4. Reference Resolution on Import

Logic bricks reference other datablocks (Objects, Sounds, Materials) by pointer. On export these are converted to name strings. On import a **temporary hash map is built once** over the current scene before resolving any reference, reducing complexity from O(n_refs × n_objects) to O(n_objects + n_refs).

```cpp
/* Build once before resolving all references */
GHash *name_to_object = BLI_ghash_str_new("logic_import_resolve");
LISTBASE_FOREACH(Object *, ob, &scene->object_bases) {
    BLI_ghash_insert(name_to_object, ob->id.name + 2, ob);
}

/* Each reference lookup is O(1) */
Object *resolved = BLI_ghash_lookup(name_to_object, ref_name);
if (!resolved)
    brick->flag |= LOGIC_BRICK_BROKEN_LINK; /* red border */

BLI_ghash_free(name_to_object, NULL, NULL); /* free after import */
```

---

## 6. Visual Feedback & Path Highlighting

### 6.1. Highlight State in SpaceLogic

```cpp
typedef struct SpaceLogic {
    /* ... */
    ListBase highlighted_bricks; /* LinkData pointers, no ownership   */
    void    *highlight_source;   /* Brick that was Shift+Clicked       */
    bool     highlight_dirty;    /* Recalculate path on next draw      */
} SpaceLogic;
```

### 6.2. Lifecycle

- **Shift+Click** on any brick: sets `highlight_source`, sets `highlight_dirty = true`.
- `highlight_dirty` triggers a **single bidirectional graph traversal** (Sensor → Controller → Actuator and reverse). Result stored in `highlighted_bricks`. Flag cleared.
- Path is recalculated (`highlight_dirty = true`) when: a brick is added/removed, a connection changes, or the active object changes.
- **Click anywhere else:** `highlight_source = NULL`, `highlighted_bricks` cleared.
- The traversal runs **once per interaction**, never per frame.

### 6.3. Glow Effect — Themeable (default: UPBGE Orange)

The highlight color is registered in Blender's theme system with UPBGE orange (`#FF8000`) as the default. Users can override it in **Preferences > Themes > Logic Editor**.

```cpp
/* DNA_userdef_types.h */
typedef struct ThemeLogicEditor {
    uchar logic_highlight[4]; /* Default: #FF8000FF (UPBGE orange) */
} ThemeLogicEditor;

/* Reading the theme color at draw time */
static void get_highlight_color(float r_color[4]) {
    rgba_uchar_to_float(r_color, U.themes[0].logic_editor.logic_highlight);
}
```

The glow is rendered as **four concentric rounded rectangles** with decreasing alpha, drawn before the brick itself. No framebuffer or post-processing is required.

```cpp
static void draw_brick_highlight(float x, float y, float w, float h)
{
    float base[4];
    get_highlight_color(base);

    for (int i = 3; i >= 0; i--) {
        float expand   = i * 3.0f;
        float color[4] = { base[0], base[1], base[2], 0.15f * (4 - i) };
        UI_draw_roundbox_aa(x - expand,    y - expand,
                            x + w + expand, y + h + expand,
                            4.0f + expand,  color);
    }
    /* Full-opacity border on top */
    UI_draw_roundbox_aa(x, y, x + w, y + h, 4.0f, base);
}
```

Bricks **not** in the highlighted path are drawn at **15% alpha** when a `highlight_source` is active, pushing them visually into the background.

---

## 7. Wire Coloring

Cables are colored by Controller type. When path highlighting is active, cables in the path use the theme highlight color at full opacity; cables outside the path are dimmed to 15% alpha. Error state (broken reference) overrides all other colors.

| Controller Type | Color | Hex | Width |
|---|---|---|---|
| AND | Blue | `#4A90D9` | 1.5px |
| OR | Green | `#5BA85A` | 1.5px |
| NAND | Desaturated Blue | `#6B8FA8` | 1.5px |
| NOR | Desaturated Green | `#7A9E79` | 1.5px |
| Python | Yellow | `#D4A843` | 1.5px |
| Expression | Lilac | `#9B72CF` | 1.5px |
| Highlighted | Theme color | `#FF8000*` | 2.5px |
| Broken Link | Red dashed | `#D45A5A` | 2.0px |

> \* Highlighted wire color follows the same `ThemeLogicEditor.logic_highlight` field as the brick glow. Changing the theme color affects both simultaneously.

```cpp
static void logic_draw_wire(float x0, float y0, float x1, float y1,
                             bController *cont, bool highlighted, bool broken)
{
    float color[4];

    if (broken) {
        copy_v4_fl4(color, 0.83f, 0.35f, 0.35f, 1.0f);
        UI_draw_wire_dashed(x0, y0, x1, y1, color, 2.0f);
        return;
    }
    if (highlighted) {
        get_highlight_color(color);
        UI_draw_bezier_wire(x0, y0, x1, y1, color, 2.5f);
        return;
    }
    switch (cont->type) {
        case CONT_LOGIC_AND:  copy_v4_fl4(color, 0.29f, 0.56f, 0.85f, 1.0f); break;
        case CONT_LOGIC_OR:   copy_v4_fl4(color, 0.36f, 0.66f, 0.35f, 1.0f); break;
        case CONT_LOGIC_NAND: copy_v4_fl4(color, 0.42f, 0.56f, 0.66f, 1.0f); break;
        case CONT_LOGIC_NOR:  copy_v4_fl4(color, 0.48f, 0.62f, 0.47f, 1.0f); break;
        case CONT_PYTHON:     copy_v4_fl4(color, 0.83f, 0.66f, 0.26f, 1.0f); break;
        case CONT_EXPRESSION: copy_v4_fl4(color, 0.61f, 0.45f, 0.81f, 1.0f); break;
        default:              copy_v4_fl4(color, 0.60f, 0.60f, 0.60f, 1.0f); break;
    }

    /* Dim if path highlighting is active but this wire is not in the path */
    if (slogic->highlight_source) color[3] = 0.15f;

    UI_draw_bezier_wire(x0, y0, x1, y1, color, 1.5f);
}
```

---

## 8. Implementation Roadmap

The implementation order is strictly dependency-driven. DNA must be stable before any IO, UI, or cache code is written.

| Phase | Scope | Depends On |
|---|---|---|
| **1 — DNA** | Add `logic_mask`, `group_id`, `bLogicGroup`, `logic_group_id_counter`, `ThemeLogicEditor` to DNA headers | Nothing |
| **2 — Layout Engine** | Dynamic column widths with Properties panel detection, shared `View2D`, scroll height calculation | Nothing (parallel with Phase 1) |
| **3 — Layer Tabs** | Tab bar UI, `active_layer_index`, `logic_mask` filtering, layer cache with dirty flag | Phase 1 |
| **4 — Groups** | `bLogicGroup` draw order, collapsible headers, group creation operator, drag-and-drop reorder | Phase 1, Phase 2 |
| **5 — IO System** | JSON serializer/deserializer, binary clipboard backend, hash map resolver, global/layer export scope | Phase 3, Phase 4 (stable brick structure) |
| **6 — Visual Polish** | Wire coloring, path highlighting with glow, theme color registration | Phase 2, Phase 3 |

---

## 9. Decision Summary

| Topic | Decision | Rationale |
|---|---|---|
| Canvas model | Fixed 3-column layout, no free canvas | Tabs + Groups eliminate scale problems; simplicity wins |
| Scroll | Single shared `View2D`, vertical only | Simplest correct implementation, no sync logic needed |
| Column widths | Equal thirds/quarters, integer residual on last column | Prevents pixel gap bugs from float truncation |
| Properties panel | 4 columns when open, 3 when closed | Properties panel occupies the 4th column slot |
| Brick reorder | Up/Down arrows + drag-and-drop within column | Both mechanisms for different user preferences |
| Cable routing | Bézier (unchanged) | Already implemented, more readable than Manhattan |
| `logic_mask` type | `uint32_t`, 32 layers | Explicit type, sufficient for any real project |
| Group ID | `uint32_t` + per-object counter | Standard engine pattern, zero collision risk |
| Layer cache | Dirty-flag cache in `SpaceLogic` | Avoids redundant iteration; simple invalidation model |
| IO format | Binary for clipboard, JSON for files | Performance where it matters, readability for files |
| Export scope | Global and per-layer | Layer export enables modular asset sharing |
| Reference resolution | Temporary `GHash` built once per import | Reduces O(n×m) to O(n+m) |
| Highlight color | Themeable, default UPBGE orange `#FF8000` | Brand consistency, user customizable |
| Glow technique | 4 concentric roundboxes, decreasing alpha | No framebuffer needed, compatible with Blender UI |
| Wire color | By Controller type; highlight overrides; error state highest priority | Semantic information at a glance |
| `do_versions` | Not required | All new DNA fields default to `0`, correct for existing projects |
| Runtime / Editor | Mutually exclusive states, no redraw during runtime | Eliminates all threading concerns in the editor |
