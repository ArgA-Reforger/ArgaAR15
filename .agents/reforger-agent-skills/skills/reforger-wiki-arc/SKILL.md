---
name: reforger-wiki-arc
description: "Trigger: ARC, Managed, weak reference, memory leak, cyclic reference, reference counting. Automatic Reference Counting memory model for Enforce Script."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "ARC"
    - "Managed"
    - "weak reference"
    - "memory leak"
    - "cyclic reference"
    - "reference counting"
---

## Activation Contract

Load this skill when the context involves memory ownership (`ref`/weak references), `Managed` base class, ARC patterns, cyclic reference bugs, or null-safety from weak references in Enforce Script. Do NOT load for generic `.c` edits without memory/reference keywords.

## Hard Rules

**ARC Fundamentals**
- Enforce Script uses Automatic Reference Counting (ARC), NOT garbage collection. There is no GC pause/stutter.
- ARC only applies to objects (reference types) that inherit from `Managed`. Value types (`int`, `float`, `vector`, etc.) are NOT reference-counted.
- When an object's strong reference count reaches 0, it is immediately released from memory.
- `ref` is ONLY meaningful on class member variables and collection elements — using `ref` on a local variable inside a method is redundant (local variables are already strong references by scope lifetime).

**Strong vs. Weak References**
- `ref ClassName m_Member;` → strong reference: increments ARC counter; object cannot be null during the holder's lifetime unless explicitly `delete`d.
- `ClassName m_Member;` (no `ref`) → weak reference: does NOT increment counter; object MAY become null at any time if all strong references are released. This applies **only** to member variables of class types that inherit from `Managed`. Does NOT apply to primitives (`int`, `float`, `bool`, etc.) or non-Managed value types.
- Weak references REQUIRE null-checking before use — never dereference without a guard.

**Collections and ref**
- `ref array<ClassName>` → the array itself is strong (kept alive), but elements are weak references.
- `ref array<ref ClassName>` → array is strong AND each element is a strong reference.
- `ref map<string, ref ClassName>` → strong map with strong values. Same pattern applies to `set`.
- Static arrays: `ref ClassName m_aItems[10];` → static array of strong references.

**Cyclic Reference (Island of Isolation)**
- A cyclic reference occurs when A holds a strong ref to B and B holds a strong ref to A — both prevent each other's counter from reaching 0, causing a memory leak.
- Solution: one side holds a strong ref (`ref`), the other holds a weak ref (no `ref`).
- Convention: the PARENT holds a strong ref to its CHILD; the CHILD holds a weak ref back to the PARENT.
- After a method ends, local variable references are dropped — if an object's only remaining references are cyclic, it leaks.

**autoptr**
- `autoptr` existed as an older mechanism; in current Enforce Script it is REDUNDANT. Classes inheriting `Managed` are handled automatically. Do not use `autoptr` in new code.

**delete keyword**
- `delete obj;` destroys an object immediately and sets all pointers to it to `null`.
- `delete` on an object still held in a container (e.g., an array) throws a VM Exception — remove from containers FIRST.

## Key APIs / Patterns

```c
class TAG_Parent
{
    // Parent owns the child (strong reference)
    protected ref TAG_Child m_Child;

    void TAG_Parent()
    {
        m_Child = new TAG_Child(this);
    }
}

class TAG_Child
{
    // Child back-references parent as WEAK to avoid cyclic leak
    protected TAG_Parent m_Parent;

    void TAG_Child(TAG_Parent parent)
    {
        m_Parent = parent;
    }

    void DoSomething()
    {
        // Always null-check weak references before use
        if (!m_Parent)
            return;
        // safe to use m_Parent here
    }
}

// Collection ownership patterns
class TAG_Manager
{
    // Array alive; elements are strong refs
    protected ref array<ref TAG_Child> m_aChildren = {};

    void AddChild(TAG_Child child)
    {
        m_aChildren.Insert(child);
    }

    void RemoveChild(TAG_Child child)
    {
        // Remove before delete to avoid VM Exception
        m_aChildren.RemoveItem(child);
        delete child;   // now safe — reference count can reach 0
    }
}
```

## References

- PDF: `Scripting_ Automatic Reference Counting – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting:_Automatic_Reference_Counting`
