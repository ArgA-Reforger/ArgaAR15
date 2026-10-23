---
name: reforger-wiki-damage-system
description: "Trigger: SCR_DamageManagerComponent, DamageType, HitZone, EDamageType, OnDamage. Damage system class hierarchy, logic flow, DamageEffects API, SetHealth caveats, and mod-compatibility patterns."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "SCR_DamageManagerComponent"
    - "DamageType"
    - "HitZone"
    - "EDamageType"
    - "OnDamage"
---

## Activation Contract

Load this skill when writing or reviewing damage handling code: overriding `OnDamage`, working with `HitZone`, applying `EDamageType`, using `SetHealth`, or reasoning about the damage system's class hierarchy and logic flow.

## Hard Rules

**Class hierarchy (engine → script)**
```
HitZoneContainerComponent         // engine-side (extends GameComponent)
DamageManagerComponent            // engine-side
  SCR_DamageManagerComponent      // script-side — PRIMARY extension point
ExtendedDamageManagerComponent    // engine-side
  SCR_ExtendedDamageManagerComponent  // script-side — read its comments and warning before use
    SCR_CharacterDamageManagerComponent  // script-side — character-specific
```
- `SCR_DamageManagerComponent` is the main script-side class. Extend it for custom damage managers.
- `SCR_ExtendedDamageManagerComponent` introduces `DamageEffects` (replaces `damageOverTime`). Read its inline comments carefully before subclassing — there is a documented warning.
- Some API from `DamageManagerComponent` is NOT useful for managers inheriting from `ExtendedDamageManagerComponent`.

**Logic flow (per damage event)**
1. `DamageManagerComponent.OnDamage()` checks: handling enabled, not hijacked, counts as a hit, entity not dead, damage should be dealt.
2. Damage is sent to the `HitZone` → calculates effective damage → replicates hit to clients → deals damage.
3. Hit zone passes damage to parent hit zones.
4. If `DamageState` did not change → exit. If it changed → hit zone fires `OnDamageStateChanged`, then manager fires its own `OnDamageStateChanged`.

**CORRECTED signatures (verified against `scripts/GameCode/Components/SCR_DamageManagerComponent.c`, Reforger_1.6.0.119)**
- `OnDamage` does NOT take 6 separate parameters. The real override signature is a single context object: `override protected void OnDamage(notnull BaseDamageContext damageContext)`. `BaseDamageContext` (`scripts/Game/generated/Damage/BaseDamageContext.c`) exposes the fields: `damageType` (`EDamageType`), `damageValue` (`float`), `hitEntity` (`IEntity`), `colliderID` (`int`), `struckHitZone` (`HitZone`), `damageSource` (`IEntity`), `instigator` (`ref Instigator`), `material` (`GameMaterial`), `hitPosition`/`hitDirection`/`hitNormal`/`impactVelocity` (`vector`), `boneIndex` (`int`), `damageEffect` (`ref BaseDamageEffect`).
- `OnDamageStateChanged` parameter order and count were wrong. The real override signature is: `protected override void OnDamageStateChanged(EDamageState newState, EDamageState previousDamageState, bool isJIP)` — note `newState` comes FIRST (not `previousState`), and there is a third `isJIP` bool parameter that was missing entirely.
- The ScriptInvoker getters for damage events are confirmed to live directly on `SCR_DamageManagerComponent`: `GetOnDamage()`, `GetOnDamageStateChanged()`, `GetOnDamageOverTimeAdded()`, `GetOnDamageOverTimeRemoved()` — NOT on `EventHandlerManagerComponent` (see `reforger-wiki-event-handlers`, which previously claimed otherwise).
- This skill's own References section notes it targets "Reforger version 1.2.0". The signatures above are from the 1.6.0.119 snapshot in this repo — the API evidently changed between those versions (context-object refactor of `OnDamage`, added `isJIP` parameter). Treat 1.2.0-era signatures as outdated for any current project.

**SetHealth — use sparingly**
- `HitZone.SetHealth(value)` changes HP directly without going through the damage logic flow. No instigator, no `OnDamage` path.
- Only `OnHealthSet` and `OnDamageStateChanged` are called when using `SetHealth`.
- Use it only when you need to force a health state change that bypasses normal damage (e.g., force-kill with damage handling disabled).
- `SetHealth` is NOT appropriate as a general-purpose damage applier.

**Preferred way to force-destroy an entity**
- Call `HandleDamage` with a "realistic" but sufficient amount of damage rather than max HP true damage.
- CORRECTED signature (verified against `scripts/Game/generated/Components/DamageManagerComponent.c:74` and `scripts/Game/generated/HitZone/HitZone.c:67`): there is no single `HandleDamage(EDamageType type, float damage, ...)` overload on the damage manager. There are two different real overloads on two different classes:
  - `DamageManagerComponent.HandleDamage(notnull BaseDamageContext damageContext)` — context-object form, same pattern as `OnDamage`.
  - `HitZone.HandleDamage(float damage, int damageType, IEntity instigator)` — note the order is `(damage, damageType, instigator)`, and `damageType` is passed as `int`.
  - Passing `(maxHealth, true)` to either is wrong regardless of exact signature, and remains a mod-compatibility anti-pattern: a modded entity (e.g. high-health Terminator mod) will be killed even if it should survive.
