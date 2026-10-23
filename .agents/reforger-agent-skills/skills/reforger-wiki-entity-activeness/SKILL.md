---
name: reforger-wiki-entity-activeness
description: "Trigger: EOnActivate, EOnDeactivate, ActiveState, SetActiveState, entity activeness. FRAME event and entity active-state management post-0.9.8."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "EOnActivate"
    - "EOnDeactivate"
    - "ActiveState"
    - "SetActiveState"
    - "entity activeness"
---

## Activation Contract

Load this skill when code involves activating/deactivating components or entities, toggling the `FRAME` event mask from `EOnActivate`/`EOnDeactivate`, managing the `EntityFlags.ACTIVE` flag, or diagnosing missing frame events. Do NOT load for unrelated lifecycle work (use `reforger-wiki-entity-lifecycle` for general lifecycle).

## Hard Rules

**Post-0.9.8 Model (current)**
- Components are considered ACTIVE by default when an entity spawns. `EOnActivate` is NOT called automatically on spawn since 0.9.8.
- Each component is responsible for managing its OWN event mask — set `EntityEvent.FRAME` in `OnPostInit` (or `EOnActivate` when re-activated), clear it in `EOnDeactivate`.
- Do NOT set `EntityFlags.ACTIVE` on the owner entity from a component just to enable `FRAME` events. The old pattern that set `ACTIVE` on the entity caused bugs when multiple components competed.

**FRAME Event and Active State**
- Setting a `FRAME` event on a component (or entity) makes the entity simulated — the engine updates bounding boxes and runs other operations.
- The `ACTIVE` flag should be set on entities that are MOVED every frame but do NOT have any component using `FRAME`. Setting it manually just for `FRAME` events is wrong post-0.9.8.

**Component Activation Pattern (current best practice)**
- `OnPostInit`: set `EntityEvent.FRAME` directly — component is active by default.
- `EOnActivate`: re-set the event mask (plus any custom condition check).
- `EOnDeactivate`: clear the event mask so the entity stops simulating when no component needs it.
- Always call `super.EOnActivate(owner)` / `super.EOnDeactivate(owner)` before your logic.

**Pre-0.9.8 Pattern (legacy — do NOT use in new code)**
- Setting `EntityFlags.ACTIVE` via `owner.SetFlags(EntityFlags.ACTIVE)` inside `OnPostInit` was previously required. This is now incorrect — it can prevent proper deactivation of other components that share the same entity.

**Conditional Activation**
- If a component has a custom condition (e.g., `m_bCustomCondition`), check it inside `EOnActivate`/`EOnDeactivate` before setting/clearing the mask. This avoids incorrectly enabling/disabling the frame event when the component is not ready.

**Verified against Doxygen**
- `EntityEvent.FRAME` (`_entity_event_8c.html`) and `EntityFlags.ACTIVE` (`_entity_flags_8c.html`) are two DIFFERENT, real, and independent flags in the engine — not two names for the same thing. That distinction is exactly what this skill is about: don't set `EntityFlags.ACTIVE` just to get `FRAME` events, they are unrelated bit flags on unrelated enums.
- `GenericComponent.Activate(IEntity owner)` (`class_generic_component.html`) is confirmed to internally call `EOnActivate()` — consistent with this skill's activation pattern. Doxygen does not carry version-history prose (like the 0.9.8 behavior change), so that specific historical claim could not be independently verified from this source, but no contradiction was found either.

## Key APIs / Patterns

```c
// GOOD — post-0.9.8 component activeness pattern
class ARGA_MyComponent : ScriptComponent
{
    protected bool m_bCustomCondition = false;

    override void OnPostInit(IEntity owner)
    {
        // Set FRAME directly; component is active by default since 0.9.8
        SetEventMask(owner, EntityEvent.FRAME);
    }

    override void EOnActivate(IEntity owner)
    {
        super.EOnActivate(owner);
        if (!m_bCustomCondition)
            return;
        SetEventMask(owner, EntityEvent.FRAME);
    }

    override void EOnDeactivate(IEntity owner)
    {
        super.EOnDeactivate(owner);
        if (!m_bCustomCondition)
            return;
        ClearEventMask(owner, EntityEvent.FRAME);
    }

    override void EOnFrame(IEntity owner, float timeSlice)
    {
        // Per-frame logic
    }

    void EnableCondition()
    {
        m_bCustomCondition = true;
        // owner not accessible here directly; use GetOwner() if needed
        SetEventMask(GetOwner(), EntityEvent.FRAME);
    }
}

// BAD — pre-0.9.8 anti-pattern (do not copy)
// override void OnPostInit(IEntity owner)
// {
//     SetEventMask(owner, EntityEvent.FRAME);
//     owner.SetFlags(EntityFlags.ACTIVE);   // WRONG — do not do this
// }
```

## References

- PDF: `Entity Activeness – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (source of truth for the flags in this skill): `_entity_flags_8c.html`, `_entity_event_8c.html`, `class_generic_component.html`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Entity_Activeness`
