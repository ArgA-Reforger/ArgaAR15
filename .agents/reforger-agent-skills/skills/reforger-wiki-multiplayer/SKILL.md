---
name: reforger-wiki-multiplayer
description: "Trigger: RplRpc, RplChannel, RpcAsk_, RpcDo_, BumpMe, JIP, Replication.IsServer, IsMaster, IsProxy. Multiplayer replication architecture, RPC patterns, and authority/proxy/owner model."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "RplRpc"
    - "RplChannel"
    - "RpcAsk_"
    - "RpcDo_"
    - "BumpMe"
    - "JIP"
    - "Replication.IsServer"
    - "IsMaster"
    - "IsProxy"
---

## Activation Contract

Load this skill when writing or reviewing multiplayer replication code: RPC methods, RplProp synchronisation, authority/proxy/owner logic, BumpMe, JIP, or any `Replication.*` call. Do not activate for single-player-only scripts that never reference replication APIs.

## Hard Rules

**Architecture**
- There is exactly ONE server per session. Clients never communicate directly with each other — all traffic passes through the server.
- In single player the local machine IS the server. Correct multiplayer code must work flawlessly in single player.
- An entity's role (authority / proxy) is set at creation and NEVER changes.
- Only call RPCs from the authority or owner — never attempt to call server-directed RPCs from a plain proxy.
- NEVER call an RPC from `EOnInit`; the RPC system is not ready during initialisation.

**RPC Prefixes (by convention)**
- `RpcAsk_*` — owner asks authority to perform an action (sender = owner, receiver = authority, use `RplRcver.Server`).
- `RpcDo_*` — authority tells proxies to perform an action (sender = authority, receiver = owner or broadcast, use `RplRcver.Owner` / `RplRcver.Broadcast`).

**Channel choice**
- `RplChannel.Reliable` — packet is guaranteed. Use for state-changing events. More expensive.
- `RplChannel.Unreliable` — packet may be lost or superseded. Use for frequent, non-critical updates (e.g. position hints).

**RplProp**
- Decorate with `[RplProp]` only properties that the authority owns and must broadcast.
- After changing a `[RplProp]` property on the authority, call `Replication.BumpMe()` — without it the change is NOT broadcast.
- `onRplName` callback runs on proxies (including JIP proxies) but NEVER on the authority.
- Do NOT call `Replication.BumpMe()` speculatively — only call it when a property was actually modified.
- Note: `RplRcver.Broadcast` runs on all **proxy** machines. If the local machine is both authority AND owner, the RPC does NOT run locally — the authority must call the underlying logic directly if needed.

**RplSave / RplLoad**
- Override `RplSave(ScriptBitWriter)` on the authority to write custom streaming data.
- Override `RplLoad(ScriptBitReader)` on the proxy to read it.
- Read order in `RplLoad` MUST exactly match write order in `RplSave` with the same bit counts, otherwise data is corrupted.
- `[RplProp]` properties are automatically handled by streaming — do NOT duplicate them in `RplSave`/`RplLoad`.

**Codecs (custom types used as RplProp or RPC arguments)**
- Most system types already have a codec implemented. You only need to write one when a user-defined type is used directly as a `[RplProp]` or as an RPC argument.
- A codec is a set of static functions on the type `T`:
  - `Extract(T instance, ScriptCtx ctx, SSnapSerializerBase snapshot)` — copies properties from instance into snapshot (opposite of `Inject`).
  - `Inject(SSnapSerializerBase snapshot, ScriptCtx ctx, T instance)` — copies properties from snapshot into instance (opposite of `Extract`).
  - `Encode(SSnapSerializerBase snapshot, ScriptCtx ctx, ScriptBitSerializer packet)` — compresses snapshot into a packet (opposite of `Decode`).
  - `Decode(ScriptBitSerializer packet, ScriptCtx ctx, SSnapSerializerBase snapshot)` — decompresses packet into snapshot (opposite of `Encode`).
  - `SnapCompare(SSnapSerializerBase lhs, SSnapSerializerBase rhs, ScriptCtx ctx)` — compares two snapshots for equality.
  - `PropCompare(T instance, SSnapSerializerBase snapshot, ScriptCtx ctx)` — compares instance against last snapshot to decide if a new snapshot is needed.
  - Optional: `EncodeDelta`/`DecodeDelta` — delta-encode/decode against a previous snapshot instead of full state.
- Writing a codec is extra work that is prone to bugs over time. Prefer splitting the type into simpler replicated properties instead of writing a codec when: the type is only ever used as one RPC's argument, the codec would just forward to helpers without any bit-packing benefit, or you don't always need to encode all of its properties.

**JIP (Join In Progress)**
- `[RplProp]` properties are automatically synchronised to JIP clients via the streaming system.
- Manual `RplSave`/`RplLoad` data is also streamed to JIP clients when the entity becomes relevant.
- Proxy relevancy (stream in / stream out) is engine-controlled; a broadcast RPC sent while the proxy is not streamed will NOT reach that client.

**Streaming out / relevancy**
- Streaming out while the authority exists is CURRENTLY UNDEFINED BEHAVIOUR — avoid triggering it.
- An entity 5 km away from a player may have its proxy deleted automatically. Broadcast RPCs will not reach that machine.

