---
name: reforger-wiki-oop-basics
description: "Trigger: class, extends, modded, sealed, inheritance, constructor, getter, setter, instance. OOP class model for Enforce Script — classes, methods, visibility, constructor, inheritance."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "class"
    - "extends"
    - "modded"
    - "sealed"
    - "inheritance"
    - "constructor"
    - "getter"
    - "setter"
    - "instance"
---

## Activation Contract

Load this skill when the context involves defining or inheriting Enforce Script classes, constructors, method overriding, visibility modifiers, or the `modded` keyword. Do NOT load for generic `.c` file edits without OOP class structure.

## Hard Rules

**Classes**
- A class is the blueprint; an object is an instance — `new ClassName()` creates one.
- A class method named identically to the class is the constructor; there can be at most one constructor.
- Two methods may share the same name only if their parameter lists differ (overloading). They CANNOT differ by return type alone.
- Class body closing brace MUST end with `;`.

**Visibility**
- `private`: accessible only within the declaring class. `modded` classes can still access private members of the modded target.
- `protected`: accessible within the class and all derived classes. Use `protected` by default for member variables.
- No modifier (public): accessible from anywhere. Avoid for member variables — use getters/setters instead.
- A public member variable is bad practice; always expose via getter/setter to allow implementation evolution.

**Inheritance**
- `extends` and `:` are equivalent: `class Child extends Parent` == `class Child : Parent`.
- `sealed class` cannot be inherited from; `sealed void Method()` cannot be overridden.
- `modded class` replaces the original class everywhere, inheriting it. All methods and variables inside a modded class must carry the mod's unique `TAG_` to avoid conflicts.
- `override` methods MUST match the base class signature exactly. Static override behavior is resolved at compile time (not polymorphic on the reference type).

**Getters and Setters**
- Member variables MUST be `protected` and exposed via getter and setter methods.
- Getters: `int GetHealth()`, setters: `void SetHealth(int health)`.
- This pattern allows internal implementation to change without breaking callers.

**Static Members**
- Static methods and variables belong to the class, not to instances.
- Access via `ClassName.Member` without an object pointer.
- Static variables are reset game-wide on modded scenario start/leave — do not rely on persistent state in static fields across scenario transitions.

**Object Lifecycle**
- `new` creates an instance. `delete` destroys it and sets all references to `null`.
- `delete` throws a VM Exception if the object is referenced in ANY external container (array, map, set) or held as a member variable in another object. Remove ALL references before calling `delete`.
- ARC (Automatic Reference Counting) manages memory — classes inheriting `Managed` are handled automatically; `autoptr` is redundant in script.

## Key APIs / Patterns

```c
// Class definition with constructor and getter/setter
class ARGA_MyComponent : ScriptComponent
{
    protected int m_iHealth;

    void ARGA_MyComponent()
    {
        m_iHealth = 100;
    }

    int GetHealth()
    {
        return m_iHealth;
    }

    void SetHealth(int health)
    {
        m_iHealth = health;
    }
};

// Inheritance with override
class ARGA_ArmoredComponent : ARGA_MyComponent
{
    override void SetHealth(int health)
    {
        // armored units have minimum 10 hp
        if (health < 10)
            health = 10;
        super.SetHealth(health);
    }
};

// Modded class (extends original transparently)
modded class ARGA_MyComponent
{
    protected bool m_bARGA_IsShielded;

    override int GetHealth()
    {
        if (m_bARGA_IsShielded)
            return super.GetHealth() + 50;
        return super.GetHealth();
    }
};
```

## References

- Doxygen: `class X : Y` / `sealed class` / getter-setter patterns confirmed consistent with dozens of real class declarations examined across this session (e.g. `sealed class BaseContainer : BaseResourceObject`, `sealed class ActionManager`). No bugs found.
- PDF: `Object Oriented Programming Basics – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen: `https://community.bistudio.com/wikidata/external-data/arma-reforger/EnfusionScriptAPIPublic/`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Object_Oriented_Programming_Basics`
