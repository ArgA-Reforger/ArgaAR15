---
name: reforger-wiki-entity
description: "Trigger: GenericEntity, IEntity, CreateEntity, SpawnEntityPrefab, EntityManager. Creating and placing scripted entities in Arma Reforger World Editor."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "GenericEntity"
    - "IEntity"
    - "CreateEntity"
    - "SpawnEntityPrefab"
    - "EntityManager"
---

## Activation Contract

Load this skill when creating or editing a World Editor entity class — i.e., any class that extends `GenericEntity` (or another `IEntity`-derived class), uses `EntityEditorProps`, or involves spawning entities at runtime via `GetGame().SpawnEntityPrefab()`. Do NOT load for pure component work that does not involve an entity class declaration.

## Hard Rules

**Entity File Placement**
- Entity script files MUST be in `scripts/Game/` (or a subdirectory). Files placed elsewhere will not appear in the World Editor entity list.
- Entity classnames MUST end with the `Entity` suffix (e.g., `ARGA_MyLogicEntity`).
- Every entity needs both an **Entity Class** (`extends GenericEntityClass`) and the **Entity** itself (`extends GenericEntity`). Place the Class declaration immediately above the Entity declaration in the same file.

**EntityEditorProps**
- Decorate the EntityClass with `[EntityEditorProps(...)]` to make it visible and configurable in World Editor.
- Required field: `category` — the Create tab category where the entity appears (e.g., `"Tutorial/Entities"`).
- Optional fields: `description`, `color`, `visible`, `style` (`"none"`, `"box"`, `"sphere"`, `"cylinder"`, `"capsule"`, `"pyramid"`, `"diamond"`), `sizeMin`, `sizeMax`, `color2`, `dynamicBox`, `icon`.

**Constructor Pattern**
- Use the entity constructor `void EntityName(IEntitySource src, IEntity parent)` — NOT `void OnInit()` or similar.
- Inside the constructor: call `SetEventMask(EntityEvent.FRAME)` (or other events) to enable lifecycle callbacks.
- To implement singleton behaviour: store `static EntityName s_Instance;` and `delete this;` on duplicate instantiation.

**Attribute Properties**
- Expose World Editor-editable fields with `[Attribute(defvalue: "...", desc: "...")]` on protected members.
- For sliders use `uiwidget: UIWidgets.Slider`.
- Only primitives and basic value types are directly attributable; complex types need `[Attribute]` combined with other decorators.

**Event Mask**
- You MUST call `SetEventMask(EntityEvent.FRAME)` in the constructor before `EOnFrame` will fire. No mask = no event.
- Available events (verified against `_entity_event_8c.html`): `EntityEvent.FRAME`, `EntityEvent.INIT`, `EntityEvent.SIMULATE`, `EntityEvent.FIXEDFRAME`, `EntityEvent.PHYSICSMOVE`, `EntityEvent.CONTACT`, `EntityEvent.TOUCH`, `EntityEvent.VISIBLE`, `EntityEvent.POSTFRAME`, `EntityEvent.ANIMEVENT`, `EntityEvent.POSTSIMULATE`, `EntityEvent.JOINTBREAK`, `EntityEvent.PHYSICSACTIVE`, `EntityEvent.POSTFIXEDFRAME`, `EntityEvent.USER3`/`USER4`/`USER5`, `EntityEvent.DISABLED`, `EntityEvent.ALL`. CORRECTED: there is NO `EntityEvent.DELETE` (cleanup is via the destructor, no mask needed), and the flag names have no underscore — it's `FIXEDFRAME`/`PHYSICSMOVE`, not `FIXED_FRAME`/`PHYSICS_MOVE` (see `reforger-wiki-entity-lifecycle` for the same correction with more detail).

**Compile & Reload**
- After creating or modifying entity files, reload via **Shift + F7** (Compile & Reload Scripts) in Workbench to see the entity in the Create tab.

## Key APIs / Patterns

```c
// Entity Class (required — placed above entity definition)
[EntityEditorProps(category: "ARGA/Entities", description: "Example entity", style: "box")]
class ARGA_ExampleEntityClass : GenericEntityClass
{
}

// Entity definition
class ARGA_ExampleEntity : GenericEntity
{
    // Editable property exposed to World Editor
    [Attribute(defvalue: "10", desc: "Interval in seconds")]
    protected int m_iCycleDuration;

    protected float m_fTimer;

    // Constructor — set event masks here
    void ARGA_ExampleEntity(IEntitySource src, IEntity parent)
    {
        SetEventMask(EntityEvent.FRAME);
    }

    // Called every frame (requires EntityEvent.FRAME mask)
    override void EOnFrame(IEntity owner, float timeSlice)
    {
        m_fTimer += timeSlice;
        if (m_fTimer < m_iCycleDuration)
            return;
        m_fTimer = 0;
        // periodic logic here
    }
}

// Runtime spawn
IEntity spawned = GetGame().SpawnEntityPrefab(Resource.Load("{GUID}"), null, null);

// Finding a component on an entity
ARGA_MyComponent comp = ARGA_MyComponent.Cast(entity.FindComponent(ARGA_MyComponent));
```

## References

- PDF: `Create an Entity – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (source of truth for `EntityEvent` in this skill): `_entity_event_8c.html`, `group___entities.html`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Create_an_Entity`
