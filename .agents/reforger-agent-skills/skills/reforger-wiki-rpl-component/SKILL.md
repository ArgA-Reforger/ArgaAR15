---
name: reforger-wiki-rpl-component
description: "Trigger: RplComponent, RplProp, Replication.BumpMe, replication component, RplIdentity. RplComponent / BaseRplComponent Doxygen API reference — role queries, ownership transfer, streaming control."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "RplComponent"
    - "RplProp"
    - "Replication.BumpMe"
    - "replication component"
    - "RplIdentity"
---

## Activation Contract

Load this skill when working directly with `RplComponent` or `BaseRplComponent` APIs: querying entity role, transferring ownership, controlling streaming/relevancy, or obtaining `RplId`/`RplIdentity` values. Complements `reforger-wiki-multiplayer` (which covers RPC patterns and `Replication.*` statics).

## Hard Rules

**Class hierarchy**
- `RplComponentClass : BaseRplComponentClass : GenericComponentClass`
- `RplComponent : BaseRplComponent : GenericComponent`
- `RplComponent` is the concrete class placed on entities in the World Editor.
- All meaningful API lives in `BaseRplComponent` — `RplComponent` itself adds nothing.

**Role queries — use these to guard authority-only or owner-only code**
- `IsMaster()` — returns `true` if this machine hosts the authority entity (server for most objects).
- `IsProxy()` — returns `true` if this machine hosts a proxy (not the authoritative instance).
- `IsOwner()` — returns `true` if this machine is the owner (elevated proxy or authority that is also owner).
- `IsOwnerProxy()` — returns `true` if this is an owner proxy specifically (not the authority).
- `IsRemoteProxy()` — returns `true` if this is a proxy that is NOT the owner.
- `Role()` — returns the `RplRole` enum value directly (`RplRole.Authority` or `RplRole.Proxy`). This is the form Bohemia's own reference examples use (e.g. `if (rplComponent.Role() == RplRole.Authority)`); prefer the boolean helpers above for everyday guards, but expect `Role()` in official sample code.

**Node structure — Head item and item gathering**
- The first item ever inserted into a node is the **Head** item. It is the only item required to implement the replication lifetime callbacks (recreate node contents, destroy it, move it within the node hierarchy, save/load initialization data). For entities, `RplComponent` itself is that Head item.
- During `EOnInit`, `RplComponent` walks the entity it is attached to and gathers every item with a non-empty replication layout (at least one `RplProp`, RPC, or replication callback) into its `RplNode`. Non-replicated entities/components (no RPCs, no replicated state) are skipped and become invisible to replication — they get no `RplId` and cannot be referenced through it.
- If the component's `Recursive` property is OFF, only the entity `RplComponent` is attached to (and its own components) are gathered — child entities are ignored entirely by replication.
- If `Recursive` is ON, child entities are walked too, EXCEPT it stops recursing into any child that has its own `RplComponent` — that child becomes the Head of a separate node instead.
- The **node hierarchy is a flat list per node**, not a mirror of the entity hierarchy — a node with recursive gathering can contain items from several entities, all as siblings in one array.
- **`"Parent Node From Parent Entity"`** (a `RplComponent` property, ON by default) makes this node's parent-in-replication automatically follow its owning entity's parent-in-the-world. Turning it OFF decouples them: the replication scheduler can then stream this sub-hierarchy independently from its visual parent — useful to avoid replicating, say, every item inside a large building whenever the building itself streams in.
- Common prefab pitfalls from this: if the prefab root entity has no `RplComponent`, replication knows nothing about it or its non-replicated children, which breaks spawning it at runtime/JIP (no `RplId` to associate with the prefab's resource GUID). If an entity between two `RplComponent`-owning entities is not itself part of a node, the client can reconstruct a different parent-child relationship than the server has — silent desync, not a crash.

**Identity and IDs**
- `Id()` — returns the `RplId` for this component's entity. Cache the result; `Replication.FindItemId()` uses a table lookup.
- `ChildId(item)` — returns `RplId` for a child item managed within this component.
- `RplIdentity` — represents a connection (machine). Used for ownership transfer.

**Ownership transfer**
- `Give(RplIdentity newOwner)` — transfers ownership of this entity to the specified machine. Call only on the authority.
- Use `Replication.FindOwner(rplId)` to get the current owner's `RplIdentity`.

**Streaming control**
- `EnableSpatialRelevancy(bool)` — enable or disable distance-based relevancy for this node.
- `EnableStreaming(bool)` — enable or disable streaming for this node entirely.
- Spatial relevancy is NOT propagated through entity hierarchy — if a root node is not in the spatial map, child spatial nodes are also excluded.

**Insertion**
- `InsertToReplication(ctx)` — manually insert this component into the replication system.
- `InsertItem(instance)` — insert a managed instance as a child item within this component's replication node.
- `ReleaseFromRpl()` — remove this component from replication. Check `IsReleasedFromRpl()` before using replication API on potentially released components.

**GetNode**
- `GetNode()` — returns the raw `RplNode`. Use for advanced replication-graph operations; prefer the role-query methods for everyday guards.

## Key APIs / Patterns

```enforce
// Get the RplComponent from an entity
RplComponent rpl = RplComponent.Cast(entity.FindComponent(RplComponent));
if (!rpl)
    return; // entity has no replication component

// Guard authority-only code
if (rpl.IsMaster())
{
    // Only runs on the authoritative machine
    m_iValue = newValue;
    Replication.BumpMe();
}

// Guard owner-only code
if (rpl.IsOwner())
{
    Rpc(RpcAsk_RequestChange, newValue);
}

// Transfer ownership to a player's machine
RplIdentity playerIdentity = playerController.GetRplIdentity();
rpl.Give(playerIdentity);

// Get this entity's RplId (cache it — avoid calling in loops)
RplId myId = rpl.Id();
```

## References

- Doxygen: `class_rpl_component_class.html`, `_base_rpl_component_8c_source.html`, `_rpl_identity_8c_source.html`
- Doxygen (source of truth for node/gathering behaviour in this skill): `_page__replication__rpl_node.html`, `_page__replication__entities_and_components.html`
- Related: `reforger-wiki-multiplayer` (RPC patterns, Replication statics, node types Loadtime/Runtime/Local, `RplStateOverride`)
