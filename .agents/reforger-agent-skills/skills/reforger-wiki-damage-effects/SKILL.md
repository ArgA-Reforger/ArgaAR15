---
name: reforger-wiki-damage-effects
description: "Trigger: DamageEffect, SCR_DamageEffect, OnEffectAdded, OnEffectApplied, OnEffectRemoved, HandleConsequences, ApplyEffect. Data-driven damage effect objects (bleeding, fire, poison, explosions, etc.) managed by SCR_ExtendedDamageManagerComponent — NOT a standalone visual/audio component."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "2.0.0"
  triggers:
    - "DamageEffect"
    - "SCR_DamageEffect"
    - "OnEffectAdded"
    - "OnEffectApplied"
    - "OnEffectRemoved"
    - "HandleConsequences"
    - "ApplyEffect"
---

## Activation Contract

Load this skill when implementing or reviewing `DamageEffect`/`SCR_DamageEffect` subclasses (bleeding, poison, fire, explosions, collisions, etc.) managed through `SCR_ExtendedDamageManagerComponent`. Do not activate for the core damage-application flow (`OnDamage`, `HandleDamage`, hit zones) — use `reforger-wiki-damage-system` for that.

## Hard Rules

**MAJOR CORRECTION (v2.0.0) — verified against real source, not the previous version of this skill**

The previous version of this skill described a `SCR_DamageEffectComponent` class with subtypes `SCR_ParticleDamageEffect`/`SCR_SoundDamageEffect`/`SCR_LightDamageEffect`, and lifecycle callbacks `OnActivate(IEntity, EDamageState)`/`OnDeactivate(IEntity)`. **None of this exists.** Verified two ways:
- A full filename search across the entire `Game`/`GameCode` source tree (Reforger_1.6.0.119 snapshot) found zero files matching `SCR_DamageEffectComponent`, `SCR_ParticleDamageEffect`, `SCR_SoundDamageEffect`, or `SCR_LightDamageEffect`.
- Cross-checked against `arexplorer.zeroy.com` (indexes `Game`/`GameCode`, current version 1.7.0.54) — same result.

**Real class hierarchy** (verified: `scripts/Game/generated/DamageEffects/BaseDamageEffect.c`, `scripts/Game/Damage/DamageEffects/BaseDamageEffects/SCR_DamageEffect.c`, and the real concrete subclass folders):
```
BaseDamageEffect : ScriptAndConfig          // engine-generated base
  SCR_DamageEffect : BaseDamageEffect       // script-side base — extend this for custom effects
    // BaseDamageEffects/ — organized by TIMING, not by visual/audio type:
    SCR_DotDamageEffect, SCR_InstantDamageEffect, SCR_PersistentDamageEffect
    // CharacterDamageEffects/ — character-specific gameplay effects:
    SCR_BleedingDamageEffect, SCR_FallDamageEffect, SCR_PoisonDamageEffect,
    SCR_BandageDamageEffect, SCR_MorphineDamageEffect, SCR_SalineDamageEffect,
    SCR_TourniquetDamageEffect, SCR_DrowningDamageEffect, SCR_AnimatedFallDamageEffect, ...
    // DamageEffectSources/ — damage-cause-specific effects:
    SCR_ExplosionDamageEffect, SCR_MeleeDamageEffect, SCR_CollisionDamageEffect,
    SCR_FragmentationDamageEffect, SCR_MuzzleBlastDamageEffect, SCR_ConcussionDamageEffect, ...
```
There is no dedicated "particle" or "sound" subclass family — visual/audio feedback is implemented per concrete effect (or on a shared evaluator), not by a fixed 3-way split.

**Real API on `BaseDamageEffect`** (verified in source):
- `ApplyEffect(SCR_ExtendedDamageManagerComponent dmgManager)` — applies the effect. `InstantDamageEffect`s apply as soon as they're added; `PersistentDamageEffect`s only apply when `ApplyEffect()` is explicitly called. This call is automatically replicated (triggers `Save`).
- `GetTotalDamage()`, `GetDamageType()`, `GetInstigator()` (`notnull Instigator`), `GetAffectedHitZone()`.
- `SetDamageType(EDamageType)`, `SetInstigator(notnull Instigator)`, `SetAffectedHitZone(notnull HitZone)` — only valid before the effect is added to a manager; check `IsValueChangeAllowed()` first, changing these after adding desyncs replication.
- `IsProxy()` — whether this is a proxy-side instance.

