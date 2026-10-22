# UPBGE — Logic Bricks 2.0
## Architecture Document — v2

**Target system:** `source/blender/editors/space_logic/` (plus the UPBGE-owned link-button code in `source/blender/editors/interface/`)
**Language:** C++ / DNA / RNA / Python
**Status:** Draft v2 — 2026-10-14. Supersedes v1. All design decisions closed (D1–D16).
**Validated against:** `UPBGE/upbge` master `e7c5aac` (2026-10-12). Struct sizes were checked by compiling the proposed layouts for 64-bit and 32-bit.

---

## 0. Changes from v1

| # | v1 | v2 | Reason |
|---|---|---|---|
| 1 | `logic_mask = 0` means "Layer 0"; filter `mask & (1 << layer)` | Effective mask `m \| (m == 0)` (D1-A) | The v1 filter evaluates `0 & 1 = 0`: every brick of every existing file would be hidden |
| 2 | `active_layer_index`, one visible layer | `view_mask` (several layers at once) + `act_layer` (target for new bricks) | D4-A |
| 3 | Layer names in `SpaceLogic` | `GameData.logic_layers[]` in the scene | D3-A: names are data, not view state; they survive "Load UI" off and travel with the scene |
| 4 | 32 layers | 30 layers | `template_layers()` lays out at most 30 correctly; same count as game states (D11) |
| 5 | Links to hidden layers silently disappear | Dashed stub at the visible end, offset below the icon axis | D2-B, D16 |
| 6 | `LogicLayerCache` (`LinkData` lists in `SpaceLogic`) | Removed; one flag pre-pass per redraw | Leaked on every rebuild, double-freed on area split (`MEM_dupalloc`), dangling pointers written to file, use-after-free after memfile undo; slower than the bit test it replaced |
| 7 | Highlight state (`ListBase` + `void *`) in `SpaceLogic` | Per-brick trace bit, propagated in the pre-pass | Stateless, works across objects, nothing to invalidate |
| 8 | Shift+Click to trace, "click anywhere" clears | Trace toggle icon in the brick header; Esc clears | Buttons consume the click before the region keymap; clearing on any click kills the trace while editing a field |
| 9 | 4 columns when the Properties panel is open | Always 3 columns | Properties is a separate `RGN_TYPE_UI` region; the main region width already excludes it |
| 10 | Custom View2D, zoom locked by hand | `V2D_COMMONVIEW_LIST`, dynamic widths with minimum width and cable gutter | D5-B |
| 11 | `bool` fields in DNA | `uint8_t` / bit flags | `bool` is not an SDNA base type |
| 12 | `bLogicGroup` 100 B, `uint32_t` id, per-object counter | 88 B, `uint16_t` id allocated as `max + 1` | v1 fails the makesdna 64-bit size rule; D9 |
| 13 | No persistence code for groups | Write/read/free in `object.cc`, copy in `BKE_sca_copy_logicbricks()` and `OBJECT_OT_logic_bricks_copy` | Groups were never saved |
| 14 | New DNA struct `ThemeLogicEditor` | Free slots of the existing `ThemeSpace tlogic` (`hilite`, `syntax*`) | Name already taken by the RNA struct of `tlogic`; zero DNA change |
| 15 | Wire colors for 6 controller types | 8 types | `CONT_LOGIC_XOR` / `CONT_LOGIC_XNOR` exist and would render grey |
| 16 | `UI_draw_bezier_wire`, `UI_draw_wire_dashed`, `UI_draw_roundbox_aa(x, y, …)`, `scene->object_bases` | Real APIs; wires drawn by the batched `ui_draw_links()` | v1 APIs do not exist; wires are drawn in `editors/interface`, not in `space_logic` |
| 17 | 4 round boxes + 1 border per highlighted brick | Rounded halo + border through the widget shader (2 draws per highlighted brick) and one batched dim draw | D7-A, D13; 5 draws per highlighted brick → 2, dimming of every other brick in 1 |
| 18 | Persisted `LOGIC_BRICK_BROKEN_LINK`, red dashed "broken" wires | Removed; import reports unresolved references | Links are pointers and cannot break; a persisted flag goes stale when the user fixes the reference |
| 19 | C++ binary clipboard + C++ JSON | Python with RNA reflection; clipboard holds the same JSON | D6-A, D10; one code path |
| 20 | Import implied the add operators | New RNA `sensors.new()` / `remove()` (and controllers, actuators) | Each add operator calls `ED_undo_push_old()`: N bricks = N memfile undo steps |
| 21 | `LOGIC_OT_brick_move` with indices; 300 ms long-press | Name-based `LOGIC_OT_brick_move_to`; grip icon with `WM_DRAG_RNA` | Visible index ≠ list index under filters; long-press collides with the name field and type menu in the header |
| 22 | — | Phase 0: hot-path fixes and existing bugs | The measured editor cost is `O(links × buttons)` link resolution, not brick filtering |
| 23 | "Logic Layers (Tabs)", location unspecified | Own tab row under the header, full area width: a `RGN_TYPE_TOOL_HEADER` region with Blender's `ButtonType::Tab` (browser-style) | D14 |
| 24 | Groups as collapsible containers inside each column | One full-width header and frame per group; the three columns are synchronised per group band | D15 |

---

## 1. Executive Summary

This document defines the overhaul of the Logic Bricks Editor. The three-column paradigm (Sensors / Controllers / Actuators) is kept; organisation is added through **Logic Layers** (a bitmask per brick, several layers viewable at once, chosen from a browser-style tab row) and collapsible **Groups** (one framed band across the three columns per group). Links whose far end is hidden by a layer or a collapsed group are shown as stubs, so hiding never conceals the existence of a connection. **Trace** highlights the logic path through any brick. Reusable logic is exported and imported as JSON through a Python operator that reflects RNA, so new brick types need no IO code.

Performance drives the design. No per-editor cache of brick pointers exists; all derived state is recomputed every redraw by one pass over bricks and links, using bits in the bricks themselves. The quadratic link resolution of the current editor is replaced by a hash map, and per-wire draw calls are replaced by at most four batched draws.

### 1.1 Invariants

1. **Edit-only data.** `logic_mask`, `group_id`, `bLogicGroup`, `GameData.logic_layers` and all trace bits are ignored by the converter. `source/gameengine/Converter` reads only `*_DEACTIVATE` and `CONT_PRIO` from brick flags. Runtime cost: zero.
2. **Single thread.** The editor does not redraw while the embedded player runs, and all editor code runs on the main thread. No locks exist or are needed.
3. **No brick pointers outside the bricks.** Derived per-redraw state lives in transient flag bits recomputed in `O(bricks + links)`. Nothing can dangle across undo, file load, area split or Python edits.
4. **Old files load unchanged.** Every new field reads as 0, and 0 is a valid state everywhere. No data `do_versions` is required. Theme defaults need one userdef versioning block (§10).
5. **Forward compatibility.** Older UPBGE builds ignore the unknown members: bricks show without layers and groups are dropped.

---

## 2. Decision Log

| ID | Decision |
|---|---|
| D1-A | An empty `logic_mask` means layer 0, evaluated branchless as `m \| (m == 0)`. A brick can never be outside every layer. |
| D2-B | All three columns are filtered by layer. A link whose far end is hidden is drawn as a dashed stub at the visible end. |
| D3-A | Layer names live in the scene (`GameData`). |
| D4-A | The editor stores a `view_mask` bitset, so several layers can be viewed at once. |
| D5-B | Dynamic column widths, zoom locked, minimum column width, fixed gutter for cables. |
| D6-A | JSON IO in Python via RNA reflection. |
| D7-A | Bricks outside the traced path are dimmed by overlay quads in one draw call. |
| D8 | Both Layers and Groups are implemented. |
| D9 | No group-id counter on `Object`: a new group takes `max(id) + 1`; deleting a group clears its members. |
| D10 | The clipboard holds the same JSON as file export, in the system clipboard (`window_manager.clipboard`). |
| D11 | 30 layers (`LOGIC_MAX_LAYERS`), matching `OB_MAX_STATES` and the layout of `template_layers()`. |
| D12 | Within an object section, ungrouped bricks are drawn after the groups. |
| D13 | The trace glow has rounded corners and is drawn with the existing widget shader, one halo and one border per highlighted brick. |
| D14 | Layer tabs sit in their own row under the editor header, spanning the full area width, styled like browser tabs. |
| D15 | Each group has a single header spanning the three columns, with separators at the column boundaries; a group is a framed band and the columns are aligned per band. |
| D16 | Stubs are drawn offset below the icon axis so a visible wire leaving the same icon cannot cover them. |

---

## 3. Data Model (DNA)

