# EXP Vector Types — Implementation Structure
## `EXP_Vector2Value` / `EXP_Vector3Value` / `EXP_Vector4Value`
### UPBGE — Expressions Module

---

## 1. Overview

This document describes the complete structure of the vector type extension
for the UPBGE Expressions system. It covers file layout, class hierarchy,
storage decisions, the Moto include chain, operator dispatch, Python
integration, and the exact changes required to existing files.

**Scope:** `source/gameengine/Expressions/` only.  
**Blender-side changes:** none.  
**Primary constraint:** performance in the game loop.

---

## 2. Storage Decision — All Three Classes Use Moto Types

All three value classes store their data using the Moto math library:

| Class              | Storage     |
|--------------------|-------------|
| `EXP_Vector2Value` | `MT_Vector2` |
| `EXP_Vector3Value` | `MT_Vector3` |
| `EXP_Vector4Value` | `MT_Vector4` |

### Why not `float[N]`

A uniform `float[N]` approach was evaluated and rejected for the following
reasons:

- **`MT_Vector2`, `MT_Vector3`, and `MT_Vector4` all exist** in the Moto
  library and expose an identical API surface: `getValue()`,
  `operator-()`, `operator*(scalar)`, `operator+`, `operator-`,
  `length()`, `length2()`, `fuzzyZero()`, and `setValue()`.
  There is no missing type that would force a fallback to raw arrays.

- **Consistency:** using the same family of types across all three classes
  means a single contract for any generic code that handles vectors
  (serialization, tweening, runtime binding). A mixed approach — MT for
  Vector3, float[] for the others — would require type-specific branches
  everywhere.

- **MT methods replace manual implementations:** `MT_fuzzyEqual()`,
  `MT_fuzzyZero()`, `length2()`, and `fuzzyZero()` already implement
  exactly the behaviour needed for `==`, `!=`, `!v`, and `GetLengthSq()`.
  Using `float[N]` would require reimplementing these correctly.

- **`getValue()` returns `const MT_Scalar*`** for all three types,
  which maps directly to `const float*` and is accepted without casting
  by `Vector_CreatePyObject` for the Vector3 Python path.

### Include chain

```
MT_Vector4.h
  └── MT_Vector3.h          (included by MT_Vector4.h)
        └── MT_Vector2.h    (included by MT_Vector3.h)
```

Including `MT_Vector4.h` transitively provides all three types. The
`EXP_PyObjectPlus.h` header already includes `MT_Vector3.h` (and
therefore `MT_Vector2.h`), so `EXP_Vector4Value.h` is the only header
that adds a new include to the chain.

---

## 3. File Layout

```
source/gameengine/Expressions/
│
│  ── Existing files (unchanged except EXP_Value.h) ──
│
├── EXP_Value.h                  ← VALUE_DATA_TYPE enum modified (see §5)
├── EXP_BoolValue.h / .cpp
├── EXP_FloatValue.h / .cpp
├── EXP_IntValue.h / .cpp
├── EXP_StringValue.h / .cpp
├── EXP_ErrorValue.h / .cpp
├── EXP_EmptyValue.h / .cpp
├── EXP_PyObjectPlus.h / .cpp    ← already includes MT_Vector3.h
│
│  ── New files ──
│
├── EXP_Vector2Value.h / .cpp
├── EXP_Vector3Value.h / .cpp
└── EXP_Vector4Value.h / .cpp
```

### CMakeLists.txt

Add the following 6 entries to the existing `set(SRC ...)` block in
`source/gameengine/Expressions/CMakeLists.txt`:

```cmake
EXP_Vector2Value.cpp
EXP_Vector3Value.cpp
EXP_Vector4Value.cpp
EXP_Vector2Value.h
EXP_Vector3Value.h
EXP_Vector4Value.h
```

No new library dependencies are introduced. Moto and mathutils are already
linked in existing targets.

---

## 4. Class Hierarchy

