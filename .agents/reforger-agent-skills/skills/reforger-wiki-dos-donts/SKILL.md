---
name: reforger-wiki-dos-donts
description: "Trigger: do's and don'ts, forbidden pattern, anti-pattern, scripting pitfall. Explicit do/don't checklist for common Enforce Script mistakes and correct alternatives."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "do's and don'ts"
    - "forbidden pattern"
    - "anti-pattern"
    - "scripting pitfall"
---

## Activation Contract

Load this skill when reviewing Enforce Script code for dangerous patterns, or when you need a quick explicit do/don't reference. Use together with `reforger-wiki-best-practices` for broader quality guidance.

## Hard Rules

### DO: Always null-check engine-returned pointers

```enforce
// CORRECT
SCR_DamageManagerComponent dmg =
    SCR_DamageManagerComponent.Cast(entity.FindComponent(SCR_DamageManagerComponent));
if (!dmg) return;
```

### DON'T: Use cast result directly without null check

```enforce
// WRONG — crashes if component is absent
SCR_DamageManagerComponent.Cast(entity.FindComponent(SCR_DamageManagerComponent)).HandleDamage(...);
```

---

### DO: Call `super` in every overridden method

```enforce
// CORRECT
override void EOnInit(IEntity owner)
{
    super.EOnInit(owner);
    // Custom logic
}
```

### DON'T: Omit `super` in a `modded` or subclassed override

```enforce
// WRONG — breaks the event chain for all other mods and base class
override void EOnInit(IEntity owner)
{
    // Missing super.EOnInit(owner)
}
```

---

### DO: Subscribe to ScriptInvokers in `EOnPostInit`

```enforce
// CORRECT — all components are ready
override void EOnPostInit(IEntity owner)
{
    super.EOnPostInit(owner);
    SCR_BaseGameMode gm = SCR_BaseGameMode.Cast(GetGame().GetGameMode());
    if (gm)
        gm.GetOnPlayerSpawned().Insert(OnPlayerSpawned);
}
```

### DON'T: Subscribe in constructors or `EOnInit`

```enforce
// WRONG — other components may not exist yet in EOnInit
override void EOnInit(IEntity owner)
{
    GetGame().GetGameMode().GetOnPlayerSpawned().Insert(OnPlayerSpawned);  // null crash risk
}
```

---

### DO: Unsubscribe in `EOnDelete`

```enforce
// CORRECT
override void EOnDelete(IEntity owner)
{
    SCR_BaseGameMode gm = SCR_BaseGameMode.Cast(GetGame().GetGameMode());
    if (gm)
        gm.GetOnPlayerSpawned().Remove(OnPlayerSpawned);
    super.EOnDelete(owner);
}
```

### DON'T: Leave dangling ScriptInvoker subscriptions

```enforce
// WRONG — after entity deletion the callback fires on freed memory
// (no EOnDelete cleanup)
```

---

### DO: Cache FindComponent results in member variables

```enforce
// CORRECT — cache once in EOnPostInit
protected SCR_DamageManagerComponent m_pDmgMgr;
override void EOnPostInit(IEntity owner)
{
    super.EOnPostInit(owner);
    m_pDmgMgr = SCR_DamageManagerComponent.Cast(owner.FindComponent(SCR_DamageManagerComponent));
}
```

### DON'T: Call FindComponent in EOnFrame or per-call logic

```enforce
// WRONG — FindComponent is expensive; called every frame
override void EOnFrame(IEntity owner, float timeSlice)
{
    SCR_DamageManagerComponent dmg =
        SCR_DamageManagerComponent.Cast(owner.FindComponent(SCR_DamageManagerComponent));
}
```

---

### DO: Use `EDamageType` correctly when calling HandleDamage

```enforce
// CORRECT — HandleDamage takes a single BaseDamageContext, not positional args
// (verified against scripts/Game/generated/Components/DamageManagerComponent.c —
// not covered by Doxygen, see reforger-wiki-damage-system)
BaseDamageContext ctx = new BaseDamageContext();
ctx.damageType = EDamageType.FIRE;   // use the semantically correct type
ctx.damageValue = fireDamage;
ctx.struckHitZone = hitZone;
ctx.instigator = instigator;
dmgMgr.HandleDamage(ctx);
```

### DON'T: Use `EDamageType.TRUE` as a general-purpose damage type

```enforce
// WRONG — TRUE damage bypasses armour, suppression, bleeding, etc.
BaseDamageContext ctx = new BaseDamageContext();
ctx.damageType = EDamageType.TRUE;
ctx.damageValue = anyDamage;
dmgMgr.HandleDamage(ctx);
// Only correct when you explicitly want to bypass all modifiers
```

---

### DO: Use `[Attribute]` for all configurable values

```enforce
// CORRECT — tuneable in Workbench without recompile
[Attribute("5.0", UIWidgets.Slider, "Damage radius", "0 20 0.5")]
protected float m_fDamageRadius;
```

### DON'T: Hard-code tunable values in method bodies

```enforce
// WRONG — requires code change to adjust, invisible to designers
void ApplyAreaDamage() { float radius = 5.0; ... }
```

---

### DO: Call `Replication.BumpMe()` after modifying `[RplProp]`

```enforce
// CORRECT
m_iScore += points;
Replication.BumpMe();
```

### DON'T: Modify `[RplProp]` from a proxy

```enforce
// WRONG — only the authority can write RplProp values
if (!Replication.IsServer())
    m_iScore += points;  // silently desyncs — proxy change is never replicated
```

---

### DO: Use `notnull` annotation to document non-null contracts

```enforce
// CORRECT — compiler enforces at call site
void ProcessHitZone(notnull HitZone hz) { ... }
```

### DON'T: Assume `notnull` eliminates the need for defensive code in callers

```enforce
// WRONG — notnull only signals intent; null can still be passed via Cast
// Always verify upstream that the value is non-null before passing as notnull
```

---

### DO: Prefer `proto external` / `proto native` calls for engine API

```enforce
// CORRECT — use the documented proto API, don't reimpliment engine logic in script
```

### DON'T: Replicate engine-side behaviour in script when a proto API exists

```enforce
// WRONG — performance cost and potential desync with engine internals
```

## Key APIs / Patterns

See DO/DON'T pairs in Hard Rules above — all patterns are inline with each rule.

## References

- PDF: `Scripting_ Do's and Don'ts – Arma Reforger - Bohemia Interactive Community.pdf`
- Corrected: `HandleDamage(EDamageType, float, HitZone, Instigator)` is not a real signature — see `reforger-wiki-damage-system` for the verified `BaseDamageContext`-based signature.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting_Dos_and_Donts`
- Related spokes: `reforger-wiki-best-practices`, `reforger-wiki-performance`, `reforger-wiki-multiplayer`
