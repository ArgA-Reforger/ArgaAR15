---
name: reforger-wiki-entity-lifecycle
description: "Trigger: EOnInit, EOnFrame, EOnDelete, SetEventMask, EntityEvent, ClearEventMask. Full entity and component lifecycle event sequence in Enfusion."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "EOnInit"
    - "EOnFrame"
    - "EOnDelete"
    - "SetEventMask"
    - "EntityEvent"
    - "ClearEventMask"
---

## Activation Contract

Load this skill when code involves entity or component lifecycle events — specifically any `EOnXXX` method override, `SetEventMask`/`ClearEventMask`, or questions about execution order (init, frame, simulate, delete). Do NOT load for pure data/config operations that have no lifecycle concerns.

## Hard Rules

**Event Mask is Required Before EOnXXX Fires**
- Engine lifecycle methods (`EOnInit`, `EOnFrame`, `EOnFixedFrame`, `EOnSimulate`, etc.) are ONLY called if the corresponding `EntityEvent` flag is set via `SetEventMask(EntityEvent.XXX)`.
- For entities: call `SetEventMask` inside the entity constructor.
- For components: call `SetEventMask(owner, EntityEvent.XXX)` inside `OnPostInit(IEntity owner)`.

**Lifecycle Order (simplified — Game mode)**
1. Constructor / `OnPostInit` — object is allocated; set event masks here.
2. `EOnInit` — entity is fully initialised; world is ready. Requires `EntityEvent.INIT` mask.
3. `EOnActivate` (component only) — called when component is activated. Since 0.9.8 components start active by default.
4. **Simulation loop** (repeating):
   - `EOnFrame(owner, timeSlice)` — every render frame. Requires `EntityEvent.FRAME`.
   - `EOnFixedFrame(owner, timeSlice)` — every ~33ms (30 fps fixed). Requires `EntityEvent.FIXEDFRAME` (no underscore — confirmed against the `EntityEvent` enum in `_entity_event_8c.html`).
   - `EOnSimulate(owner, timeSlice)` — every ~16ms (60 fps physics). May run multiple times per render frame. Requires `EntityEvent.SIMULATE`.
   - `EOnPostSimulate` — after simulate pass.
   - `EOnPhysicsMove` — when physics engine moves entity (60 fps). Requires `EntityEvent.PHYSICSMOVE` (no underscore — same correction as above). May run on a non-main thread — avoid cross-entity mutations here.
   - `EOnPostFrame` — after all EOnFrame calls this frame.
   - `EOnFixedPostFrame` — after EOnFixedFrame this frame.
5. `EOnDeactivate` (component only) — called when component is deactivated.
6. Destructor — cleanup happens in the class destructor (`void ~MyEntity()` / `void ~MyComponent()`), NOT through an `EOnDelete` event. Do NOT delete components from an entity destructor; do NOT reference the parent entity from a component destructor (entity already deleted by then).

**Workbench (Edit mode) Additional Events**
- `_WB_SetTransform`, `_WB_OnInit`, `_WB_MakeVisible`, `_WB_AfterWorldUpdate` — only fire in Workbench edit mode; do NOT rely on these in game logic.

**Frame vs. Fixed Frame**
- `EOnFrame`: tied to render FPS — variable time step. Use for visual/UI updates.
- `EOnFixedFrame`: fired 30 times/second regardless of FPS. Use for gameplay logic needing stable timing.
- `EOnSimulate`: fired 60 times/second (configurable in Workbench → Physics Settings → Ticks). Use for physics-tight logic.

**Component Destructor Safety**
- NEVER reference `owner` (the parent entity) inside a component's destructor — by the time the component destructor runs, the entity is already destroyed.
- NEVER call `delete` on components inside an entity's destructor — causes null pointer exceptions.

**Corrected against Doxygen (`_entity_event_8c.html`, `group___entities.html`, `class_i_entity.html`)**
- `EntityEvent.DELETE` does NOT exist, and `IEntity` has no `EOnDelete` method at all in the current engine. The full `EntityEvent` enum is: `TOUCH`, `INIT`, `VISIBLE`, `FRAME`, `POSTFRAME`, `ANIMEVENT`, `SIMULATE`, `POSTSIMULATE`, `JOINTBREAK`, `PHYSICSMOVE`, `CONTACT`, `PHYSICSACTIVE`, `FIXEDFRAME`, `POSTFIXEDFRAME`, `USER3`, `USER4`, `USER5`, `DISABLED`, `ALL`. Cleanup on deletion is handled purely through the class destructor — there is no event mask to set for it.
- The flag names have NO underscore: it's `EntityEvent.FIXEDFRAME` and `EntityEvent.PHYSICSMOVE`, not `FIXED_FRAME`/`PHYSICS_MOVE`. Copy-pasting the old names would fail to compile.
- Only `USER3`, `USER4`, `USER5` exist as `EntityEvent` mask flags — there is no `USER0`/`USER1`/`USER2` flag, even though `IEntity` exposes five callback methods `EOnUser0`...`EOnUser4`. The exact mapping between the 3 named mask flags and the 5 callback methods is not documented in Doxygen; verify at runtime before relying on it (see `reforger-wiki-event-handlers`, which lists all 5 `EOnUserN` methods).

## Key APIs / Patterns

```c
// Entity lifecycle
class ARGA_ExampleEntity : GenericEntity
{
    void ARGA_ExampleEntity(IEntitySource src, IEntity parent)
    {
        // Set masks in constructor
        SetEventMask(EntityEvent.INIT | EntityEvent.FRAME);
    }

    override void EOnInit(IEntity owner)
    {
        // World is ready; safe to query other entities
    }

    override void EOnFrame(IEntity owner, float timeSlice)
    {
        // Called every render frame; timeSlice = seconds since last frame
    }

    void ~ARGA_ExampleEntity()
    {
        // Cleanup happens here, not in an EOnDelete event (it does not exist)
        // do NOT delete child components here
    }
}

// Component lifecycle
class ARGA_ExampleComponent : ScriptComponent
{
    override void OnPostInit(IEntity owner)
    {
        // Set mask on owner — component-level mask management
        SetEventMask(owner, EntityEvent.FRAME);
    }

    override void EOnFrame(IEntity owner, float timeSlice)
    {
        // Frame logic
    }

    // Since 0.9.8 EOnActivate is not called on spawn; components are active by default
    override void EOnActivate(IEntity owner)
    {
        super.EOnActivate(owner);
        SetEventMask(owner, EntityEvent.FRAME);
    }

    override void EOnDeactivate(IEntity owner)
    {
        super.EOnDeactivate(owner);
        ClearEventMask(owner, EntityEvent.FRAME);
    }
}

// EntityEvent flags (no underscore in FIXEDFRAME/PHYSICSMOVE; no DELETE flag exists)
SetEventMask(EntityEvent.FRAME);
SetEventMask(EntityEvent.FIXEDFRAME);
SetEventMask(EntityEvent.SIMULATE);
SetEventMask(EntityEvent.INIT);
```

## References

- PDF: `Entity Lifecycle – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (source of truth for `EntityEvent`/`EOnXXX` in this skill): `_entity_event_8c.html`, `group___entities.html`, `class_i_entity.html`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Entity_Lifecycle`