```
EXP_PyObjectPlus
  └── EXP_Value
        └── EXP_PropValue
              ├── EXP_Vector2Value   MT_Vector2
              ├── EXP_Vector3Value   MT_Vector3
              └── EXP_Vector4Value   MT_Vector4
```

All three are leaf nodes — no further subclassing is intended.

---

## 5. Changes to Existing Files

### 5.1 `EXP_Value.h` — `VALUE_DATA_TYPE` enum

This is the **only** existing file that requires modification.

The pre-existing `VALUE_VECTOR_TYPE` slot was unused in all scanned `.cpp`
files. It is replaced by three explicit subtypes, matching the pattern of
the scalar types. `VALUE_MAX_TYPE` remains the final sentinel.

```cpp
// BEFORE
enum VALUE_DATA_TYPE {
  VALUE_NO_TYPE,
  VALUE_INT_TYPE,
  VALUE_FLOAT_TYPE,
  VALUE_STRING_TYPE,
  VALUE_BOOL_TYPE,
  VALUE_ERROR_TYPE,
  VALUE_EMPTY_TYPE,
  VALUE_LIST_TYPE,
  VALUE_VOID_TYPE,
  VALUE_VECTOR_TYPE,   // unused — replaced
  VALUE_MAX_TYPE
};

// AFTER
enum VALUE_DATA_TYPE {
  VALUE_NO_TYPE,
  VALUE_INT_TYPE,
  VALUE_FLOAT_TYPE,
  VALUE_STRING_TYPE,
  VALUE_BOOL_TYPE,
  VALUE_ERROR_TYPE,
  VALUE_EMPTY_TYPE,
  VALUE_LIST_TYPE,
  VALUE_VOID_TYPE,
  VALUE_VECTOR2_TYPE,  // replaces VALUE_VECTOR_TYPE
  VALUE_VECTOR3_TYPE,
  VALUE_VECTOR4_TYPE,
  VALUE_MAX_TYPE
};
```

Using three explicit subtypes keeps `CalcFinal` dispatch symmetric with
scalar types and avoids a secondary branch inside `CalcFinal` (e.g. a
`GetVectorSize()` call). It also gives Logic Brick runtime binding the
ability to dispatch on specific vector dimensions directly.

---

## 6. Public Interface

All three classes expose the same virtual surface to `EXP_Value` callers,
plus type-specific inline accessors.

### 6.1 Virtual overrides (common to all three)

```cpp
std::string  GetText()       override;  // "(x, y)" / "(x,y,z)" / "(x,y,z,w)"
double       GetNumber()     override;  // MT length() cast to double
int          GetValueType()  override;  // VALUE_VECTOR{2,3,4}_TYPE
void         SetValue(EXP_Value *newval) override;
EXP_Value   *GetReplica()    override;
void         ProcessReplica() override;
EXP_Value   *Calc(VALUE_OPERATOR op, EXP_Value *val) override;
EXP_Value   *CalcFinal(VALUE_DATA_TYPE dtype,
                        VALUE_OPERATOR op,
                        EXP_Value *val) override;
```

### 6.2 Type-specific inline accessors

```cpp
// EXP_Vector2Value
const MT_Vector2 &GetVector2() const;
void              SetVector2(const MT_Vector2 &v);
MT_Scalar         GetLengthSq() const;  // delegates to MT_Vector2::length2()

// EXP_Vector3Value
const MT_Vector3 &GetVector3() const;
void              SetVector3(const MT_Vector3 &v);
MT_Scalar         GetLengthSq() const;  // delegates to MT_Vector3::length2()

// EXP_Vector4Value
const MT_Vector4 &GetVector4() const;
void              SetVector4(const MT_Vector4 &v);
MT_Scalar         GetLengthSq() const;  // delegates to MT_Vector4::length2()
```

`GetLengthSq()` delegates to `MT_VectorN::length2()` which is defined in
the Moto inline files as `dot(*this, *this)` — no `sqrt` involved. This
is the preferred path for Logic Brick sensors that compare magnitudes
every frame.