All new members either replace padding or are appended with explicit padding. Replacing `_pad*` members is not a rename, so no `dna_rename_defs.h` entry is needed; old files simply lack the new members and read them as zero. makesdna requires numeric literals in array sizes, so macros appear only in comments.

### 3.1 Bricks

```cpp
/* DNA_sensor_types.h — fills existing padding; size unchanged (128 B on 64-bit). */
typedef struct bSensor {
  struct bSensor *next = nullptr, *prev = nullptr;
  short type = 0, otype = 0, flag = 0, pulse = 0;
  short freq = 0, totlinks = 0;
  uint32_t logic_mask = 0; /* Was _pad1, _pad2. Bit N = member of layer N. 0 = layer 0 (D1-A). */
  char name[64] = "";      /* MAX_NAME */
  void *data = nullptr;
  struct bController **links = nullptr;
  Object *ob = nullptr;
  short invert = 0;
  short level = 0;
  short tap = 0;
  uint16_t group_id = 0;   /* Was _pad. bLogicGroup::id in the owner object; 0 = ungrouped. */
} bSensor;
```

```cpp
/* DNA_controller_types.h — 136 B -> 144 B on 64-bit. */
typedef struct bController {
  struct bController *next = nullptr, *prev = nullptr, *mynew = nullptr;
  short type = 0, flag = 0, inputs = 0, totlinks = 0;
  short otype = 0, totslinks = 0;
  uint32_t logic_mask = 0; /* Was _pad2, _pad3. */
  char name[64] = "";
  void *data = nullptr;
  struct bActuator **links = nullptr;
  struct bSensor **slinks = nullptr;
  short val = 0, valo = 0;
  unsigned int state_mask = 0;
  uint16_t group_id = 0;
  char _pad4[6] = {};
} bController;
```

```cpp
/* DNA_actuator_types.h — 112 B -> 120 B on 64-bit. */
typedef struct bActuator {
  struct bActuator *next, *prev, *mynew;
  short type;
  short flag;
  short otype, go;
  char name[64]; /* MAX_NAME */
  void *data;
  Object *ob = nullptr;
  uint32_t logic_mask = 0;
  uint16_t group_id = 0;
  char _pad[2] = {};
} bActuator;
```

| Struct | 64-bit | 32-bit | `logic_mask` offset (64) |
|---|---|---|---|
| `bSensor` | 128 (unchanged) | 108 | 28 |
| `bController` | 144 (+8) | 120 | 36 |
| `bActuator` | 120 (+8) | 100 | 112 |
| `bLogicGroup` | 88 | 80 | — |
| `SpaceLogic` | 144 (+8) | 124 | — |

The growth applies to editor data only; the converter copies bricks into `SCA_*` objects and never iterates DNA bricks at runtime.

### 3.2 Flag Bits

Transient bits follow the existing precedent of `SENS_VISIBLE`, `ACT_LINKED` and `ACT_VISIBLE`: they live in the saved `flag`, are recomputed every redraw, and are never read by the converter. All values below are currently unused (checked in DNA headers, RNA, `sca.cc`, `space_logic/` and the converter).

```cpp
/* DNA_sensor_types.h — sensor->flag (existing bits 1..64) */
#define SENS_HILITE_SRC 128 /* Persistent: trace source. */
#define SENS_IN_PATH    256 /* Transient: on a traced path. */
#define SENS_HIDDEN     512 /* Transient: hidden by layer filter or collapsed group. */

/* DNA_controller_types.h — controller->flag (existing bits 1..32) */
#define CONT_STATE_VISIBLE  64   /* Transient: passes the v1.x visibility rules (object, column, state). */
#define CONT_HILITE_SRC     128  /* Persistent: trace source. */
#define CONT_PATH_DOWN      256  /* Transient: reached from a sensor source or is a source. */
#define CONT_PATH_UP        512  /* Transient: reached from an actuator source or is a source. */
#define CONT_HIDDEN         1024 /* Transient: hidden by layer filter or collapsed group. */
#define CONT_LINK_HIDDEN_IN 2048 /* Transient: at least one linked sensor is hidden. */
#define CONT_TRANSIENT_MASK \
  (CONT_STATE_VISIBLE | CONT_PATH_DOWN | CONT_PATH_UP | CONT_HIDDEN | CONT_LINK_HIDDEN_IN)

/* DNA_actuator_types.h — actuator->flag (existing bits 1..64; ACT_LINKED/ACT_VISIBLE are transient) */
#define ACT_HILITE_SRC     128  /* Persistent: trace source. */
#define ACT_IN_PATH        256  /* Transient: on a traced path. */
#define ACT_HIDDEN         512  /* Transient: hidden by layer filter or collapsed group. */
#define ACT_LINK_HIDDEN_IN 1024 /* Transient: at least one linked controller is hidden. */
#define ACT_TRANSIENT_MASK (ACT_LINKED | ACT_VISIBLE | ACT_IN_PATH | ACT_HIDDEN | ACT_LINK_HIDDEN_IN)
```

### 3.3 Groups

```cpp
/* DNA_object_types.h */
typedef struct bLogicGroup {
  struct bLogicGroup *next, *prev;
  char name[64];     /* MAX_NAME */
  uint16_t id;       /* Unique within the owner object, > 0. Bricks store 0 for "ungrouped". */
  uint8_t color[3];  /* Header accent, sRGB. */
  uint8_t flag;      /* LOGIC_GROUP_COLLAPSED */
  char _pad[2];
} bLogicGroup;       /* 88 B (64-bit), 80 B (32-bit). */

#define LOGIC_GROUP_COLLAPSED (1 << 0)

/* Object, next to `actuators` (16-byte ListBase keeps alignment): */
ListBaseT<struct bLogicGroup> logic_groups = {nullptr, nullptr};
```

**Id allocation (D9).** Ids are only needed to be unique among the live groups of one object, so there is no counter. Deleting a group clears its members first, so a reused id can never capture stale members.

**Order invariant.** `ob->logic_groups` is always in ascending id order, because every creation path appends at the tail with `max + 1`. That covers the editor, RNA `new()` and import, and copy and file read preserve list order. The editor therefore draws groups in list order, which is group 1, 2, …, N, with no sorting.

```cpp
/* sca.cc */
bLogicGroup *BKE_sca_group_add(Object *ob, const char *name)
{
  uint16_t max_id = 0;
  for (const bLogicGroup *g = ob->logic_groups.first(); g; g = g->next) {
    max_id = std::max(max_id, g->id);
  }
  if (max_id == UINT16_MAX) {
    return nullptr; /* Only reachable after 65535 creations without deleting the newest group. */
  }
  bLogicGroup *g = MEM_new_zeroed<bLogicGroup>(__func__);
  g->id = max_id + 1;
  STRNCPY(g->name, name);
  BLI_addtail(&ob->logic_groups, g);
  BLI_uniquename(&ob->logic_groups, g, DATA_("Group"), '.', offsetof(bLogicGroup, name), sizeof(g->name));
  return g;
}

void BKE_sca_group_remove(Object *ob, bLogicGroup *g)
{
  const uint16_t id = g->id;
  for (bSensor *s = ob->sensors.first(); s; s = s->next) {
    s->group_id = (s->group_id == id) ? 0 : s->group_id;
  }
  /* Same for ob->controllers and ob->actuators. */
  BLI_remlink(&ob->logic_groups, g);
  MEM_freeN(g);
}
```

**Persistence and copy** (all next to the existing SCA code paths):

| Path | Change |
|---|---|
| `object.cc` write | Write the `logic_groups` list next to `write_actuators()` |
| `object.cc` read | `BLO_read_struct_list(reader, bLogicGroup, &ob->logic_groups)` |
| `object.cc` free | `BLI_freelistN(&ob->logic_groups)` next to `BKE_sca_free_actuators()` |
| `BKE_sca_copy_logicbricks()` | `BLI_duplicatelist(&ob_new->logic_groups, &ob->logic_groups)` (bricks keep their `group_id`, which stays valid because ids are copied verbatim) |
| `OBJECT_OT_logic_bricks_copy` | Free and duplicate `logic_groups` along with the three brick lists |

### 3.4 Scene Layer Names (D3-A)

```cpp
/* DNA_scene_types.h */
#define LOGIC_MAX_LAYERS 30

typedef struct LogicLayer {
  char name[64]; /* MAX_NAME. Empty = UI shows the index. */
} LogicLayer;

typedef struct GameData {
  /* ... existing members, last one is `void *_pad10` ... */
  LogicLayer logic_layers[30]; /* LOGIC_MAX_LAYERS. 1920 B per scene, editor-only. */
} GameData;
```

Masks are per object and names are per scene. An object linked into two scenes shows the names of the scene being edited. This is accepted.

