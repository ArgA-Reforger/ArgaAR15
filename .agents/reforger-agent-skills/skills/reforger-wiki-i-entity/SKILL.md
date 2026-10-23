---
name: reforger-wiki-i-entity
description: "Trigger: IEntity interface, QueryEntitiesBySphere, FindEntityByName, GetChildren, GetParent. IEntity base class API reference for Enforce Script."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "IEntity interface"
    - "QueryEntitiesBySphere"
    - "FindEntityByName"
    - "GetChildren"
    - "GetParent"
---

## Activation Contract

Load this skill when code calls methods directly on `IEntity` references — traversing parent/child hierarchy, querying nearby entities, reading/writing transforms, managing event masks, or using `FindComponent` on an `IEntity` instance. Do NOT load for generic entity class declaration (use `reforger-wiki-entity` instead).

## Hard Rules

**IEntity is the Root Entity Interface**
- `IEntity` is the base class for ALL entities. `GenericEntity` extends it. Do not cast to `IEntity` unless you need base-level API access.
- `IEntity` methods are `proto external` — they are engine-implemented, not script-implemented. Never override them directly; override the `EOnXXX` event methods instead.

**Hierarchy Traversal**
- `GetParent()` — returns the direct parent `IEntity`, or null if root.
- `GetRootParent()` — returns the top-most ancestor in the hierarchy.
- `GetChildren()` — returns the FIRST child. Iterate siblings with `GetSibling()`.
- `AddChild(child, pivot, flags)` — adds a child entity; pivot is pivot index or -1 for center. Verified real signature: `int AddChild(notnull IEntity child, TNodeId pivot, EAddChildFlags flags = EAddChildFlags.AUTO_TRANSFORM)` — returns `int`, and `flags` defaults to `AUTO_TRANSFORM` if omitted.
- `RemoveChild(child, keepTransform)` — detaches a child. Verified real signature: `RemoveChild(notnull IEntity child, bool keepTransform = false)`.

**Transform Methods**
- `GetOrigin()` / `SetOrigin(vector orig)` — world-space position.
- `GetTransform(out vector mat[4])` / `SetTransform(vector mat[4])` — full 4-column matrix (3 axis vectors + origin).
- `GetWorldTransform` / `GetLocalTransform` — same family; local is relative to parent.
- `GetYawPitchRoll()` / `SetYawPitchRoll(vector angles)` — Euler angles shortcut.
- `GetAngles()` / `SetAngles(vector angles)` — rotation around X, Y, Z axes.

**Entity Identification**
- `GetName()` / `SetName(string name)` — runtime name (World Editor-assigned).
- `GetID()` — returns `EntityID` (stable engine ID).
- `IsLoaded()` / `IsDeleted()` — state guards; always check `IsDeleted()` before using a cached reference.

**Components**
- `FindComponent(typename typeName)` — returns `Managed`; must `Cast` to specific type.
- `FindComponents(typename, notnull array<Managed> out)` — fills array with all matching components.

**Event Mask Management (on IEntity)**
- `SetEventMask(EntityEvent e)` / `ClearEventMask(EntityEvent e)` / `GetEventMask()`.
- Flags are applied on the entity; you must have the correct mask set before the engine calls `EOnXXX`.

**Flags**
- `SetFlags(EntityFlags flags, bool recursively)` / `ClearFlags(EntityFlags flags, bool recursively)` / `GetFlags()`.
- `EntityFlags.ACTIVE` controls whether the entity participates in the simulation loop (see `reforger-wiki-entity-activeness`).

**SendEvent**
- `SendEvent(notnull IEntity actor, EntityEvent e, void extra)` — dynamically invoke an `EOnXXX` event on any entity at runtime. Volatile call.

## Key APIs / Patterns

```c
// Hierarchy traversal
IEntity parent = entity.GetParent();
IEntity child = entity.GetChildren();   // first child
while (child)
{
    // process child
    child = child.GetSibling();         // walk siblings
}

// Component lookup (always null-check)
ARGA_MyComponent comp = ARGA_MyComponent.Cast(entity.FindComponent(ARGA_MyComponent));
if (!comp)
    return;

// Transform read/write
vector pos = entity.GetOrigin();
entity.SetOrigin(pos + Vector(0, 1, 0));

// Event mask management
entity.SetEventMask(EntityEvent.FRAME);
entity.ClearEventMask(EntityEvent.FRAME);

// State guard before using a cached entity reference
if (!entity || entity.IsDeleted())
    return;

// QueryEntitiesBySphere (called via world, not directly on IEntity)
// See reforger-wiki-component for QueryEntitiesBySphere pattern
BaseWorld world = entity.GetWorld();
// world.QueryEntitiesBySphere(origin, radius, callback, ...)
```

## References

- Doxygen: `class_i_entity.html` — re-verified against the 1.7.0.54 build. All claims in this skill (`GetParent`/`GetRootParent`/`GetChildren`/`GetSibling`, `AddChild`/`RemoveChild`, `SetEventMask`/`ClearEventMask`/`GetEventMask`, `IsLoaded`/`IsDeleted`, `FindComponent`/`FindComponents`, `GetFlags`, `GetID`, `GetYawPitchRoll`, `SendEvent`) matched the real signatures — no bugs found in this skill.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Entity_Lifecycle`
