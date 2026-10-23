---
name: reforger-wiki-best-practices
description: "Trigger: best practices, code quality, avoid null check omission, single responsibility. Enforce Script coding best practices for safety, readability, and mod compatibility."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "best practices"
    - "code quality"
    - "avoid null check omission"
    - "single responsibility"
---

## Activation Contract

Load this skill when reviewing Enforce Script code for quality, safety, and mod-compatibility. Applies to all `.c` files. Supplements `reforger-wiki-conventions` (style rules) with design-level guidance. Use together with `reforger-wiki-dos-donts` for a complete review checklist.

## Hard Rules

**Null safety**
- ALWAYS null-check the result of `FindComponent`, `Cast`, `GetGame`, `GetGameMode`, and any pointer returned from engine API before using it.
- Use `notnull` parameter annotation only when the caller contract guarantees non-null — document why.
- Crash-on-null is the correct failure mode for debug builds; a silent null skip can cause invisible state corruption.

**Single responsibility**
- Each class has one job. A component that handles input, applies damage, AND plays UI sounds violates SRP.
- Split responsibilities by creating separate components or helper classes.
- "One job" means: one clearly named class, one `EOnInit`/`EOnPostInit`, one primary responsibility.

**Encapsulation**
- Member variables are `protected` by default. Only expose via getter methods.
- Never make a member `public` for "convenience" — add a getter.
- Setters that change replicated state MUST also call `Replication.BumpMe()` only from the **authority**. Guard with: `if (GetRplComponent() && GetRplComponent().IsMaster())` — encapsulate this inside the setter.

**Inheritance vs composition**
- Prefer composition (FindComponent) over deep inheritance chains.
- Subclass only when IS-A is truly correct and the parent is designed for extension.
- Avoid subclassing concrete game classes beyond one level — it makes mod order fragile.

**Magic numbers and strings**
- Never hard-code numbers or strings inline. Move to: `[Attribute]`, `enum`, `const`, or `static const`.
- Exception: trivial constants (`0`, `1`, `true`, `false`) in immediately-obvious context.

**Lifecycle awareness**
- Code in `EOnInit` must not depend on other components existing yet — use `EOnPostInit`.
- Code in `EOnPostInit` can safely access other components on the same entity.
- `EOnDelete` MUST unsubscribe all ScriptInvoker listeners and cancel any pending timers.
- Never access `GetOwner()` after `EOnDelete` has been called — the entity is being torn down.

**Error handling**
- Return early (guard clause) on unexpected state instead of nesting.
- Log meaningful messages with `Print(string, LogLevel.WARNING)` for recoverable unexpected states.
- For unrecoverable states (programmer error), call `Error("message")` to abort with a clear message in debug.

**Mod-friendliness**
- Always call `super` in overridden methods unless you have a documented reason not to.
- Expose extension points (ScriptInvoker events, virtual methods) so modders can hook in without modding your class.
- Avoid `sealed` unless you have a concrete security or correctness reason — `sealed` is a permanent door-close to modders.

**Performance awareness**
- Do NOT allocate objects inside `EOnFrame` — allocate once, reuse.
- Do NOT call `FindComponent` per frame — cache in `EOnPostInit`.
- Do NOT call `string` concatenation in tight loops — build into a buffer or use `PrintFormat`.

## Key APIs / Patterns

```enforce
// Guard clause pattern — null safety first
void ProcessEntity(IEntity entity)
{
    if (!entity)
    {
        Print("ProcessEntity: entity is null", LogLevel.WARNING);
        return;
    }

    SCR_DamageManagerComponent dmgMgr =
        SCR_DamageManagerComponent.Cast(entity.FindComponent(SCR_DamageManagerComponent));
    if (!dmgMgr)
        return;  // entity doesn't have damage manager — not an error, just not applicable

    // Safe to use dmgMgr here — HandleDamage takes a single BaseDamageContext, not positional args
    // (see reforger-wiki-damage-system for the verified signature)
    BaseDamageContext ctx = new BaseDamageContext();
    ctx.damageType = EDamageType.TRUE;
    ctx.damageValue = 10.0;
    dmgMgr.HandleDamage(ctx);
}

// Encapsulated setter with replication bump (authority guard)
class ARGA_MyComponent : ScriptComponent
{
    [RplProp(onRplName: "OnActiveChanged")]
    protected bool m_bActive;

    void SetActive(bool active)
    {
        if (m_bActive == active) return;  // no-op — avoid unnecessary BumpMe
        m_bActive = active;
        if (GetRplComponent() && GetRplComponent().IsMaster())
            Replication.BumpMe();
    }

    bool IsActive() { return m_bActive; }

    protected void OnActiveChanged()
    {
        // proxy callback — update local state/UI
    }
}

// EOnPostInit safe component caching
class ARGA_MyComponent : ScriptComponent
{
    protected SCR_DamageManagerComponent m_pDmgMgr;

    override void EOnPostInit(IEntity owner)
    {
        super.EOnPostInit(owner);
        m_pDmgMgr = SCR_DamageManagerComponent.Cast(owner.FindComponent(SCR_DamageManagerComponent));
        // m_pDmgMgr may be null — check before use in other methods
    }
}
```

## References

- PDF: `Scripting_ Best Practices – Arma Reforger - Bohemia Interactive Community.pdf`
- Corrected: `HandleDamage` does not take `(EDamageType, float, HitZone, Instigator)` — verified real signature is `HandleDamage(notnull BaseDamageContext damageContext)` (see `reforger-wiki-damage-system`, sourced from `scripts/Game/generated/Components/DamageManagerComponent.c`, since this class is not covered by Doxygen).
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting_Best_Practices`
- Related spokes: `reforger-wiki-dos-donts`, `reforger-wiki-conventions`, `reforger-wiki-performance`