### 3.5 SpaceLogic

```cpp
/* DNA_space_types.h — 136 B -> 144 B on 64-bit. */
typedef struct SpaceLogic {
  SpaceLink *next, *prev;
  ListBase regionbase;
  int spacetype;
  float blockscale DNA_DEPRECATED;
  short blockhandler[8] DNA_DEPRECATED;
  short flag, scaflag;
  uint32_t view_mask;     /* Was _pad[4]. Layers shown in this editor; 0 = layer 0. */
  char import_string[64];
  struct bGPdata *gpd;
  uint32_t used_mask;     /* Transient: OR of effective masks of displayed bricks (tab row, §5.4). */
  uint8_t act_layer;      /* Layer receiving new bricks; always a member of view_mask. */
  char _pad1[3];
} SpaceLogic;
```

Only values are added, so the existing `logic_duplicate()` (`MEM_dupalloc`) and `logic_blend_write()` (raw struct write) stay correct. `logic_new()` sets `view_mask = 1`.

---

## 4. Visibility Model

### 4.1 Predicates

```
eff(m)        = m | (m == 0)                       /* D1-A */
view          = eff(slogic->view_mask)
layer_vis(b)  = (eff(b.logic_mask) & view) != 0
collapsed(b)  = b.group_id != 0 && (group(b).flag & LOGIC_GROUP_COLLAPSED)
HIDDEN(b)     = !layer_vis(b) || collapsed(b)
legacy_vis(b) = v1.x rules (object displayed, column expanded, state / pin / link rules)
drawn(b)      = legacy_vis(b) && !HIDDEN(b)
```

The v1.x rules per column are:

| Column | `legacy_vis` |
|---|---|
| Controller | `OB_VIS_CONT && OB_SHOWCONT && (OB_ALLSTATE \|\| ob->state & state_mask)` — stored as `CONT_STATE_VISIBLE` |
| Sensor | `OB_ALLSTATE \|\| !BUTS_SENS_STATE \|\| totlinks == 0 \|\| (PIN && BUTS_SENS_STATE) \|\| any linked controller has CONT_STATE_VISIBLE` |
| Actuator | `OB_ALLSTATE \|\| !BUTS_ACT_STATE \|\| !ACT_LINKED \|\| ACT_VISIBLE \|\| (PIN && BUTS_ACT_STATE)` |

The state rules test links against **state visibility**, not against what is drawn. A sensor whose only controller sits on a hidden layer therefore stays visible and shows a stub (D2-B), instead of vanishing through the state rule. When `view_mask` covers every used layer and no group is collapsed, `drawn == legacy_vis`, which is exactly the current behaviour.

A group is drawn once per object, as a band across the three columns (§5.3), when at least one of its members in any column is on a visible layer (`layer_vis`). Members are drawn when the group is expanded.

### 4.2 Pre-pass

Replaces the "set visible/linked flags" loop at the top of `logic_buttons()`. The controller/actuator transient bits are cleared in `set_sca_ob()`, which already visits every controller and actuator in `bmain` and writes `mynew` in the same cache line as `flag` (Appendix A.1).

```cpp
/* logic_window.cc */

/* D1-A. (m == 0) is 0 or 1, so the empty mask maps to bit 0 without a branch. */
BLI_INLINE uint32_t logic_mask_eff(const uint32_t m)
{
  return m | uint32_t(m == 0);
}

static bool logic_group_collapsed(const Object *ob, const uint16_t gid)
{
  if (gid == 0) {
    return false;
  }
  /* G is small and the nodes stay hot in L1 after the first brick of the object. */
  for (const bLogicGroup *g = ob->logic_groups.first(); g; g = g->next) {
    if (g->id == gid) {
      return (g->flag & LOGIC_GROUP_COLLAPSED) != 0;
    }
  }
  return false; /* Dangling id cannot happen (D9 clears members); treat as ungrouped. */
}

struct LogicPrepass {
  uint32_t used_mask; /* For the tab row. */
  bool any_src;       /* A trace is active: enables span collection, glow and dimming. */
};

static LogicPrepass logic_prepass(const SpaceLogic *slogic, ID **idar, const int count)
{
  const uint32_t view = logic_mask_eff(slogic->view_mask);
  LogicPrepass r = {0, false};
  bool any_up = false;

  /* Pass A — one visit per displayed brick: hidden bits, state visibility, used mask. */
  for (int a = 0; a < count; a++) {
    Object *ob = (Object *)idar[a];
    const bool all_states = (ob->scaflag & OB_ALLSTATE) != 0;
    const bool cont_col = (ob->scavisflag & OB_VIS_CONT) && (ob->scaflag & OB_SHOWCONT);

    for (bSensor *s = ob->sensors.first(); s; s = s->next) {
      const uint32_t m = logic_mask_eff(s->logic_mask);
      const bool hidden = !(m & view) || logic_group_collapsed(ob, s->group_id);
      r.used_mask |= m;
      s->flag &= ~(SENS_VISIBLE | SENS_IN_PATH | SENS_HIDDEN);
      s->flag |= hidden ? SENS_HIDDEN : 0;
      r.any_src |= (s->flag & SENS_HILITE_SRC) != 0;
    }
    for (bController *c = ob->controllers.first(); c; c = c->next) {
      const uint32_t m = logic_mask_eff(c->logic_mask);
      const bool hidden = !(m & view) || logic_group_collapsed(ob, c->group_id);
      const bool in_state = all_states || (ob->state & c->state_mask) != 0;
      r.used_mask |= m;
      c->flag |= (hidden ? CONT_HIDDEN : 0) | ((cont_col && in_state) ? CONT_STATE_VISIBLE : 0);
      if (c->flag & CONT_HILITE_SRC) {
        c->flag |= CONT_PATH_DOWN | CONT_PATH_UP;
        r.any_src = any_up = true;
      }
    }
    for (bActuator *act = ob->actuators.first(); act; act = act->next) {
      const uint32_t m = logic_mask_eff(act->logic_mask);
      const bool hidden = !(m & view) || logic_group_collapsed(ob, act->group_id);
      r.used_mask |= m;
      act->flag |= hidden ? ACT_HIDDEN : 0;
      r.any_src |= (act->flag & ACT_HILITE_SRC) != 0;
    }
  }

  /* Pass B — sensor -> controller. Most sensors are neither sources nor hidden:
   * the common case is one predictable branch and no link walk. */
  for (int a = 0; a < count; a++) {
    for (bSensor *s = ((Object *)idar[a])->sensors.first(); s; s = s->next) {
      const short mark = ((s->flag & SENS_HILITE_SRC) ? CONT_PATH_DOWN : 0) |
                         ((s->flag & SENS_HIDDEN) ? CONT_LINK_HIDDEN_IN : 0);
      if (mark == 0) {
        continue;
      }
      for (int i = 0; i < s->totlinks; i++) {
        if (s->links[i]) {
          s->links[i]->flag |= mark;
        }
      }
    }
  }

  /* Pass C — controller -> actuator. Same walk as the v1.x linked/visible loop. */
  for (int a = 0; a < count; a++) {
    Object *ob = (Object *)idar[a];
    const bool all_states = (ob->scaflag & OB_ALLSTATE) != 0;
    for (bController *c = ob->controllers.first(); c; c = c->next) {
      const short mark = ACT_LINKED |
                         ((all_states || (ob->state & c->state_mask)) ? ACT_VISIBLE : 0) |
                         ((c->flag & CONT_PATH_DOWN) ? ACT_IN_PATH : 0) |
                         ((c->flag & CONT_HIDDEN) ? ACT_LINK_HIDDEN_IN : 0);
      for (int i = 0; i < c->totlinks; i++) {
        bActuator *act = c->links[i];
        if (act == nullptr) {
          continue;
        }
        act->flag |= mark;
        if (act->flag & ACT_HILITE_SRC) {
          c->flag |= CONT_PATH_UP;
          any_up = true;
        }
      }
    }
  }

  /* Pass D — controller -> sensor, only when an upstream trace exists. */
  if (any_up) {
    for (int a = 0; a < count; a++) {
      for (bSensor *s = ((Object *)idar[a])->sensors.first(); s; s = s->next) {
        for (int i = 0; i < s->totlinks; i++) {
          if (s->links[i] && (s->links[i]->flag & CONT_PATH_UP)) {
            s->flag |= SENS_IN_PATH;
            break;
          }
        }
      }
    }
  }
  return r;
}
```

During column drawing, one walk over a sensor's links answers both the state rule and the stub rule:

