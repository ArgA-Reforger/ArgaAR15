---
name: reforger-wiki-class-template
description: "Trigger: class template, generic class, <Class T>, type parameter, TAG_EntityCastHelper. Template/generic class patterns for Enforce Script."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "class template"
    - "generic class"
    - "<Class T>"
    - "type parameter"
    - "TAG_EntityCastHelper"
    - "template class"
---

## Activation Contract

Load this skill when the context involves defining or using template (generic) classes with `<Class T>` syntax, or writing reusable type-parameterized containers and helpers in Enforce Script. Do NOT load for regular class definitions without type parameters.

## Hard Rules

**Template Declaration**
- Template classes use `class MyClass<Class T>` syntax — the `Class` keyword is required before the parameter name.
- By convention the type parameter is named `T` (single type) or `T, U, V` for multiple parameters.
- Template methods must be generic enough to apply to the declared type — e.g., `T.Cast()` works for `Managed` subtypes but will error on primitives like `int`.

**Known Limitation: Null-check behavior**
- `if (!value)` on a class instance behaves differently from `if (!value)` on an `int` — the null-check semantics differ. Be explicit: use `value == null` for reference types and `value == 0` for integers when using templates that accept both.

**Silent Errors**
- The Script Editor CANNOT detect errors inside a template class unless the template is instantiated somewhere. Silent errors (wrong method calls, incompatible types) will only surface at runtime when the template is used with a specific type.
- Always test templates with at least one concrete instantiation before shipping.

**Multiple Type Parameters**
- `class MyHelper<Class T, Class U, Class V>` is valid for helpers that manage parallel typed collections.
- Each type parameter is independent — methods can accept, return, or store any combination.

**Static Methods in Templates**
- Static template methods work as expected. Access via the parameterized type: `MyHelper<SomeType>.StaticMethod()`.

**Usage Pattern**
- Instantiate with concrete type: `TAG_Container<IEntity> entities = new TAG_Container<IEntity>();`
- The `array<T>` built-in is the canonical example: `array<string>`, `array<bool>`, `array<IEntity>`.

## Key APIs / Patterns

```c
// Single-type template — cast helper
class TAG_EntityCastHelper<Class T>
{
    static T Get(Managed something)
    {
        return T.Cast(something);
    }

    static bool IsValid(Managed something)
    {
        return Get(something) != null;
    }

    static void Output(T anInstance)
    {
        if (anInstance)
            Print(anInstance.ToString());
    }
}

// Usage
TAG_EntityCastHelper<IEntity> helper = new TAG_EntityCastHelper<IEntity>();
IEntity entity = EntityUtils.GetPlayer();
bool valid = helper.IsValid(entity);  // true if entity is an IEntity

// Multi-type template — parallel typed storage
class TAG_TripleStore<Class T, Class U, Class V>
{
    protected ref array<T> m_aT = {};
    protected ref array<U> m_aU = {};
    protected ref array<V> m_aV = {};

    void Insert(T item1, U item2, V item3)
    {
        m_aT.Insert(item1);
        m_aU.Insert(item2);
        m_aV.Insert(item3);
    }

    bool GetByItem1(T item1, out U item2, out V item3)
    {
        int index = m_aT.Find(item1);
        if (index < 0)
            return false;
        item2 = m_aU[index];
        item3 = m_aV[index];
        return true;
    }
}
```

## References

- PDF: `Class Template Example – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen: `class MyClass<Class T>` syntax confirmed real and matches this skill exactly — verified via the real engine class `ScriptInvokerBase<Class T> : Managed` (`tools_8c_source.html`, see `reforger-wiki-script-invoker`). No bugs found.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Class_Template_Example`
