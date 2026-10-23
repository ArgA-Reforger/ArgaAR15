---
name: reforger-wiki-keywords
description: "Trigger: override, out, inout, notnull, owned, auto, new, delete, thread, super, vanilla, proto, native, volatile, event, typedef. Enforce Script reserved keywords — semantics, usage rules, and code flow."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "override"
    - "out"
    - "inout"
    - "notnull"
    - "owned"
    - "auto"
    - "new"
    - "delete"
    - "thread"
    - "super"
    - "vanilla"
    - "proto"
    - "native"
    - "volatile"
    - "event"
    - "typedef"
---

## Activation Contract

Load this skill when the context uses Enforce Script reserved keywords beyond basic type and visibility modifiers. Specifically: `override`, `out`/`inout`, `notnull`, `auto`, `new`/`delete`, `thread`/`Sleep`/`Wait`, `super`, `vanilla`, `proto`, `native`, `volatile`, `event`, `typedef`, or code flow keywords (`if`, `else`, `for`, `foreach`, `while`, `switch`, `continue`, `return`). Do NOT load for pure type or OOP class definition work.

## Hard Rules

**Method Modifiers**
- `override` — must match the base class method signature exactly. Compiler verifies this. Static `override` resolves at compile time, not polymorphically on the reference type.
- `sealed` on a method — prevents override in derived classes. `sealed` on a class — prevents inheritance.
- `event` — hint for Workbench to expose the method as an entity script event.
- `proto` — engine-side method declaration, compiled from C++. Do not implement in script.
- `native` — alternative compiler hint for internal/API methods.
- `volatile` — internal method that may call back into script; hint for the compiler to preserve stack context.

**Parameter Modifiers**
- `out param` — the method may assign a new value to `param` (replaces the caller's variable). The parameter may enter as `null`.
- `inout param` — the method reads the current value AND may replace it. Both entering value and the new value matter.
- `notnull param` — VM throws an exception at call time if `null` is passed. Removes the need for null-check inside the method — but still null-check the object BEFORE calling when the reference might be null.
- `owned` on return — tells the VM the returned array/string must not be released.

**Lifecycle Keywords**
- `new ClassName(args)` — allocates and calls the constructor. Returns a strong reference.
- `delete obj` — destroys the object and sets all references to `null`. CANNOT delete if an external container still holds a reference (throws VM Exception). Remove from arrays/maps before deleting.
- `auto` — type inference. Avoid: it can silently infer `int` when `float` was intended. Use explicit types.
- `thread MethodName()` — spawns a script thread from a method. Script threads are cooperative (not OS threads), so member variable access is thread-safe. In-game: use `GetGame().GetCallQueue().CallLater()` instead of `thread`. **Important**: `thread` spawns a **synchronous cooperative thread** that runs to completion within the same frame tick (or yields). `CallLater()` schedules **asynchronous deferred execution** for a future frame. They are NOT semantically equivalent — choose based on execution model needed.
- `Sleep(ms)` — suspends the current thread for `ms` milliseconds. Only usable inside a `thread`.
- `null` — represents an unset object reference. Primitives (`int`, `float`, `bool`, `string`) cannot be `null`.

**Inheritance Navigation**
- `this` — refers to the current object instance. Supported in static methods (1.1.0+) as the current type.
- `super` — refers to the immediate parent class method/variable. Only accessible in non-static context (or static 1.1.0+).
- `vanilla` — refers to the original unmodded version of the class when inside a `modded class`. Useful to bypass mod chains.

**Code Flow Keywords**

| Keyword | Rule |
|---|---|
| `if (cond)` | No `then`. Single-statement body doesn't need `{}` but braces are always recommended. |
| `else if` | Two separate words — NOT `elseif`. |
| `for (init; cond; step)` | `init` optional (`int i` auto-inits to 0). |
| `foreach (Type item : collection)` | Iterate arrays, maps, sets. Maps yield key and value. |
| `while (cond)` | Can lock the program — ensure condition terminates. |
| `switch (expr)` | Each `case` requires explicit `break`. `default` label supported. |
| `continue` | Skips to next iteration. Valid in `for`, `foreach`, `while`. |
| `return` | Exits the method immediately. Provide a value if method return type is not `void`. |
| `debug` | Triggers a Workbench breakpoint. Use inside `#ifdef WORKBENCH`. |

**typedef**
- `typedef string Text;` — creates a type alias. Useful for readability.

## Key APIs / Patterns

```c
// out parameter — method assigns to caller's variable
void BuildList(out array<string> result)
{
    result = {};
    result.Insert("Alpha");
}
array<string> myList = null;
BuildList(myList); // myList is now { "Alpha" }

// notnull parameter — skip null check inside
void ProcessEntity(notnull IEntity entity)
{
    entity.DoSomething(); // safe to call directly
}
// caller must guard:
if (myEntity)
    ProcessEntity(myEntity);

// delete safely — remove from container first
array<ref ARGA_MyClass> m_aObjects = {};
ARGA_MyClass obj = new ARGA_MyClass();
m_aObjects.Insert(obj);
m_aObjects.Remove(m_aObjects.Find(obj)); // remove reference first
delete obj; // now safe

// thread — Workbench/plugin only
void MainMethod()
{
    thread BackgroundWork();
    Print("continues immediately");
}
protected void BackgroundWork()
{
    Sleep(500);
    Print("500ms later");
}

// super / vanilla in modded class
modded class ARGA_MyComponent
{
    override int GetHealth()
    {
        int base = super.GetHealth();      // previous mod in chain
        int orig = vanilla.GetHealth();    // original unmodded value
        return base + 10;
    }
};

// Optimised for loop
for (int i, count = m_aItems.Count(); i < count; i++)
{
    Print(m_aItems[i]);
}

// foreach with index
foreach (int i, string item : m_aItems)
{
    PrintFormat("%1: %2", i, item);
}

// switch with break
switch (m_eState)
{
    case ARGA_EState.ACTIVE:
        Start();
        break;
    case ARGA_EState.IDLE:
        Pause();
        break;
    default:
        break;
}
```

## References

- PDF: `Scripting_ Keywords – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting:_Keywords`