```cpp
struct SensorLinkScan {
  bool state_linked; /* Replaces is_sensor_linked(): O(links) instead of O(links * buttons). */
  bool hidden_out;   /* At least one target controller is hidden -> stub. */
};

static SensorLinkScan sensor_link_scan(const bSensor *s)
{
  SensorLinkScan r = {false, false};
  for (int i = 0; i < s->totlinks; i++) {
    if (const bController *c = s->links[i]) {
      r.state_linked |= (c->flag & CONT_STATE_VISIBLE) != 0;
      r.hidden_out |= (c->flag & CONT_HIDDEN) != 0;
    }
  }
  return r;
}
```

`SENS_VISIBLE` is set only for sensors that are actually drawn, so the existing move operators skip hidden ones automatically.

**Cost.** Pass A touches each displayed brick once (the v1.x loop already does). Pass B costs one branch per sensor. Pass C is the v1.x walk. Pass D runs only while an actuator or controller is traced. The total is `O(bricks + links)` with no allocation.

---

## 5. Viewport & Layout

### 5.1 View2D

```cpp
/* space_logic.cc -> logic_main_region_init() */
view2d_region_reinit(&region->v2d, V2D_COMMONVIEW_LIST, region->winx, region->winy);
```

`V2D_COMMONVIEW_LIST` locks zoom at 1.0 (`minzoom = maxzoom = 1`), aligns to the +X/−Y quadrant and keeps `tot` strict. These settings are applied on every init, not only the first, so old files with a zoomed `cur` are normalised without versioning. The custom zoom setup in `logic_new()` becomes redundant.

Consequence: Ctrl+wheel / Ctrl+MMB zoom is removed. Size follows *Preferences › Interface › Resolution Scale* through `U.widget_unit`. Vertical scrolling is unchanged. Horizontal scrolling only becomes possible when the region is narrower than three minimum-width columns plus gutters.

### 5.2 Column Geometry (D5-B)

```cpp
/* logic_window.cc -> logic_buttons()
 * View space == pixel space (zoom locked at 1.0). The gutter is not proportional to the
 * column width: it only hosts the horizontal run of the Bezier between the Link icon of
 * one column and the Inlink icon of the next. */
const int unit = U.widget_unit;
const int margin = unit / 2;
const int gutter = (5 * unit) / 2;  /* 2.5 units, as in the current fixed layout. */
const int min_col = 17 * unit;      /* Width the brick layouts were designed for. */
const int avail = BLI_rcti_size_x(&region->v2d.mask) - 2 * margin - 2 * gutter;
const int w_col = std::max(min_col, avail / 3);
const int w_last = std::max(min_col, avail - 2 * w_col); /* Absorbs the integer remainder. */

const int x_sens = margin;
const int x_cont = x_sens + w_col + gutter;
const int x_act = x_cont + w_col + gutter;
const int tot_w = x_act + w_last + margin; /* Exceeds the region only below 3 * min_col. */

/* After the last band (§5.3): height = band cursor. */
view2d_totRect_set(&region->v2d, tot_w, y - unit);
```

### 5.3 Group Bands and Draw Order (D12, D15)

A group is drawn as one horizontal **band** spanning the three columns: a single full-width header, then the members of each column side by side, all starting at the same y. The band is as tall as its tallest column.

**Order within an object section:** group 1, 2, …, N (`ob->logic_groups` is kept in ascending id order, §3.3), then the ungrouped bricks (D12), which get neither header nor frame. Legacy files have no groups, so they look exactly as today. New bricks are created ungrouped and appear at the end of their object section.

**Consequences:**

- The columns are synchronised per band instead of flowing independently. Shorter columns of a group leave empty space. In exchange, links inside a group run nearly horizontal and the group reads as one unit.
- Object sections are aligned across the columns too, because each object's first band starts at the same y in all three. The per-column object row (show toggle, Add menu) stays as it is today; it simply sits at the same y in each column.

**Group header.** One layout row spanning `x_sens … tot_w - margin`, split into three segments whose boundaries sit at the gutter centres and are drawn as 1 px vertical separators:

| Segment | Contents |
|---|---|
| Sensors span | Disclosure (`LogicGroup.show_expanded`), colour swatch (`color`), name (non-embossed text field: click to rename), visible-sensor count right-aligned |
| Controllers span | Visible-controller count, right-aligned |
| Actuators span | Visible-actuator count, right-aligned; "⋯" menu: Copy (`logic.bricks_copy(scope='GROUP')`), Export, Delete Group |

- **Counts:** members on visible layers (`layer_vis`). State filtering does not change them.
- **Skipping:** a group with no member on a visible layer is skipped entirely.
- **Collapsed:** header only, and the members are not built at all (no buttons are created), which is where collapsing saves time.

**Frame.** After `block_end()`, each drawn group's rectangle (header + band) is filled with `TH_BACK` shaded slightly lighter, outlined with the group colour at 35 % alpha, and given a darker header strip. That is two `draw_roundbox_4fv_ex()` calls per drawn group, issued before `block_draw()`. Groups per object are few, so this stays within tens of draws.

**Band layout.** This replaces the three independent column passes of `logic_buttons()`. Each column of a band is its own layout root, resolved immediately so the band height is known before the next band starts. It is the same `block_layout_resolve()` pattern as today, called per band and column instead of once per column.

```cpp
/* logic_window.cc -> logic_buttons(). View-space y decreases downwards. */
for (int a = 0; a < count; a++) {
  Object *ob = (Object *)idar[a];
  y = logic_draw_object_rows(C, block, ob, geo, y); /* Per-column toggle + Add menu, shared y. */

  LogicSlots<bSensor> sens = logic_bucket_by_group(ob, ob->sensors, view);
  LogicSlots<bController> cont = logic_bucket_by_group(ob, ob->controllers, view);
  LogicSlots<bActuator> act = logic_bucket_by_group(ob, ob->actuators, view);

  const bLogicGroup *g = ob->logic_groups.first();
  const int nslots = sens.start.size() - 1; /* Groups in list order, then the ungrouped slot. */
  for (int s = 0; s < nslots; s++) {
    const bool is_group = (g != nullptr);
    const int visible = sens.visible[s] + cont.visible[s] + act.visible[s];
    if (is_group && visible == 0) {
      g = g->next;
      continue; /* Nothing on a visible layer: no header, no band. */
    }
    const int band_top = y;
    if (is_group) {
      y = logic_draw_group_header(C, block, ob, g, sens.visible[s], cont.visible[s], act.visible[s], geo, y);
    }
    if (!is_group || !(g->flag & LOGIC_GROUP_COLLAPSED)) {
      /* Each call builds one layout root at (x, y) and returns block_layout_resolve(block).y. */
      int bottom = y;
      bottom = std::min(bottom, logic_draw_sensor_slot(C, block, sens.slot(s), geo.x_sens, y));
      bottom = std::min(bottom, logic_draw_controller_slot(C, block, cont.slot(s), geo.x_cont, y));
      bottom = std::min(bottom, logic_draw_actuator_slot(C, block, act.slot(s), geo.x_act, y));
      y = bottom - band_pad;
    }
    if (is_group) {
      frames.append({band_top, y, g}); /* Drawn after block_end(), before block_draw(). */
      g = g->next;
    }
    y -= band_gap;
  }
}
```

**Bucketing.** Walking the brick list once per group would cost `N × G` pointer chases over scattered allocations. Instead, each object's bricks are bucketed by group slot with a stable counting sort, which also produces the header counts. The id→slot lookup is a linear scan over a small array that stays in L1, so the only cache misses are one per brick, which the draw loop pays anyway.

```cpp
template<typename Brick> struct LogicSlots {
  blender::Vector<Brick *, 64> order; /* Bricks sorted by slot, stable within a slot. */
  blender::Vector<int, 17> start;     /* Slot s = order[start[s], start[s + 1]). */
  blender::Vector<int, 17> visible;   /* Members on a visible layer, per slot (header counts). */

  blender::Span<Brick *> slot(const int s) const
  {
    return order.as_span().slice(start[s], start[s + 1] - start[s]);
  }
};

template<typename Brick>
static LogicSlots<Brick> logic_bucket_by_group(const Object *ob,
                                               const ListBaseT<Brick> &lb,
                                               const uint32_t view)
{
  blender::Vector<uint16_t, 16> ids; /* Slot k <-> ids[k]; slot ids.size() = ungrouped (D12). */
  for (const bLogicGroup *g = ob->logic_groups.first(); g; g = g->next) {
    ids.append(g->id);
  }
  const int ungrouped = ids.size();
  auto slot_of = [&](const uint16_t gid) -> int {
    for (int k = 0; gid && k < ids.size(); k++) {
      if (ids[k] == gid) {
        return k;
      }
    }
    return ungrouped; /* gid == 0, or a dangling id (cannot happen, D9). */
  };

  LogicSlots<Brick> r;
  blender::Vector<int, 64> slot;
  r.start.append_n_times(0, ids.size() + 2);
  r.visible.append_n_times(0, ids.size() + 1);
  for (Brick *b = lb.first(); b; b = b->next) {
    const int s = slot_of(b->group_id);
    slot.append(s);
    r.start[s + 1]++;
    r.visible[s] += (logic_mask_eff(b->logic_mask) & view) != 0;
  }
  for (int s = 1; s < r.start.size(); s++) {
    r.start[s] += r.start[s - 1];
  }
  r.order.resize(slot.size());
  blender::Vector<int, 17> cursor(r.start.as_span().drop_back(1));
  int i = 0;
  for (Brick *b = lb.first(); b; b = b->next, i++) {
    r.order[cursor[slot[i]]++] = b;
  }
  return r;
}
```