`GetNumber()` calls `MT_VectorN::length()` (which does call `sqrtf`) and
returns the result as `double`. It exists for backward compatibility with
the expression evaluation pipeline and must not be removed.

### 6.3 `SetValue()` contract

`SetValue(EXP_Value *newval)` handles two cases, matching the pattern of
`EXP_FloatValue`:

- **Same vector type** → direct MT assignment, no allocation.
- **Scalar** (`VALUE_FLOAT_TYPE` / `VALUE_INT_TYPE`) → broadcast the
  scalar to all components via `MT_VectorN::setValue()`.
- **Any other type** → silent no-op (mirrors `EXP_FloatValue` behaviour).

This contract is relied upon by the Property Actuator, which calls
`SetValue()` generically without knowing the concrete type.

---

## 7. Operator Dispatch

The dispatch path is identical to `EXP_FloatValue`:

```
Calc(op, rhs)
  │
  ├─ unary POS, NEG, NOT  → handled locally
  ├─ AND / OR             → EXP_ErrorValue (not defined for vectors)
  └─ binary               → rhs->CalcFinal(VALUE_VECTOR{N}_TYPE, op, this)
                                  │
                                  └─ resolved in rhs by switching on dtype
```

### 7.1 Unary operators

| Operator   | Implementation                  | Notes                              |
|------------|---------------------------------|------------------------------------|
| `+v`       | `new EXP_VectorNValue(m_vec)`   | Copy via MT copy constructor       |
| `-v`       | `new EXP_VectorNValue(-m_vec)`  | MT unary `operator-`               |
| `!v`       | `m_vec.fuzzyZero()`             | Uses `MT_fuzzyZero2(length2())`; no sqrt |
| `&&` `\|\|` | `EXP_ErrorValue`               | Not defined for vectors            |

### 7.2 Binary vector–vector (same dimension)

| Operator | Implementation                          | Notes                              |
|----------|-----------------------------------------|------------------------------------|
| `v + w`  | `lhs + m_vec`                           | MT `operator+`                     |
| `v - w`  | `lhs - m_vec`                           | MT `operator-`                     |
| `v * w`  | `EXP_ErrorValue`                        | Ambiguous (dot? component-wise?)   |
| `v / w`  | `EXP_ErrorValue`                        | Ambiguous                          |
| `v == w` | `MT_fuzzyEqual(lhs, m_vec)`             | `MT_fuzzyZero(lhs-rhs)`; no sqrt   |
| `v != w` | `!MT_fuzzyEqual(lhs, m_vec)`            |                                    |
| `v > w`  | `lhs.GetLengthSq() > rhs.GetLengthSq()` | Squared comparison; no sqrt       |
| `v < w`  | as above                                |                                    |
| `v >= w` | as above                                |                                    |
| `v <= w` | as above                                |                                    |

### 7.3 Binary scalar–vector

| Operator | Implementation                            | Notes                          |
|----------|-------------------------------------------|--------------------------------|
| `s + v`  | `(s+vx, s+vy, ...)`                       | Component-wise broadcast       |
| `s - v`  | `(s-vx, s-vy, ...)`                       | Component-wise broadcast       |
| `s * v`  | `m_vec * s`                               | MT `operator*(scalar)`         |
| `s / v`  | `(s/vx, s/vy, ...)` with `fuzzyZero` guard|                                |
| `s == v` | `MT_fuzzyEqual(GetLengthSq(), s*s)`       | Squared; no sqrt               |
| `s != v` | `!MT_fuzzyEqual(GetLengthSq(), s*s)`      |                                |
| `s > v`  | `s > GetNumber()`                         | Calls `length()`, sqrt needed  |
| `s < v`  | as above                                  |                                |
| `s >= v` | as above                                  |                                |
| `s <= v` | as above                                  |                                |

### 7.4 Cross-dimension operations

Any operation mixing different vector dimensions returns `EXP_ErrorValue`
with a descriptive message. No silent promotion or truncation occurs.

---

## 8. `GetText()` Format

