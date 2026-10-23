---
name: reforger-wiki-values
description: "Trigger: int, float, bool, string, vector, array, map, set, typename, enum, const, ref, autoptr, casting, scope. Enforce Script type system, value declarations, passing semantics, and casting."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "int"
    - "float"
    - "bool"
    - "string"
    - "vector"
    - "array"
    - "map"
    - "set"
    - "typename"
    - "enum"
    - "const"
    - "ref"
    - "autoptr"
    - "casting"
    - "scope"
---

## Activation Contract

Load this skill when the context involves declaring or using Enforce Script value types (`int`, `float`, `bool`, `string`, `vector`, `array`, `map`, `set`, `typename`, `enum`), `const`, `ref`, type casting, or variable scope rules. Do NOT load for networking or component lifecycle work — those are separate spoke skills.

## Hard Rules

**Identifiers**
- Identifiers: ASCII letters, digits, `_`. Must NOT start with a digit. Case-sensitive.
- Cannot be identical to a language keyword (e.g. `auto`, `class`, `const`).
- Regex: `^[a-zA-Z_][a-zA-Z0-9_]*$`

**Naming Prefixes (member variables)**
- `m_b` — `bool`, `m_i` — `int`, `m_f` — `float`, `m_s` — `string`/`ResourceName`/`LocalizedString`
- `m_e` — enum, `m_v` — vector, `m_a` — `array`/`Curve`, `m_m` — `map`
- No prefix: class instances, `typename`, `set`, `Color`
- Constants: `UPPER_SNAKE_CASE`, no `m_` or type prefix — e.g. `MAX_VALUE = 9999`

**Strong Types**
- Enforce Script uses strong typing — a variable's type CANNOT change after declaration.
- Declare with type: `int myNumber;` (auto-initialises to default) or `int myNumber = 10;`
- `const` prevents re-assignment. An object can be `const ref` yet still mutate its contents — only the reference is frozen.

**Passing Semantics**
- Primitives (`int`, `float`, `bool`, `string`, `vector`, `enum`) are passed by **value** — copying modifies a separate variable.
- Objects (`array`, `map`, `set`, class instances) are passed by **reference** — assignment shares the same object.
- `string` is passed by value but is effectively immutable at the reference level.

**Types Quick Reference**

| Type | Prefix | Default | Pass by | Notes |
|---|---|---|---|---|
| `bool` | `m_b` | `false` | value | |
| `int` | `m_i` | `0` | value | 32-bit signed |
| `float` | `m_f` | `0.0` | value | Use `float.AlmostEqual()` for comparison |
| `string` | `m_s` | `""` | value | Cannot be `null` |
| `vector` | `m_v` | `{0,0,0}` | value | 3× float |
| `array<T>` | `m_a` | `null` | reference | Dynamic or static |
| `map<K,V>` | `m_m` | `null` | reference | Float cannot be key |
| `set<T>` | none | `null` | reference | Unique values |
| `typename` | none | `typename.Empty` | reference | Cannot be `null` assigned |
| `enum` | `m_e` | `0` | value | Backed by int |
| class | none | `null` | reference | |

**Enum**
- First value defaults to `0`; subsequent values auto-increment. Can assign explicit values.
- Bit-flag enums: `VALUE = 1 << 0`, `VALUE2 = 1 << 1`, etc.
- `enum` value can be cast to `int`.

**Array**
- Dynamic `array<T>` — grows on `Insert()`. Default is `null`, not empty.
- Static `T myArr[N]` — fixed size; index access `[]` is constant-time (no `.Get()` overhead).
- Prefer `foreach` over indexed `for` for arrays; cache the result of `myArray[i]` when used multiple times.
- `array ==` compares references, not values. Two different arrays with the same content are NOT equal.

**Casting**
- `DerivedClass.Cast(baseRef)` — casts a base reference to a derived type. Returns `null` if the underlying type is incompatible — NEVER throws.
- Always null-check the cast result before use.

**Scope**
- Variables exist within the block `{}` where they were declared — they are destroyed when the block exits.
- Declaring the same name in a nested scope creates a new variable; the outer variable is inaccessible in that scope.

## Key APIs / Patterns

```c
// Enum with bit flags
enum ARGA_EStatusFlags
{
    HEALTHY    = 1 << 0, // 1
    HAS_AMMO   = 1 << 1, // 2
    IN_VEHICLE = 1 << 2, // 4
}

// const ref array (reference locked, contents mutable)
const ref array<string> NAMES = {};
NAMES.Insert("Alpha"); // OK
// NAMES = null; // error

// Casting (always null-check)
IEntity entity = GetGame().FindEntity("soldier1");
ARGA_SoldierComponent soldier = ARGA_SoldierComponent.Cast(entity);
if (!soldier)
    return;
soldier.DoSomething();

// float comparison
float a = 0.1 + 0.1 + 0.1;
if (float.AlmostEqual(a, 0.3)) // correct
    Print("close enough");

// Static array — fast indexed access
int primes[5] = { 2, 3, 5, 7, 11 };
int third = primes[2]; // 5, no .Get() overhead

// Dynamic array — cache index result
foreach (int i, string name : m_aNames)
{
    if (name.IsEmpty())
        continue;
    PrintFormat("%1: %2", i, name);
}
```

## References

- Doxygen: `float.AlmostEqual(float a, float b, float epsilon = 0.0001)` confirmed real (`float_8c_source.html`) — matches this skill exactly. No bugs found.
- PDF: `Scripting_ Values – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting:_Values`