### 5.4 Layer Tab Row (D14)

The layer tabs live in their own row directly under the editor header, spanning the full area width. The row is a `RGN_TYPE_TOOL_HEADER` region, the same region kind the 3D Viewport uses for its tool settings bar. It is drawn in C, without a Python header type, so `rna_ui_api.cc` is not touched.

- **Region:**
  - Registered in `ED_spacetype_logic()` with `regionid = RGN_TYPE_TOOL_HEADER`, `prefsizey = HEADERY` and the header keymap flags.
  - `logic_new()` inserts it right after the header, so it spans the whole area above the sidebar.
  - Old files get it on demand in `logic_init()`, as `logic_has_buttons_region()` already does for the sidebar.
  - *View › Layer Tabs* toggles it through `show_region_tool_header` (`rna_def_space_generic_show_region_toggles()`).
- **Tabs:** one `ButtonType::Tab` per shown layer. This is the widget of Blender's workspace tabs (`template_ID_tabs()`).
  - With `BUT_ALIGN_DOWN` the bottom corners are square and the active tab merges with the editor below, which gives the browser look.
  - `rnapoin`/`rnaprop` point to `SpaceLogicEditor.active_logic_layer`, a pointer property.
  - `custom_data` points to `&scene->gm.logic_layers[i]`.

| State | Look | Source |
|---|---|---|
| Active (`act_layer`) | Pushed; merges with the region below | Pointer property equals `custom_data` (native Tab behaviour) |
| Visible (in `view_mask`) | Normal | — |
| Hidden | Greyed text | `UI_BUT_INACTIVE` |

- **Which tabs are shown:** layers that are named, in `view_mask`, in `used_mask`, or equal to `act_layer`, in index order. The label is the scene name, or "Layer N" when the name is empty. A trailing "+" shows the first layer not yet shown.
- **Clicking:** handled by a button func that reads Shift from `win->runtime->eventstate`, as `handle_layer_buttons()` does.
  - Plain click: that layer only, and it becomes active.
  - Shift+click: toggles that layer in `view_mask`.
  - `LOGIC_OT_layer_view` exposes the same helper to keymaps and Python.
- **Rules:** `view_mask` never becomes empty, so turning off the last visible layer is ignored. A layer that becomes visible becomes `act_layer`. If `act_layer` is turned off, `act_layer = ctz(view_mask)`. No undo push, because this is view state.
- **Renaming:** double-click or Ctrl+click on a tab enters text editing natively; `do_but_TAB()` does this for Tab buttons with a pointer property and `custom_data`. The text is written to `LogicLayer.name` through its RNA name property, so no N-panel is needed.
- **Keeping the row current:** the main region writes `used_mask` after the pre-pass. Only when the value changed does it call `ED_area_tag_redraw_regiontype(area, RGN_TYPE_TOOL_HEADER)`, so the row redraws only on a change.

### 5.5 Brick Header Additions

Each brick header gets three icon buttons:

- **Grip:** drag source (§8.2).
- **Trace:** toggles `*_HILITE_SRC` (§7).
- **Organize:** popover `LOGIC_PT_brick_organize` (Python). It shows the layer grid via `template_layers(brick, "layers", space, "used_layers", 1 << act_layer)`, which gives click-exclusive / Shift-extend and used/active icons for free, plus the group dropdown and "New Group". The popover reads the brick from the context pointer already set per brick (`context.sensor` / `.controller` / `.actuator`).

New bricks get `logic_mask = 1u << act_layer` when added from a Logic Editor. Added from Python or another editor, they get 0 (layer 0).

The header width at `min_col` must be checked during implementation. If the three icons do not fit, the up/down arrows move into the organize popover.

---

## 6. Links

### 6.1 Stubs (D2-B)

| Button | Flag set when |
|---|---|
| Sensor `Link` | `sensor_link_scan().hidden_out` |
| Controller `Inlink` | `CONT_LINK_HIDDEN_IN` |
| Controller `Link` | any linked actuator has `ACT_HIDDEN` |
| Actuator `Inlink` | `ACT_LINK_HIDDEN_IN` |

- **Shape:** a dashed segment of 1.5 UI units running from the icon edge toward the hidden side (+X for `Link`, −X for `Inlink`), ending in a short vertical tick.
- **Offset (D16):** the stub sits 0.35 UI units below the icon axis. A visible wire leaving the same icon starts horizontally, because the Bézier tangent at the icon is horizontal, so on the axis the solid wire would cover the dashed stub.
- **Count:** one stub per button, regardless of how many hidden links it has.
- **When stubs appear:** only for hiding by layer or collapsed group among displayed objects. Links hidden by state rules or pointing to non-displayed objects keep the current behaviour (no line).

### 6.2 `upbgeflag` Bits

```cpp
/* UI_interface_c.hh — Button::upbgeflag (UPBGE section). */
enum {
  UI_BUT_SCA_LINK_GREY = 1 << 0,       /* Existing. */
  UI_BUT_SCA_LINK_HIDDEN_OUT = 1 << 1, /* Link button: draw outgoing stub. */
  UI_BUT_SCA_LINK_HIDDEN_IN = 1 << 2,  /* Inlink button: draw incoming stub. */
  UI_BUT_SCA_LINK_HILITE = 1 << 3,     /* Brick is on a traced path. */
  UI_BUT_SCA_LINK_DIM = 1 << 4,        /* Trace active and brick not on a path. */
};
/* Bits 8..11: controller type + 1 on the controller's Link/Inlink buttons (0 elsewhere).
 * Every line has exactly one controller endpoint, so OR-ing both endpoints yields its type. */
#define UI_BUT_SCA_LINK_CONT_SHIFT 8
#define UI_BUT_SCA_LINK_CONT_MASK (0xF << UI_BUT_SCA_LINK_CONT_SHIFT)
```

### 6.3 Wire Rendering

`ui_draw_links()` currently binds a program and issues one draw per wire, in three passes. v2 classifies each line once and appends its Bézier segments to one of three vertex streams.

| Order | Class | Width | Members | Color |
|---|---|---|---|---|
| 1 | DIM | 1.0 px | deactivated (`GREY`) or either endpoint `DIM` | type color, alpha 0.25 (deactivated) / 0.15 (dimmed) |
| 2 | NORMAL | 1.5 px | everything else | type color; selected lines keep their current grey |
| 3 | HILITE | 2.5 px | both endpoints `HILITE`, or hovered | `TH_HILITE`; hovered keeps `TH_TEXT_HI` |

- **Wire draws:** each class is one `GPU_SHADER_3D_POLYLINE_FLAT_COLOR` draw with `GPU_PRIM_LINES`. The vertex is a 2×f32 position plus a packed 4×u8 colour (12 B). Order 1→3 preserves today's layering, with hovered lines on top.
- **Stubs:** one `GPU_SHADER_3D_LINE_DASHED_UNIFORM_COLOR` draw.
- **Total:** at most 4 draws per redraw, against one per wire today.
- **Vertex budget:** `2 × LINK_RESOL` (48) vertices per wire, so 1000 wires stream about 576 KB per redraw. Optional: `segments = clamp(length_px / 8, 6, 24)`.
- **Theme lookups:** the 8 type colours are resolved once per `ui_draw_links()` call, not per line.

```cpp
/* Controller type (CONT_* value 0..7) -> theme slot of bTheme::tlogic. */
static const int logic_wire_theme_id[8] = {
    TH_SYNTAX_L, /* CONT_LOGIC_AND   */
    TH_SYNTAX_S, /* CONT_LOGIC_OR    */
    TH_SYNTAX_B, /* CONT_EXPRESSION  */
    TH_SYNTAX_N, /* CONT_PYTHON      */
    TH_SYNTAX_V, /* CONT_LOGIC_NAND  */
    TH_SYNTAX_C, /* CONT_LOGIC_NOR   */
    TH_SYNTAX_D, /* CONT_LOGIC_XOR   */
    TH_SYNTAX_R, /* CONT_LOGIC_XNOR  */
};
```

