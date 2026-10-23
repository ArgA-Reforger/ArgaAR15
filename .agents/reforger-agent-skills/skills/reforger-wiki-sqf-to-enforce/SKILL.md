---
name: reforger-wiki-sqf-to-enforce
description: "Trigger: SQF, forEach, hint, systemChat, isServer, player, then, migration, Arma 3, Arma Reforger migration. SQF to Enforce Script migration — syntax differences, code flow, type system, OOP shift."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "SQF"
    - "forEach"
    - "hint"
    - "systemChat"
    - "isServer"
    - "migration"
    - "Arma 3"
    - "then"
    - "_forEachIndex"
    - "_x"
---

## Activation Contract

Load this skill when the context involves migrating SQF code to Enforce Script, referencing Arma 3 scripting patterns, or when SQF-style syntax appears in Enforce Script context. Do NOT load for pure Enforce development without SQF context.

## Hard Rules

**Language Philosophy**
- SQF is keyword-based and procedural — commands bridge you to engine objects (e.g., `player`).
- Enforce Script is OOP — everything is an object. Engine interaction is through class instances and APIs.
- Enforce is stricter, closer to the engine (inspired by C++), allows more power but more responsibility.

**Syntax Differences**

| Feature | SQF | Enforce Script |
|---|---|---|
| Variables | `private _myVar = true;` | `bool myVar = true;` |
| Comments | `// inline`, `/* block */` | `// inline`, `/* block */` |
| `if` | `if (cond) then { }` | `if (cond) { }` — no `then` |
| `else if` | `else { if (cond) then { } }` | `else if (cond) { }` |
| `for` | `for "_i" from 0 to 10 do {}` | `for (int i; i < 10; i++) {}` |
| `foreach` | `{ hint str _x } forEach _items` | `foreach (Type item : items) {}` |
| `while` | `while { cond } do {}` | `while (cond) {}` |
| `switch` | `switch (i) do { case 0: {}; default {} }` | `switch (i) { case 0: ...; break; default: ...; break; }` |
| `i++` | does not exist in SQF | `i++` works natively |
| Strong types | no (dynamic) | yes — cannot change type after declaration |
| Case sensitivity | no | YES — `myVar` ≠ `MyVar` |

**Key Migration Rules**
1. Remove `then` from all `if` statements. No `then` keyword exists in Enforce.
2. Replace `_` prefix variables with typed declarations: `private _health = 80;` → `int health = 80;`
3. Replace `forEach` postfix with `foreach` prefix: `{ ... } forEach list` → `foreach (Type item : list) { ... }`
4. `_forEachIndex` → `int currentIndex` as the first param in `foreach (int currentIndex, Type item : list)`.
5. `_x` → the named element variable in the `foreach` signature.
6. `hint` and `systemChat` have no direct equivalent — use `Print()` / `PrintFormat()` for logging, or UI API for HUD.
7. `isServer` → `Replication.IsServer()` (runtime check — no compile-time guards).
8. There is no `player` global in Enforce Script. Use: `IEntity player = GetGame().GetPlayerController().GetControlledEntity();` — UNCONFIRMED: `GetControlledEntity()` was not found anywhere in the local Doxygen dump, and a direct `PlayerController` class-page guess on arexplorer.zeroy.com 404'd. The general shape (get the player controller, ask it for its controlled entity) is very likely the right idea, but verify the exact method/class name before relying on it.
9. `while { condition } do {}` → `while (condition) {}`.
10. `switch (x) do { case 0: {}; }` → `switch (x) { case 0: ...; break; }` — `break` is required in Enforce.
11. SQF is NOT case-sensitive — Enforce IS. All identifiers and keywords must match exact case.
12. A variable declared as `int` cannot later hold a `float` — types are fixed at declaration.

**OOP Shift**
- SQF gives direct access to engine globals (`player`, `allUnits`). Enforce requires API calls through object instances.
- Logic that was a flat sequence of commands in SQF becomes a class method in Enforce.
- Reusable SQF functions become class methods with `protected` or `private` visibility.
- SQF `params` → Enforce method parameters with types.

**Similarities (very few)**
- `//` and `/* */` comment syntax is the same.
- Basic variable assignment and arithmetic operators are syntactically similar.
- Boolean literals `true` / `false` are the same.

## Key APIs / Patterns

```c
// SQF:
// if (alive player) then { hint "I am alive!" };
// Enforce:
if (myPlayer.GetHealth() > 0)
    Print("I am alive!");

// SQF:
// { hint str _x } forEach _items;
// Enforce:
foreach (string item : m_aItems)
{
    Print(item);
}

// SQF (with index):
// { hint str _forEachIndex } forEach _items;
// Enforce:
foreach (int i, string item : m_aItems)
{
    PrintFormat("%1: %2", i, item);
}

// SQF:
// for "_i" from 0 to 10 do { };
// Enforce:
for (int i; i < 10; i++)
{
    Print(i.ToString());
}

// SQF:
// while { _i < 10 } do { _i = _i + 1; };
// Enforce:
while (i < 10)
{
    i++;
}

// SQF:
// switch (i) do { case 0: { systemChat "zero" }; default { systemChat "other" }; };
// Enforce:
switch (i)
{
    case 0:
        Print("zero");
        break;
    default:
        Print("other");
        break;
}

// SQF:
// private _health = 80;
// Enforce:
int health = 80; // typed, cannot change to float later

// isServer equivalent:
if (Replication.IsServer())
{
    // server-only logic
}
```

## References

- PDF: `From SQF to Enforce Script – Arma Reforger - Bohemia Interactive Community.pdf`
- Checked, inconclusive: `PlayerController.GetControlledEntity()` — not found in local Doxygen dump; direct class-page guess on `arexplorer.zeroy.com` 404'd (see reference memory `reforger/arexplorer-online-doxygen`). `float.AlmostEqual`, `Replication.IsServer()`, and the general syntax comparisons in this skill were not contradicted by anything found this session.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:From_SQF_to_Enforce_Script`