**Node types (Loadtime / Runtime / Local)**
- **Loadtime** — items already placed in the world when it loads (buildings, street signs). No prefab needed. Their insertion MUST be deterministic on server and every client: the engine validates that `RplId` and type of each loadtime item match on client vs. server, and on mismatch it logs "inconsistent item table" and disconnects that client with `JIP_ERROR`. They may stay out of sync with the server for a long time — the scheduler decides when to stream current state in based on relevancy (e.g. proximity) — except complete removal on server, which always replicates unconditionally. As long as the authority exists on server, a proxy exists on every client (streaming in just syncs its state).
- **Runtime** — items created dynamically on the server from a prefab during the session (vehicles, characters, pickups). May only be inserted on the server (the authority). A proxy may or may not exist on a given client: streaming in creates it, streaming out destroys it; while it exists it receives updates and can send/receive RPCs.
- **Local** — items inserted only on a client during the session (e.g. locally-predicted effects, like showing a fired rocket immediately instead of waiting for the server's authoritative one to stream in). May only be inserted on the client (which is the authority for this item). There is no proxy anywhere else — server and other clients never see it. Arma Reforger disables prefab spawning on clients by default specifically to prevent accidentally creating unintended local items.
- **RplStateOverride** — an `RplComponent` property ("Rpl State Override") that forces a node (and its descendants, evaluated only at the moment of insertion) inserted at loadtime to instead behave as Runtime. Only `None → Runtime` is supported today. Useful for a single prefab mixing static parts (a building) with runtime parts (loot inside it). Do NOT spawn a child entity and attach it afterwards if you rely on this — the override only applies to entities that were already children of the overridden node at insertion time; spawn-then-attach breaks this and is undefined behaviour when it mixes runtime/loadtime state.

**Replication static helpers**
- `Replication.IsServer()` — returns `true` on the server machine (authority for most entities).
- `Replication.IsClient()` — returns `true` on client machines.
- `Replication.BumpMe()` — signals that one or more `[RplProp]` properties changed and should be broadcast.
- `Replication.FindItemId(item)` — returns `RplId` for a managed item (avoid in tight loops; use a cached variable).
- `Replication.FindItem(rplId)` — returns the managed item for an `RplId`.

## Key APIs / Patterns

```enforce
// RplRpc attribute signature
[RplRpc(RplChannel.Reliable, RplRcver.Server)]
void RpcAsk_DoSomething(int param)
{
    // Runs on authority (server)
    m_iValue = param;
    Replication.BumpMe();
}

[RplRpc(RplChannel.Reliable, RplRcver.Broadcast)]
void RpcDo_NotifyProxies(int param)
{
    // Runs on all proxies (not on authority)
}

// Triggering an RPC
Rpc(RpcAsk_DoSomething, 42);    // called owner-side → runs on server
Rpc(RpcDo_NotifyProxies, 42);   // called authority-side → runs on all proxies

// RplProp with callback
[RplProp(onRplName: "OnValueUpdated")]
protected int m_iValue;

protected void OnValueUpdated()
{
    // Called on proxies (including JIP) when m_iValue is synchronised
    // NOT called on authority
}

// RplSave / RplLoad (custom streaming)
override bool RplSave(ScriptBitWriter writer)
{
    writer.Write(m_iSoldierId, 32);
    writer.Write(m_iHealth, 7);   // 7 bits sufficient for 0..100
    writer.WriteBool(m_bHadLunch);
    return true;
}

override bool RplLoad(ScriptBitReader reader)
{
    if (!reader.Read(m_iSoldierId, 32)) return false;
    if (!reader.Read(m_iHealth, 7))    return false;
    if (!reader.ReadBool(m_bHadLunch)) return false;
    return true;
}
```

**RplRcver routing table (RRT) — corrected against Doxygen `_page__replication__overview.html#RemoteProcedureCalls_RoutingTable`**

Where the RPC body actually executes depends on BOTH who calls it (server or client) AND, when called from a client, whether that client is the owner. There are two separate tables — do not collapse them into one "authority vs proxy" table, that loses the owner/not-owner distinction on the client side.

RPC invoked from the server (the authority):

| Is owner | `RplRcver.Server` | `RplRcver.Owner` | `RplRcver.Broadcast` |
|---|---|---|---|
| Owner | On Server | On Server | On all Clients |
| Not Owner | On Server | On Client Owner | On all Clients |

RPC invoked from a client:

| Is owner | `RplRcver.Server` | `RplRcver.Owner` | `RplRcver.Broadcast` |
|---|---|---|---|
| Owner | On Server | Locally | Locally |
| Not Owner | Dropped | Dropped | Locally |

- Note the easy-to-miss case: a **non-owner client** calling an `RplRcver.Broadcast` RPC still runs it **locally** on that same client (it is not dropped) — it just never reaches the server or other clients from that call.
- Calling from the authority when it is also the owner behaves like the "Owner" row of the server table (`RplRcver.Owner` still runs "On Server", i.e. locally, since server and owner are the same machine).

## References

- PDF: `Multiplayer Scripting – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen: `_replication_8c_source.html`, `_rpl_rcver_8c.html`, `_rpl_channel_8c.html`
- Doxygen (source of truth for architecture in this skill): `_page__replication__overview.html`, `_page__replication__loadtime_and_runtime.html`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Multiplayer_Scripting`