Deactivated lines currently use `TH_GRID`, which the factory theme does not define for `tlogic` (alpha 0). They are invisible today; see Appendix A.4.

---

## 7. Path Highlighting (Trace)

### 7.1 Semantics

| Source | Highlighted |
|---|---|
| Sensor S | S, the controllers S feeds, the actuators of those controllers (downstream) |
| Controller C | the sensors feeding C, C, the actuators of C |
| Actuator A | A, the controllers driving A, the sensors feeding those controllers (upstream) |

Several sources produce the union of their paths. A brick is highlighted when it has `SENS_HILITE_SRC | SENS_IN_PATH`, `CONT_PATH_DOWN | CONT_PATH_UP`, or `ACT_HILITE_SRC | ACT_IN_PATH`. A wire is highlighted when both endpoints are; with several sources this can over-highlight an edge shared by two traces, which is accepted. Links are followed across objects.

### 7.2 Interaction

- **Starting a trace:** the trace icon calls `LOGIC_OT_brick_trace(object, kind, name)`, which toggles the source bit.
- **Clearing:** Esc in the main region calls `LOGIC_OT_trace_clear`, which clears source bits on displayed objects.
- **Undo:** neither operator pushes undo.
- **Persistence:** source bits are saved with the file, which is harmless because the converter ignores them.
- **Editing during a trace:** editing fields does not clear the trace.

### 7.3 Rendering (D7-A)

Spans are recorded only while a trace is active. For each drawn brick, `logic_buttons()` stores `{but_first, but_end, hilite}` as indices into `block->buttons_ptrs`, captured before and after the brick's widgets are created. Indices, unlike pointers, survive the old/new button matching in `block_end()`. A span's rectangle is the union of its button rectangles (`logic_window.cc` already includes `interface_intern.hh`).

```cpp
/* logic_buttons(), tail */
UI_block_links_compose(block);
blender::ui::block_end(C, block);
if (pre.any_src) {
  logic_draw_trace_glow(block, spans); /* Behind the bricks: 1 draw. */
}
block_draw(C, block);                  /* Buttons, then links (batched, §6.3). */
if (pre.any_src) {
  logic_draw_trace_dim(block, spans);  /* On top: 1 draw. */
}
```

- **Glow (D13):** each highlighted span gets two rounded outlines through the existing widget shader (`draw_roundbox_4fv_ex()`, which anti-aliases the curve): a translucent halo, then a crisp border. Both sit just outside the span rectangle, so the brick's own box, drawn afterwards, cannot cover them. The corner radius continues the brick box radius (`wcol_box.roundness * U.widget_unit`; zoom is locked at 1.0, so `widget_radius_from_zoom()` reduces to that).

```cpp
/* logic_window.cc. Two widget-shader draws per highlighted brick, only while tracing. */
static void logic_draw_trace_glow(const Block *block, const Span<BrickSpan> spans)
{
  const float px = UI_SCALE_FAC;
  const float rad = ui::theme::theme_get()->tui.wcol_box.roundness * U.widget_unit;
  float border[4], halo[4];
  ui::theme::get_color_4fv(TH_HILITE, border);
  copy_v4_v4(halo, border);
  halo[3] = 0.35f;

  draw_roundbox_corner_set(CNR_ALL);
  for (const BrickSpan &s : spans) {
    if (!s.hilite) {
      continue;
    }
    const rctf r = logic_span_rect(block, s); /* Union of the span's button rectangles. */
    rctf ring = r;
    /* Halo: band [r + 1.5 px, r + 4.5 px]. The outline is drawn inward from the rectangle edge. */
    BLI_rctf_pad(&ring, 4.5f * px, 4.5f * px);
    draw_roundbox_4fv_ex(&ring, nullptr, nullptr, 0.0f, halo, 3.0f * px, rad + 4.5f * px);
    /* Border: band [r, r + 1.5 px], full opacity. */
    ring = r;
    BLI_rctf_pad(&ring, 1.5f * px, 1.5f * px);
    draw_roundbox_4fv_ex(&ring, nullptr, nullptr, 0.0f, border, 1.5f * px, rad + 1.5f * px);
  }
}
```

- **Dim:** one `GPU_SHADER_3D_UNIFORM_COLOR` triangle draw covers every non-highlighted span with `TH_BACK` at alpha 0.85, so those bricks read at about 15 %. The quads stay square: where they overhang the rounded corners they paint background over background, which is invisible.
- **Cost:** zero without a trace. With a trace: an `O(buttons)` union, `2 × H` widget draws (H = highlighted bricks) and 1 dim draw. A trace typically covers a handful of bricks.

---

## 8. Interaction & Operators

### 8.1 Operators

| Operator | Impl. | Undo | Notes |
|---|---|---|---|
| `LOGIC_OT_layer_view` | C | no | `index`, `extend` (Shift in `invoke`); the tab buttons call the same helper directly (§5.4) |
| `LOGIC_OT_brick_trace` | C | no | `object`, `kind`, `name` |
| `LOGIC_OT_trace_clear` | C | no | Esc in the main region |
| `LOGIC_OT_brick_move_to` | C | yes | Drop target of drag-and-drop; `object`, `kind`, `name`, `target`, `before`; adopts the target's group |
| `LOGIC_OT_{sensor,controller,actuator}_add` | C (existing) | yes | `logic_mask = 1u << act_layer` when called from a Logic Editor |
| `LOGIC_OT_{sensor,controller,actuator}_move` | C (existing) | yes | Neighbour rule updated (§8.2) |
| `logic.group_add`, `logic.group_remove`, `logic.group_assign` | Python | yes | Thin wrappers over RNA (§9.1). Renaming and recolouring happen in the group header; Delete is in its "⋯" menu (§5.3) |
| `logic.bricks_export`, `logic.bricks_import`, `logic.bricks_copy`, `logic.bricks_paste` | Python | import/paste | §9 |

### 8.2 Reordering

**Up/down arrows.** The neighbour rule in `sca.cc` becomes the following (the move never crosses hidden bricks or group boundaries):

| Kind | Neighbour `tmp` qualifies when |
|---|---|
| Sensor | `(tmp->flag & SENS_VISIBLE) && tmp->group_id == s->group_id` |
| Controller | `(tmp->state_mask & c->state_mask) && !(tmp->flag & CONT_HIDDEN) && tmp->group_id == c->group_id` |
| Actuator | `(tmp->flag & ACT_VISIBLE) && !(tmp->flag & ACT_HIDDEN) && tmp->group_id == a->group_id` |

**Drag-and-drop.**

- **Source:** the grip icon calls `button_drag_set_rna(but, &brick_ptr)`, producing a `WM_DRAG_RNA` drag. The standard `U.drag_threshold` applies; there is no timer.
- **Dropbox:** on the main region. The poll accepts only a brick of the same kind (same column) and the same owner object, as the brick under the cursor.
- **Target:** the brick under the cursor, found through the button under the mouse and its context pointer (`"sensor"` / `"controller"` / `"actuator"`, already set per brick). Dropping on a group header (`"logic_group"` context pointer) inserts before the group's first member.
- **Before/after:** decided by the cursor Y against the centre of the target's span. The span is computed on demand, only during the drag.
- **Exec:** `BLI_remlink` + `BLI_insertlinkbefore/after`; the moved brick adopts the target's `group_id`. Properties are names, so redo is safe.
- **Feedback:** the dropbox `draw_in_view` callback draws a 2 px `TH_HILITE` line at the insertion edge.

---

## 9. Asset IO (D6-A)

### 9.1 RNA Prerequisites