**Real callbacks to override — NOT `OnActivate`/`OnDeactivate`**:
- `HijackDamageEffect(dmgManager)` returns `bool` — return `true` to intercept/block the effect from being added/applied (any modifications you make still persist).
- `OnEffectAdded(dmgManager)` — fires when added to the manager.
- `OnEffectApplied(dmgManager)` — fires when applied.
- `HandleConsequences(dmgManager, evaluator)` — called from `ApplyEffect`; prefer implementing consequences on the `evaluator` (`DamageEffectEvaluator`) when possible, for reuse across effects.
- `OnEffectRemoved(dmgManager)` — fires on removal.
- `OnDiag(dmgManager)` — writes debug text to the Diag Menu when effect diagnostics are enabled.
- `Save(ScriptBitWriter w)` / `Load(ScriptBitReader r)` — DamageEffects CANNOT use `[RplProp]`/RPCs, so replication of an effect's state goes through these two methods instead. `SCR_DamageEffect` already overrides both to (de)serialize the damage type (skipping the write when it equals `GetDefaultDamageType()`, which defaults to `EDamageType.TRUE`) — call `super.Save(w)`/`super.Load(r)` first in your own overrides.

**Ownership and management**
- Effects are added to and driven by `SCR_ExtendedDamageManagerComponent` (see `reforger-wiki-damage-system`) — there is no separate standalone "effects component" on the entity.
- Effects are data-driven via prefab/config attributes where the concrete project wires them up, but the exact attribute schema was not independently re-derived in this pass — verify against a real prefab or `SCR_ManagedDamageEffectsContainer`/`SCR_BatchedDamageEffects` (`scripts/Game/Systems/DamageEffects/`) before assuming a specific `[Attribute]` layout.

## Key APIs / Patterns

```enforce
// Custom damage effect — override the REAL lifecycle callbacks
class ARGA_MyCustomDamageEffect : SCR_DamageEffect
{
    override void OnEffectAdded(SCR_ExtendedDamageManagerComponent dmgManager)
    {
        // Custom effect startup — e.g. register visual/audio state
    }

    override void OnEffectRemoved(SCR_ExtendedDamageManagerComponent dmgManager)
    {
        // Cleanup
    }

    protected override void HandleConsequences(SCR_ExtendedDamageManagerComponent dmgManager, DamageEffectEvaluator evaluator)
    {
        // Prefer implementing on the evaluator when the logic is reusable across effects
        evaluator.HandleEffectConsequences(this, dmgManager);
    }
}

// Checking current damage state (from the damage manager, not a separate effects component)
SCR_DamageManagerComponent dmgMgr =
    SCR_DamageManagerComponent.Cast(entity.FindComponent(SCR_DamageManagerComponent));
if (!dmgMgr) return;
// See reforger-wiki-damage-system for OnDamageStateChanged and related APIs.
```

## References

- PDF: `Damage Effects – Arma Reforger - Bohemia Interactive Community.pdf` (superseded for class names/API by the source-verified corrections above)
- Doxygen coverage gap: `Game`/`GameCode` (where `DamageEffect` subclasses live) is not covered by the local Doxygen dump. Verified instead against `scripts/Game/generated/DamageEffects/BaseDamageEffect.c` and `scripts/Game/Damage/DamageEffects/BaseDamageEffects/SCR_DamageEffect.c` (Reforger_1.6.0.119 raw source), cross-checked with `arexplorer.zeroy.com` (see reference memory `reforger/arexplorer-online-doxygen`).
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Damage_Effects`
- Related spoke: `reforger-wiki-damage-system` (damage logic, `SCR_ExtendedDamageManagerComponent`, `OnDamage`/`OnDamageStateChanged`, `SetHealth` caveats)
