---
name: reforger-wiki-script-invoker
description: "Trigger: ScriptInvoker, ScriptInvokerVoid, ScriptInvokerBase, event handler, event subscription. Event/callback system using ScriptInvoker in Enforce Script."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "ScriptInvoker"
    - "ScriptInvokerVoid"
    - "ScriptInvokerBase"
    - "event handler"
    - "event subscription"
    - "GetOnBark"
---

## Activation Contract

Load this skill when the context involves event systems, callbacks, or observer patterns using `ScriptInvoker`, `ScriptInvokerVoid`, or `ScriptInvokerBase<func>` in Enforce Script. Do NOT load for generic `.c` edits without event/invoker keywords.

## Hard Rules

**Core Rule — Never use ScriptInvoker directly**
- Using the raw `ScriptInvoker` class is considered BAD PRACTICE — always use the typed variants (`ScriptInvokerVoid`, `ScriptInvokerBase<func>`) or a `typedef`-ed type alias.

**Ownership and Lazy Initialization**
- The event owner declares the invoker as a `protected ref` member and initializes it lazily inside a getter method.
- The getter returns the invoker, creating it if null: `if (!m_OnBark) m_OnBark = new ScriptInvokerVoid();`
- Subscribers NEVER access the invoker field directly — always through the owner's getter.

**Invoker Types**
- The ONLY invoker type defined in the engine core (`scripts/GameLib/tools.c`) is `ScriptInvokerBase<Class T> : Managed`, plus the bare typedef `ScriptInvoker = ScriptInvokerBase<func>` (no argument typing at all).
- `ScriptInvokerVoid` and every other typed variant (`ScriptInvokerInt`, `ScriptInvokerBool`, `ScriptInvokerFloat`, `ScriptInvokerString` — each with 1-5 argument versions, `ScriptInvokerEntity`/`Entity2`, `ScriptInvokerVector`, `ScriptInvokerWidget`, `ScriptInvokerBaseWorld`, `ScriptInvokerRplId`, `ScriptInvokerEntityAndStorage`, `ScriptInvokerFaction`) are project-level typedefs defined in `scripts/Game/Helpers/SCR_ScriptInvokerHelper.c`, NOT in the engine core. That file's own header comment says its purpose: "Generic Script invokers. To be used in any script that does not need specific invokers. If a series of types comes up often, it is good to add it here."
- Before writing a custom `typedef` for a new argument signature, check `SCR_ScriptInvokerHelper.c` first — a matching generic invoker very likely already exists there (e.g. `ScriptInvokerInt`, `ScriptInvokerBool2`, `ScriptInvokerFloat3`, `ScriptInvokerEntity2`). Only define your own typedef when the signature is not one of the generic ones already provided.
- Custom typed invokers using `typedef` remain the preferred pattern for non-void events not already covered by the helper file.

**Subscribe / Unsubscribe**
- Subscribe: `owner.GetOnEvent().Insert(OnEvent);` — registers a method reference.
- Unsubscribe: `owner.GetOnEvent().Remove(OnEvent);` — MUST be called in the subscriber's destructor to prevent dangling callbacks.
- Failing to unsubscribe causes the invoker to call a method on a destroyed object → VM crash.

**Firing an Event**
- Check if the invoker exists before calling `Invoke()`: `if (m_OnBark) m_OnBark.Invoke();`
- The lazy-init getter pattern means an owner with zero subscribers never allocates the invoker object.

**Signature Matching**
- Subscriber methods MUST match the invoker's declared signature exactly (same parameter types, same count).
- Mismatched signatures are a runtime error — the Script Editor may not catch them at compile time.

**Back-References from Subscriber**
- Subscriber holds a back-reference to the event source as a WEAK reference (no `ref`) to avoid cyclic ARC leaks.

## Key APIs / Patterns

```c
// --- Event Owner (void event, no args) ---
class TAG_Dog
{
    protected ref ScriptInvokerVoid m_OnBark;  // lazy-initialized

    ScriptInvokerVoid GetOnBark()
    {
        if (!m_OnBark)
            m_OnBark = new ScriptInvokerVoid();
        return m_OnBark;
    }

    void Bark()
    {
        if (m_OnBark)          // only invoke if invoker exists (subscribers registered)
            m_OnBark.Invoke();
        Print("Dog barked", LogLevel.DEBUG);
    }
}

// --- Subscriber ---
class TAG_Neighbour
{
    protected TAG_Dog m_Dog;  // WEAK reference — no ref keyword

    void TAG_Neighbour(notnull TAG_Dog dog)
    {
        m_Dog = dog;
        m_Dog.GetOnBark().Insert(OnDogBark);  // subscribe
    }

    void ~TAG_Neighbour()
    {
        if (m_Dog)
            m_Dog.GetOnBark().Remove(OnDogBark);  // MUST unsubscribe in destructor
    }

    protected void OnDogBark()
    {
        Print("The dog barked!", LogLevel.NORMAL);
    }
}

// --- Event with typed argument (int) ---
void ScriptInvokerIntMethod(int i);
typedef func ScriptInvokerIntMethod;
typedef ScriptInvokerBase<ScriptInvokerIntMethod> ScriptInvokerInt;

class TAG_EventHolder
{
    protected ref ScriptInvokerInt m_OnScoreChanged;

    ScriptInvokerInt GetOnScoreChanged()
    {
        if (!m_OnScoreChanged)
            m_OnScoreChanged = new ScriptInvokerInt();
        return m_OnScoreChanged;
    }

    void AddScore(int points)
    {
        m_iScore += points;
        if (m_OnScoreChanged)
            m_OnScoreChanged.Invoke(m_iScore);  // pass arg to all subscribers
    }

    protected int m_iScore;
}

// Subscriber method MUST match signature: void WarnOfScoreChange(int newScore)
```

## References

- PDF: `ScriptInvoker Usage – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (engine core — source of truth, confirmed no `ScriptInvokerVoid` here): `tools_8c_source.html` / `scripts/GameLib/tools.c` — `ScriptInvokerBase<Class T>` at line 117, `typedef ScriptInvokerBase<func> ScriptInvoker` at line 134. Corrected: the previous "ScriptInvokerVoid at line 131" reference was wrong — that line is `Dump()`, and `ScriptInvokerVoid` is not declared in this file at all.
- Source (not in Doxygen — Game-layer, project convention, not engine core): `scripts/Game/Helpers/SCR_ScriptInvokerHelper.c` — defines `ScriptInvokerVoid` and every other typed generic invoker.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:ScriptInvoker_Usage`