| RNA | Addition |
|---|---|
| `GameObjectSettings.sensors` / `.controllers` / `.actuators` | `new(type, name)`, `remove(brick)` — no undo push (the calling operator owns undo) |
| `GameObjectSettings.logic_groups` | Collection of `LogicGroup` with `new(name)` / `remove(group)` |
| `LogicGroup` | `name`, `color`, `show_expanded` (inverse of `LOGIC_GROUP_COLLAPSED`), `id` (read-only) |
| `Sensor` / `Controller` / `Actuator` | `layers`: bool[30], `PROP_LAYER_MEMBER` over `logic_mask`; the getter returns the effective mask, and an all-false set stores layer 0. `logic_group`: dynamic enum (owner's groups + None) over `group_id` |
| `SpaceLogicEditor` | `view_layers` bool[30] (`view_mask`), `active_layer` int, `used_layers` bool[30] read-only, `active_logic_layer` pointer to the scene's `LogicLayer` at `act_layer` (setter derives the index from the pointer; drives the tab state, §5.4), `show_region_tool_header` |
| `SceneGameData` | `logic_layers`: fixed-length collection of `LogicLayer {name}` (array iterator); `name` is the struct's name property, which tab renaming requires |
| `ThemeLogicEditor` | `path_highlight`, `wire_and`, `wire_or`, `wire_nand`, `wire_nor`, `wire_xor`, `wire_xnor`, `wire_python`, `wire_expression` (§10) |

The bodies go in the UPBGE-owned `rna_sensor_api.cc`, `rna_controller_api.cc` and `rna_actuator_api.cc`. `rna_object.cc` only gains one `RNA_def_property_srna()` line per collection.

### 9.2 JSON Format (version 1)

```json
{
  "format": "upbge.logic_bricks",
  "format_version": 1,
  "upbge_version": [0, 54, 0],
  "scope": "LAYERS",
  "layers": [2],
  "groups": [
    {"id": 1, "name": "PlayerJump", "color": [255, 128, 0], "collapsed": false}
  ],
  "sensors": [
    {"name": "Space", "type": "KEYBOARD", "logic_mask": 4, "group": 1,
     "links": ["And"],
     "props": {"key": "SPACE", "use_pulse_true_level": false}}
  ],
  "controllers": [
    {"name": "And", "type": "LOGIC_AND", "logic_mask": 4, "group": 1,
     "links": ["Jump", "Dust", ["Camera", "Shake"]],
     "props": {"states": 1, "use_priority": false}}
  ],
  "actuators": [
    {"name": "Jump", "type": "MOTION", "logic_mask": 4, "group": 1,
     "props": {"offset_location": [0.0, 0.0, 0.2]}},
    {"name": "Dust", "type": "EDIT_OBJECT", "logic_mask": 4, "group": 1,
     "props": {"mode": "ADDOBJECT",
               "object": {"$id": ["OBJECT", "DustPuff", null]}}}
  ]
}
```

- **Scopes:** `OBJECT` (all bricks of the active object), `LAYERS` (bricks with `eff(mask) & view_mask`), `GROUP` (one group), `BRICK` (one brick).
- **Link entries:** a brick name in the same object, or `[object, brick]` for a cross-object link.
- **`props`:** produced by RNA reflection over the concrete subclass. It skips `rna_type`, `name`, `type`, read-only non-struct properties, the link collections, `layers` and `logic_group`, which are stored explicitly.
- **Value encoding:** enums are stored by identifier (flag enums as sorted lists), arrays as lists, nested non-ID structs recursively. ID pointers are stored as `{"$id": [id_type, name, library_filepath | null]}`.
- **Group ids** are local to the file.

### 9.3 Import

1. Parse, then check `format` and `format_version`.
2. Create the groups, mapping file id → new id (D9 allocation).
3. Create every brick with `new(type, name)`. Type goes first, because the RNA `type` setters re-initialise `data` (`BKE_sca_init_sensor()` etc.). Keep a `json_name → brick` map; names may be uniquified, so nothing is looked up by name after creation.
4. Set `props` in RNA definition order. ID references are resolved through one dictionary per ID type, built lazily from `bpy.data`, keyed `(name, library)`. This is `O(ids + refs)`. A miss leaves the property unset and adds a warning.
5. Create links through the brick map. Cross-object references are resolved by object name; unresolved ones are dropped with a warning.
6. Apply layers. `LAYERS`-scoped files are remapped to `act_layer` unless `keep_layers=True`; `OBJECT`-scoped files keep their masks.
7. Finish with one undo step (`bl_options = {'REGISTER', 'UNDO'}`), one `NC_LOGIC` notifier, and a summary report with the warnings.

### 9.4 Clipboard

`logic.bricks_copy` writes the same JSON to `context.window_manager.clipboard`. `logic.bricks_paste` validates `format` and runs §9.3. Copy and paste between two UPBGE instances therefore comes for free. The parse costs a few kilobytes of work per user action, which is irrelevant at interaction rate (D10).

### 9.5 Serialisation Sketch

```python
# scripts/startup/bl_operators/logic_bricks_io.py (new; one line in bl_operators/__init__.py)
import bpy

_SKIP = {"rna_type", "name", "type", "controllers", "actuators", "layers", "logic_group"}

def _ser(struct):
    out = {}
    for p in struct.bl_rna.properties:
        k = p.identifier
        if k in _SKIP or (p.is_readonly and p.type not in {'POINTER', 'COLLECTION'}):
            continue
        v = getattr(struct, k)
        if p.type == 'POINTER':
            if v is None:
                out[k] = None
            elif isinstance(v, bpy.types.ID):
                out[k] = {"$id": [v.id_type, v.name, v.library.filepath if v.library else None]}
            else:
                out[k] = _ser(v)
        elif p.type == 'COLLECTION':
            out[k] = [_ser(e) for e in v]
        elif p.type == 'ENUM':
            out[k] = sorted(v) if p.is_enum_flag else v
        elif getattr(p, "is_array", False):
            out[k] = list(v)
        else:
            out[k] = v
    return out
```

---

## 10. Theme

The design reuses free slots of `bTheme::tlogic` (a `ThemeSpace`). The `syntax*` slots are generic in `resources.cc`, and the node editor already repurposes them the same way. No DNA change is needed.

| Slot | RNA (`ThemeLogicEditor`) | Default |
|---|---|---|
| `hilite` | `path_highlight` | `#FF8000` |
| `syntaxl` | `wire_and` | `#4A90D9` |
| `syntaxs` | `wire_or` | `#5BA85A` |
| `syntaxv` | `wire_nand` | `#6B8FA8` |
| `syntaxc` | `wire_nor` | `#7A9E79` |
| `syntaxd` | `wire_xor` | `#3FAFAF` |
| `syntaxr` | `wire_xnor` | `#6E9E9E` |
| `syntaxn` | `wire_python` | `#D4A843` |
| `syntaxb` | `wire_expression` | `#9B72CF` |

The work touches three places:

- **Defaults:** `.tlogic` in `release/datafiles/userdef/userdef_default_theme.c`.
- **RNA:** `rna_def_userdef_theme_space_logic()` in `rna_userdef.cc`.
- **Versioning:** in `do_versions_theme()` (`versioning_userdef.cc`), a UPBGE block `if (!MAIN_VERSION_UPBGE_ATLEAST(userdef, 54, N)) { FROM_DEFAULT_V4_UCHAR(tlogic.hilite); … }` for all nine slots, with N being the next `UPBGE_FILE_SUBVERSION`. Without it, existing users read `#00000000` and see invisible highlights and wires.

---

## 11. Performance

### 11.1 Per-Redraw Cost Model

N = displayed bricks, L = links, B = buttons in the block, G = groups per object.

| Stage | Current | v2 |
|---|---|---|
| Object gathering (`get_selected_and_linked_obs`) | fixed-point loop over all objects in `bmain` | unchanged (out of scope) |
| Visibility flags | `O(N + L)` | `O(N + L)`, plus one sensor link walk only for upstream traces |
| Sensor "is linked" test | `O(links × B)` block search | `O(links)` flag read |
| Link resolution (`UI_block_links_compose`) | `O(L × B)` | `O(B + L)` hash map |
| Group bands | — | Bucketing `O(N × G)` ALU on L1 data + one miss per brick; 3 layout resolves per band; 2 widget draws per drawn group (frame) |
| Wires | one program bind + draw per wire | ≤ 4 draws |
| Trace glow/dim | — (v1: 5 draws per highlighted brick) | `2 × H + 1` draws (H = highlighted bricks), only while tracing |

### 11.2 Measured: Link Resolution (Synthetic)

Model of `UI_block_links_compose()` with 12 buttons per brick and 1.2 links per brick, `-O2`, one run:

| Bricks | Buttons | Linear (current) | Hash map |
|---|---|---|---|
| 100 | 1 200 | 0.07 ms | 0.009 ms |
| 500 | 6 000 | 1.8 ms | 0.04 ms |
| 1 500 | 18 000 | 15.9 ms | 0.13 ms |

The real cost is roughly double, because `is_sensor_linked()` repeats the same search for every sensor.

### 11.3 Acceptance

- **Phase 0:** `SCOPED_TIMER_AVERAGED("logic_buttons")` (`BLI_timeit.hh`), run on scenes generated by Appendix B with 100, 500 and 1500 bricks per column; report before and after.
- **Phases 2–6:** no measurable regression of `logic_buttons()` on the same scenes while layers, groups and trace are unused (all bricks on layer 0, no groups, no trace).

---

## 12. Implementation Roadmap

| Phase | Scope | Files | Depends on |
|---|---|---|---|
| **0 — Hot path & bugs** | A.1 sensor link test via `CONT_STATE_VISIBLE` + cross-object clear fix; A.2 hash-map link resolution; A.3 controller move-down; A.4 invisible deactivated wires | `logic_window.cc`, `interface.cc`\*, `sca.cc`, `DNA_controller_types.h` | — |
| **1 — Data** | DNA fields and structs (§3); write/read/free/copy; RNA (§9.1 except brick `new()`) | `DNA_{sensor,controller,actuator,object,scene,space}_types.h`, `object.cc`\*, `sca.cc`, `object_edit.cc`\*, `rna_*.cc` | — |
| **2 — Layout** | `V2D_COMMONVIEW_LIST`, dynamic columns (§5.1–5.2) | `space_logic.cc`, `logic_window.cc` | — |
| **3 — Layers** | Pre-pass (§4.2), tab row region and Tab buttons (§5.4), `layer_view`, organize popover, stubs (§6.1) | `space_logic.cc`, `logic_window.cc`, `logic_ops.cc`, `interface.cc`\*, `UI_interface_c.hh`\*, `rna_space.cc`\*, `space_logic.py` | 0, 1 |
| **4 — Groups** | Bands, full-width group header and frame, bucketing (§5.3), collapse, group operators, drag-and-drop (§8.2) | `logic_window.cc`, `logic_ops.cc`, `sca.cc`, `space_logic.py`, `bl_operators/logic_bricks_io.py` | 1, 2, 3 |
| **5 — Trace & wires** | Trace bits and operators, glow/dim (§7.3), batched wires (§6.3), theme (§10) | `logic_window.cc`, `logic_ops.cc`, `interface.cc`\*, `interface_widgets.cc`\*, `rna_userdef.cc`\*, `userdef_default_theme.c`\*, `versioning_userdef.cc`\* | 3 |
| **6 — IO** | Brick `new()/remove()`, JSON export/import, clipboard (§9) | `rna_{sensor,controller,actuator}_api.cc`, `rna_object.cc`\*, `bl_operators/logic_bricks_io.py` (new), `bl_operators/__init__.py`\* | 1 (4 for groups) |

\* Blender file. In every case the touched code is already UPBGE-specific (link buttons, SCA read/write, `GameObjectSettings` RNA, `tlogic` theme, UPBGE versioning block) or the change is a one-line registration.

---

## 13. Open Decisions

None. The four questions left open in the first draft of v2 are recorded as D10–D13 in §2.

---

## Appendix A — Phase 0 Patches

### A.1 Sensor Link Test and Cross-Object Flag Clearing

Current bug: the loop at the top of `logic_buttons()` clears `ACT_LINKED | ACT_VISIBLE` per displayed object and marks actuators in the same iteration. When a controller of `idar[0]` drives an actuator of `idar[1]`, the second iteration erases the mark. That actuator is then treated as unlinked and is shown even when its controller's state is hidden.

```cpp
/* Transient bits are recomputed every redraw. They are cleared here because set_sca_ob()
 * already visits every controller/actuator in bmain and writes `mynew`, which shares the
 * cache line with `flag`. */
static void set_sca_ob(Object *ob)
{
  for (bController *cont = ob->controllers.first(); cont; cont = cont->next) {
    cont->mynew = (bController *)ob;
    cont->flag &= ~CONT_STATE_VISIBLE; /* Phase 3: ~CONT_TRANSIENT_MASK */
  }
  for (bActuator *act = ob->actuators.first(); act; act = act->next) {
    act->mynew = (bActuator *)ob;
    act->flag &= ~(ACT_LINKED | ACT_VISIBLE); /* Phase 3: ~ACT_TRANSIENT_MASK */
  }
}
```

```cpp
/* logic_buttons(): replaces the per-object clear + mark loop. */
for (a = 0; a < count; a++) {
  ob = (Object *)idar[a];
  const bool cont_col = (ob->scavisflag & OB_VIS_CONT) && (ob->scaflag & OB_SHOWCONT);
  for (bSensor *s = ob->sensors.first(); s; s = s->next) {
    s->flag &= ~SENS_VISIBLE;
  }
  for (bController *c = ob->controllers.first(); c; c = c->next) {
    const bool in_state = (ob->scaflag & OB_ALLSTATE) || (ob->state & c->state_mask);
    /* Same conditions under which the controller column draws this controller's Inlink. */
    c->flag |= (cont_col && in_state) ? CONT_STATE_VISIBLE : 0;
    const short mark = ACT_LINKED | (in_state ? ACT_VISIBLE : 0);
    for (int i = 0; i < c->totlinks; i++) {
      if (c->links[i]) {
        c->links[i]->flag |= mark;
      }
    }
  }
}

/* O(links) instead of a search over every button of the block per link. */
static bool is_sensor_linked(const bSensor *sens)
{
  for (int i = 0; i < sens->totlinks; i++) {
    const bController *cont = sens->links[i];
    if (cont && (cont->flag & CONT_STATE_VISIBLE)) {
      return true;
    }
  }
  return false;
}
```

`get_selected_and_linked_obs()` (and therefore `set_sca_ob()`) is also called from the three header menu handlers. Each of those is followed by a redraw that recomputes the bits, so clearing there is harmless.

### A.2 Hash-Map Link Resolution

```cpp
/* interface.cc */
void UI_block_links_compose(Block *block)
{
  /* Index Inlink buttons once. The previous per-link linear scan over all buttons was
   * O(links * buttons): ~1.8 ms at 500 bricks, ~16 ms at 1500 (§11.2).
   * Map::add() keeps the first insertion, matching the old first-match semantics. */
  blender::Map<const void *, Button *> inlinks;
  for (const std::unique_ptr<Button> &b : block->buttons_ptrs) {
    if (b->type == ButtonType::Inlink) {
      inlinks.add(b->poin, b.get());
    }
  }

  for (const std::unique_ptr<Button> &b : block->buttons_ptrs) {
    Button *but = b.get();
    uiLink *link = but->link;
    if (but->type != ButtonType::Link || link == nullptr) {
      continue;
    }
    const bool grey = (but->upbgeflag & UI_BUT_SCA_LINK_GREY) != 0;
    if (link->ppoin) {
      for (int a = 0; a < *link->totlink; a++) {
        if (Button *bt = inlinks.lookup_default((*link->ppoin)[a], nullptr)) {
          ui_linkline_add(&link->lines, but, bt, grey || (bt->upbgeflag & UI_BUT_SCA_LINK_GREY));
        }
      }
    }
    else if (link->poin) {
      if (Button *bt = inlinks.lookup_default(*link->poin, nullptr)) {
        ui_linkline_add(&link->lines, but, bt, grey || (bt->upbgeflag & UI_BUT_SCA_LINK_GREY));
      }
    }
  }
}
```

### A.3 `BKE_sca_move_controller()` Move-Down

When no later controller shares a state, `tmp` is `nullptr`, and `BLI_insertlinkafter(lb, nullptr, cont)` inserts at the **head**: the last controller of a state jumps to the top of the list. Guard it as the move-up branch already does:

```cpp
  else if (val == 2 && cont->next) {
    tmp = cont->next;
    while (tmp) {
      if (tmp->state_mask & cont->state_mask) {
        break;
      }
      tmp = tmp->next;
    }
    if (tmp) {
      BLI_remlink(&ob->controllers, cont);
      BLI_insertlinkafter(&ob->controllers, tmp, cont);
    }
  }
```

### A.4 Invisible Deactivated Wires

`ui_draw_linkline()` colours lines to or from deactivated bricks with `TH_GRID`. With the factory theme, `tlogic.grid` is not set, so it reads `#00000000`, and the draw uses alpha blending. Interim fix until Phase 5:

```cpp
  if (dashInactiveLines) {
    rgba_float_args_set_ch(color, 100, 100, 100, 90); /* Regular wire grey at ~35 % alpha. */
  }
```

---

## Appendix B — Stress Scene Generator

```python
# Run once in a new file with one object selected. Uses the add operators (one undo push
# each) until the RNA new() functions of §9.1 exist, so large N takes a few seconds.
import bpy

N = 500  # bricks per column
ob = bpy.context.active_object
ops = bpy.ops.logic
for i in range(N):
    ops.sensor_add(type='ALWAYS', name=f"S{i}", object=ob.name)
    ops.controller_add(type='LOGIC_AND', name=f"C{i}", object=ob.name)
    ops.actuator_add(type='MOTION', name=f"A{i}", object=ob.name)

g = ob.game
for i in range(N):
    s, c, a = g.sensors[i], g.controllers[i], g.actuators[i]
    c.link(sensor=s, actuator=a)
    if i % 5 == 0 and i + 1 < N:  # ~1.2 links per brick
        g.controllers[i + 1].link(sensor=s)
```