- Always think: "could a valid modded entity survive this damage amount?" If no, reconsider the approach.

**DamageEffects (SCR_ExtendedDamageManagerComponent only)**
- `DamageEffects` replaces the legacy `damageOverTime` API.
- Legacy API (`OnDamageOverTimeAdded`, `OnDamageOverTimeRemoved`, `IsDamagedOverTime`, `GetDamageOverTime`, `RemoveDamageOverTime`) maps to new equivalents (`OnDamageEffectAdded`, etc.).
- `SCR_CharacterDamageManagerComponent.IsBleeding()` wraps the persistent effect check for characters.

**EDamageType**
- The damage type enum (`EDamageType`) classifies the source of damage (ballistic, explosion, fire, etc.).
- Always pass the correct `EDamageType` when calling damage-related API — incorrect types may cause wrong gameplay behaviour (suppression, bleeding, armour penetration, etc.).

**Replication**
- The damage system replicates hits to clients so they can calculate damage locally and display effects. Do not duplicate replication for damage events handled by the system.

## Key APIs / Patterns

```enforce
// Override OnDamage in SCR_DamageManagerComponent subclass — single context object, not 6 params
override protected void OnDamage(notnull BaseDamageContext damageContext)
{
    // Always call super to preserve base behaviour
    super.OnDamage(damageContext);

    // Custom logic after base damage is applied, using fields on the context:
    // damageContext.damageType, damageContext.damageValue, damageContext.struckHitZone,
    // damageContext.instigator, damageContext.hitPosition, etc.
}

// Override OnDamageStateChanged — newState comes FIRST, plus an isJIP flag
protected override void OnDamageStateChanged(EDamageState newState, EDamageState previousDamageState, bool isJIP)
{
    super.OnDamageStateChanged(newState, previousDamageState, isJIP);
    // React to state transition (e.g. play death animation)
    // isJIP is true when this callback fires because a Join-In-Progress client
    // is catching up to the current state, not because damage just happened
}

// Bad: direct SetHealth to kill — bypasses OnDamage, no instigator
// pHitZone.SetHealth(0);  // AVOID unless you specifically need to bypass the flow

// Better: HandleDamage with a realistic amount — context-object form on the manager
// Ensure the amount is "survivable by a modded entity" in principle
BaseDamageContext ctx = new BaseDamageContext();
ctx.damageType = EDamageType.TRUE;
ctx.damageValue = sufficientDamage;
ctx.struckHitZone = hitZone;
ctx.instigator = instigator;
GetDamageManager().HandleDamage(ctx);

// Alternative: HitZone-level overload takes (damage, damageType, instigator) — note the order
hitZone.HandleDamage(sufficientDamage, EDamageType.TRUE, instigatorEntity);

// Get the damage manager from an entity
SCR_DamageManagerComponent dmgMgr =
    SCR_DamageManagerComponent.Cast(entity.FindComponent(SCR_DamageManagerComponent));
if (!dmgMgr) return;

// DamageEffects API (SCR_ExtendedDamageManagerComponent)
// OnDamageEffectAdded replaces OnDamageOverTimeAdded
// OnDamageEffectRemoved replaces OnDamageOverTimeRemoved
```

## References

- PDF: `Damage System – Arma Reforger - Bohemia Interactive Community.pdf` — targets Reforger version 1.2.0; several signatures in that document are now outdated (see corrections above).
- Doxygen coverage gap: this Doxygen build only documents the generic Enfusion engine (`Core`/`GameLib`/`WorkbenchCommon`), not the Arma Reforger game-specific layer (`Game`/`GameCode`) where `SCR_DamageManagerComponent` lives. Verified instead against raw source: `scripts/GameCode/Components/SCR_DamageManagerComponent.c`, `scripts/Game/generated/Damage/BaseDamageContext.c`, `scripts/Game/generated/Components/DamageManagerComponent.c`, `scripts/Game/generated/HitZone/HitZone.c` (all from the Reforger_1.6.0.119 snapshot, which at the time of this audit was the only local source with the full `Game/`/`GameCode/` tree). **Outdated as of 2026-08-18:** `Reforger_1.8.0.10` now ships engine *and* gameplay and is the base source for new lookups; the 1.6.0.119 paths above stand as the record of what was verified then. **Outdated as of 2026-09-03:** `Reforger_1.8.0.13` is now the base source for new lookups (matches the installed game build); it lacks `Core/`, `WorkbenchCommon/`, `WorkbenchGameCommon/` though, so fall back to `Reforger_1.8.0.10` for those.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Damage_System`