```
EXP_Vector2Value  →  "(x, y)"
EXP_Vector3Value  →  "(x, y, z)"
EXP_Vector4Value  →  "(x, y, z, w)"
```

Formatted with `snprintf` into a stack-allocated `char[]`. `std::ostringstream`
is intentionally not used — it carries locale handling and heap allocation
overhead that is unnecessary for a debug string.

---

## 9. Python Integration

### 9.1 `ConvertValueToPython`

| Class              | Return type             | Condition                       |
|--------------------|-------------------------|---------------------------------|
| `EXP_Vector2Value` | `PyTuple` (2 floats)    | Always — no mathutils.Vector2   |
| `EXP_Vector3Value` | `mathutils.Vector` (3D) | `WITH_PYTHON && USE_MATHUTILS`  |
| `EXP_Vector3Value` | `PyTuple` (3 floats)    | `WITH_PYTHON && !USE_MATHUTILS` |
| `EXP_Vector4Value` | `PyTuple` (4 floats)    | Always — no mathutils.Vector4   |

For `EXP_Vector3Value`, `MT_Vector3::getValue()` returns `const MT_Scalar*`
which is accepted directly by `Vector_CreatePyObject(const float*, int, ...)`.
No intermediate array or copy is needed.

### 9.2 `ConvertPythonToValue`

All three classes override `ConvertPythonToValue`. The base `EXP_Value`
version does not handle vector types and this override is required for
Logic Brick runtime binding to write Python values back into vector
properties.

| Class              | Accepted Python input                                     |
|--------------------|-----------------------------------------------------------|
| `EXP_Vector2Value` | Any sequence of length 2 with numeric items               |
| `EXP_Vector3Value` | `mathutils.Vector` size 3 (fast path via `VectorObject_Check`), or any sequence of 3 floats |
| `EXP_Vector4Value` | Any sequence of length 4 with numeric items               |

The `mathutils.Vector` fast path for Vector3 calls `BaseMath_ReadCallback`
to ensure the internal buffer is valid before reading.

---

## 10. `GetReplica()` / `ProcessReplica()`

All three classes implement `GetReplica()` as copy-construct + `ProcessReplica()`.
`ProcessReplica()` calls `EXP_PropValue::ProcessReplica()` only — MT vector
types own no heap pointers beyond what the base class already manages.

---

## 11. Logic Brick Runtime Binding — Integration Points

No modifications to Property Actuator, Property Sensor, or any other Logic
Brick class are required at this stage. The new types are fully accessible
through the existing `EXP_Value` virtual interface.

| Operation                    | Entry point                                            |
|------------------------------|--------------------------------------------------------|
| Read vector property         | `EXP_Value::GetProperty(name)` → cast to vector type  |
| Write vector property        | `EXP_Value::SetProperty(name, vecValue)`              |
| Sensor magnitude comparison  | `EXP_VectorNValue::GetLengthSq()` directly (no sqrt)  |
| Full expression comparison   | `EXP_Value::Calc(op, rhs)` → `CalcFinal` dispatch     |
| Runtime type check           | `GetValueType() == VALUE_VECTOR{2,3,4}_TYPE`           |
| Assign from Python           | `EXP_VectorNValue::ConvertPythonToValue`               |
| Expose to Python             | `EXP_VectorNValue::ConvertValueToPython`               |
| Scalar coercion              | `GetNumber()` → `MT_VectorN::length()` as double       |
| Dimension conversion         | `MT_Vector4::to3d()`, `MT_Vector4::to2d()`, `MT_Vector3::to2d()` (Moto built-in) |

The dimension conversion methods (`to2d`, `to3d`) are provided by Moto
at no cost and are available for future use in runtime binding without
requiring any changes to this layer.

---

## 12. Out of Scope

- Property Actuator / Sensor UI or RNA serialization.
- Expression parser grammar for vector literals (e.g. `vec3(1, 0, 0)`).
- Dot product / cross product — ambiguous at this abstraction level;
  available in Moto directly when needed by higher layers.
- Any modification to files outside `source/gameengine/`.
