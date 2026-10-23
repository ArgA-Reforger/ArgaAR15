---
name: reforger-wiki-oop-advanced
description: "Trigger: Cast, upcasting, downcasting, class hierarchy. Advanced OOP in Enforce Script — type casting, class hierarchies, mod load order."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "Cast"
    - "upcasting"
    - "downcasting"
    - "class hierarchy"
---

## Activation Contract

Load this skill when the context involves type casting between class hierarchies, template (generic) classes, `modded` class declaration, or mod load ordering in Enforce Script. Do NOT load for basic class definition without casting or generics.

## Hard Rules

**Casting**
- Upcasting (child → parent) is implicit and always safe: `Animal animal = cocker;`
- Downcasting (parent → child) MUST use static `Cast()` on the target type: `Cocker cocker = Cocker.Cast(dog);`
- If the cast fails at runtime, `Cast()` returns `null` — always null-check after downcast.
- Primitive casts are implicit but truncate: `int value = 4.9f;` → `value == 4`. Casting to `bool` converts any non-zero value to `true`.
- You CANNOT cast unrelated types (e.g., a class instance to `string`).

**Template Classes**
- Declare with `class MyClass<Class T>`. By convention the type parameter is named `T`.
- Methods in a template must be generic enough for the type — e.g., `T.Cast()` works for `Managed`-derived types but NOT for primitives like `int`.
- Template type is specified at instantiation: `Item<string> item = new Item<string>("Hello!");`
- Multiple type parameters are allowed: `class Helper<Class T, Class U, Class V>`.
- The Script Editor cannot detect errors in template code unless the template is used somewhere — silent errors can occur.

**Modded Classes**
- `modded class ClassName` automatically inherits from the original class and replaces it engine-wide.
- Use `super.Method()` to call the original implementation from a modded override.
- `private` members of the original class ARE accessible from the `modded` class — this is by design.
- Only classes within the SAME MODULE can be modded (to mod a class in `GameLib`, the modded class must be in `GameLib`).
- All methods/variables added in a modded class MUST carry the mod's unique `TAG_` prefix to avoid conflicts with other mods.

**Mod Load Order**
- Multiple mods can mod the same class. Load order is defined by mod dependency declarations.
- If Mod B depends on Mod A: load order is `Vanilla → Mod A → Mod B`.
- If no dependencies: load order is arbitrary — do NOT rely on a specific order.
- Each modded class in the chain calls `super` to reach the previous layer, so all mods compose correctly.

## Key APIs / Patterns

```c
// --- Downcasting ---
void HandleDog(Dog dog)
{
    Cocker cocker = Cocker.Cast(dog);  // may return null
    if (!cocker)
        return;
    // safe to use cocker here
}

// --- Template class ---
class TAG_Container<Class T>
{
    protected ref array<T> m_aItems = {};

    void Insert(T item)      { m_aItems.Insert(item); }
    T    Get(int index)      { return m_aItems[index]; }
    int  Count()             { return m_aItems.Count(); }
}

// Instantiation
TAG_Container<IEntity> entities = new TAG_Container<IEntity>();
TAG_Container<string>  names    = new TAG_Container<string>();

// --- Modded class ---
modded class SCR_ExampleClass
{
    override string GetMessage()
    {
        return super.GetMessage() + ", Mod A";  // composes with vanilla + other mods
    }

    // New members MUST carry mod TAG to avoid conflicts
    protected bool m_bARGA_ModdedFlag;
}
```

## References

- Doxygen: `Cast()`/null-on-failure semantics and `<Class T>` template syntax confirmed consistent with dozens of real class declarations examined across this session (e.g. `ScriptInvokerBase<Class T>`, `BaseContainer.Cast`). No bugs found.
- PDF: `Object Oriented Programming Advanced Usage – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Object_Oriented_Programming_Advanced_Usage`
