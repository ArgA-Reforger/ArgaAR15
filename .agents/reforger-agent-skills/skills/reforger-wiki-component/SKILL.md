---
name: reforger-wiki-component
description: "Trigger: ScriptComponent, GameComponent, FindComponent, EOnPostInit. Creating and attaching World Editor components in Arma Reforger."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "ScriptComponent"
    - "GameComponent"
    - "FindComponent"
    - "EOnPostInit"
---

## Activation Contract

Load this skill when creating or editing a World Editor component class — any class that extends `ScriptComponent`, uses `ComponentEditorProps`, overrides `OnPostInit`, or involves attaching a component to an entity in World Editor. Do NOT load for pure entity logic without component concerns.

## Hard Rules

**Component File Placement**
- Component script files MUST be in `scripts/Game/` (or a subdirectory). Files placed elsewhere will not appear in the Add Component list.
- Component classnames MUST end with the `Component` suffix (e.g., `ARGA_MyLogicComponent`).
- Every component needs both a **Component Class** (`extends ScriptComponentClass`) and the **Component** itself (`extends ScriptComponent`). Place the Class declaration immediately above the Component declaration.

**ComponentEditorProps**
- Decorate the ComponentClass with `[ComponentEditorProps(...)]` to make it visible in World Editor's Add Component UI.
- Required field: `category` — the path where the component appears (e.g., `"ARGA/Components"`).
- Optional fields: `description`, `color`, `visible`, `insertable`, `icon` (path to PNG, e.g., `"WBData/ComponentEditorProps/componentEditor.png"`).

**Initialization Entry Point**
- `OnPostInit(IEntity owner)` is the component initialization entry point — called after all components on the entity have been created.
- Set `EntityEvent` masks here: `SetEventMask(owner, EntityEvent.FRAME)` to enable `EOnFrame`.
- Initialize collections here: `m_aItems = {};`
- Do NOT use a constructor for initialization logic — use `OnPostInit`.

**EOnFrame Pattern**
- `EOnFrame(IEntity owner, float timeSlice)` fires every render frame if `EntityEvent.FRAME` is set in `OnPostInit`.
- Always call `super.EOnFrame(owner, timeSlice)` as the first line when overriding.
- Use `timeSlice` (seconds since last frame) for frame-rate-independent logic.

**World Queries**
- To find nearby entities: `owner.GetWorld().QueryEntitiesBySphere(origin, radius, callback, null, ...)`.
- The callback signature is `bool CallbackMethod(IEntity e)` — return `true` to continue querying, `false` to stop immediately.
- Use `Cast` to filter by type inside the callback. Returning `false` on a non-match will abort the entire query — only return `false` to deliberately stop iteration early. For "find all matching" intent, always return `true`.

**GetOwner()**
- Inside a component method, `GetOwner()` returns the owning `IEntity`. Do NOT store the owner as a member — use `GetOwner()` when needed.
- Note: `GetOwner()` and `IEntity owner` parameter (passed to `EOnFrame`) are equivalent; prefer the parameter when available.

**Attribute Properties**
- Expose editable fields with `[Attribute(defvalue: "...", desc: "...", category: "...")]`.
- Attribute values are available after `OnPostInit` fires.

**Compile & Reload**
- After creating or modifying component files, reload via **Shift + F7** in Workbench to see the component in the Add Component UI.

## Key APIs / Patterns

```c
// Component Class (required — placed above component definition)
[ComponentEditorProps(category: "ARGA/Components", description: "Example component")]
class ARGA_ExampleComponentClass : ScriptComponentClass
{
}

// Component definition
class ARGA_ExampleComponent : ScriptComponent
{
    [Attribute(defvalue: "10", desc: "Check radius in metres")]
    protected float m_fCheckRadius;

    [Attribute(defvalue: "0.25", desc: "Seconds between checks")]
    protected float m_fCheckPeriod;

    protected float m_fCheckDelay;
    protected ref array<IEntity> m_aNearbyEntities;

    // Initialization — set masks and init collections here
    override void OnPostInit(IEntity owner)
    {
        m_aNearbyEntities = {};
        SetEventMask(owner, EntityEvent.FRAME);
    }

    // Query callback — return true to continue, false to stop
    protected bool OnQueryEntity(IEntity e)
    {
        ARGA_SomeClass sc = ARGA_SomeClass.Cast(e);
        if (sc)
            m_aNearbyEntities.Insert(sc);
        return true;  // always continue iterating
    }

    override void EOnFrame(IEntity owner, float timeSlice)
    {
        super.EOnFrame(owner, timeSlice);

        m_fCheckDelay -= timeSlice;
        if (m_fCheckDelay > 0)
            return;
        m_fCheckDelay = m_fCheckPeriod;

        m_aNearbyEntities.Clear();
        owner.GetWorld().QueryEntitiesBySphere(owner.GetOrigin(), m_fCheckRadius, OnQueryEntity);
    }
}

// Finding a component from outside
ARGA_ExampleComponent comp = ARGA_ExampleComponent.Cast(entity.FindComponent(ARGA_ExampleComponent));
if (!comp)
    return;
```

## References

- PDF: `Create a Component – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (verified accurate, no bugs found): `OnPostInit(IEntity owner)` confirmed on `ScriptComponent` (`class_script_component.html`); `QueryEntitiesBySphere(vector center, float radius, QueryEntitiesCallback addEntity, QueryEntitiesCallback filterEntity = null, EQueryEntitiesFlags flags = ...)` confirmed on `BaseWorld` (`_base_world_8c_source.html`) — matches this skill's usage.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Create_a_Component`
