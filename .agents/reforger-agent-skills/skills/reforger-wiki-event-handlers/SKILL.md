---
name: reforger-wiki-event-handlers
description: "Trigger: GetOnPlayerSpawned, SCR_BaseGameMode, event mask, AddEventMask, GetGame. Event handler patterns: ScriptInvoker getters, EventHandlerManagerComponent, and IEntity event list."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "GetOnPlayerSpawned"
    - "SCR_BaseGameMode"
    - "event mask"
    - "AddEventMask"
    - "GetGame"
---

## Activation Contract

Load this skill when code subscribes to or fires game events via `ScriptInvoker` getters (e.g., `GetOnPlayerSpawned`), uses `EventHandlerManagerComponent`, or queries the full list of `IEntity` event methods. Do NOT load for low-level `SetEventMask` usage on a single entity (use `reforger-wiki-entity-lifecycle`); load this skill specifically for handler subscription and game-level events.

## Hard Rules

**ScriptInvoker Event Getters**
- NEVER instantiate a `ScriptInvoker` directly as a member. Always use the getter pattern: the class lazily creates the invoker on first call.
- Using `ScriptInvoker` directly (not `ScriptInvokerBase`-derived typed variants) is BAD PRACTICE — use typed wrappers from `SCR_ScriptInvokerHelper.c`.
- Subscribe with `GetOnXXX().Insert(HandlerMethod)` and unsubscribe with `GetOnXXX().Remove(HandlerMethod)` — always remove when the subscriber is deleted.

**EventHandlerManagerComponent — CORRECTED (verified against `scripts/Game/generated/Events/EventHandlerManagerComponent.c`, class definition)**
- This section previously described `EventHandlerManagerComponent` as exposing per-event `GetOnXXX()` ScriptInvoker getters (`GetOnDestroyed()`, etc.). That does NOT match the actual generated class — it has no such getters at all. The real API is a generic, STRING-KEYED bridge between native (gamecode) events and script:
  - `RaiseEvent(string eventName, int argsCount, void param1 = NULL, ...)` — raises a named event with up to 9 positional args (native → script direction; supported arg types: `int`/`float`/`bool`, `string`, `vector`, `Instance` pointers, and `array<float/int/string/vector>`).
  - `RegisterScriptHandler(string eventName, Managed inst, func callback, bool delayed = true, bool singleUse = false)` — subscribes `callback` on `inst` to the named event. Execution is DELAYED by default (runs next frame, not immediately) — any instance passed as an argument must still be valid by then.
  - `RemoveScriptHandler(string eventName, Managed inst, func callback, bool delayed = true)` — unsubscribes.
  - `GetEventHandlers(out notnull array<BaseEventHandler> outEventHandlers)` — lists all registered handlers.
- Correct subscription pattern is by event NAME, not by calling a getter method:
  ```c
  EventHandlerManagerComponent ehmc = EventHandlerManagerComponent.Cast(entity.FindComponent(EventHandlerManagerComponent));
  if (ehmc)
      ehmc.RegisterScriptHandler("OnSomeNativeEvent", this, OnSomeNativeEvent);
  ```
- The individual gameplay events previously listed here (`OnADSChanged`, `OnInspectionModeChanged`, `OnCompartmentEntering/Entered/Left/Leaving`, `OnLightStateChanged`, `OnTurretReload`, `OnWeaponChanged`/`OnMuzzleChanged`/`OnAmmoCountChanged`/`OnMagazineChanged`, `OnProjectileShot`/`OnFiremodeChanged`/`OnWeaponAttachmentChanged`/`OnZeroingChanged`, `OnDayStart`/`OnNightStart`, `OnGrenadeThrown`) are real, but at least the damage-related ones do NOT live on `EventHandlerManagerComponent` — `GetOnDamage()`, `GetOnDamageStateChanged()`, `GetOnDamageOverTimeAdded()`, `GetOnDamageOverTimeRemoved()` are confirmed to live directly on `SCR_DamageManagerComponent` (see `reforger-wiki-damage-system`). The remaining events in this list were NOT individually re-verified in this pass — treat their exact owning component as unconfirmed until checked against source, rather than assuming they hang off `EventHandlerManagerComponent`.

**IEntity Event Methods (full list)**
- `EOnInit` — entity allocated and initialised
- `EOnVisible` — entity becomes visible
- `EOnFrame` — every simulation frame
- `EOnPostFrame` — after all frame events this frame
- `EOnFixedFrame` — every ~33ms (30 fps fixed)
- `EOnFixedPostFrame` — after fixed frame pass
- `EOnAnimEvent` — animation system event
- `EOnSimulate` — physics sub-step (may run multiple times/frame)
- `EOnPostSimulate` — after simulate pass
- `EOnPhysicsActive` — RigidBody physics activated/deactivated
- `EOnPhysicsMove` — physics engine moved this entity (may run on non-main thread — AVOID cross-entity mutation here)
- `EOnJointBreak` — joint on RigidBody broken
- `EOnTouch` — touched by another entity (requires `TouchComponent`)
- `EOnContact` — contact with another RigidBody registered
- `EOnDiag` — diagnostic frame event (only when Entity Diag is enabled in Diag Menu)
- `EOnUser0-4` — user-defined events (`EntityEvent.EV_USER+0` through `+4`)

**GetGame() Access Pattern**
- `GetGame()` returns the global `BaseGame` instance. Always null-check in early-init code.
- Use `GetGame().GetGameMode()` to obtain `SCR_BaseGameMode` and access its ScriptInvoker getters.

## Key APIs / Patterns

```c
// Subscribing to a game-level ScriptInvoker
class ARGA_MyManager : Managed
{
    void Init()
    {
        // Get game mode and subscribe
        SCR_BaseGameMode gm = SCR_BaseGameMode.Cast(GetGame().GetGameMode());
        if (!gm)
            return;
        gm.GetOnPlayerSpawned().Insert(OnPlayerSpawned);
    }

    void Cleanup()
    {
        SCR_BaseGameMode gm = SCR_BaseGameMode.Cast(GetGame().GetGameMode());
        if (!gm)
            return;
        gm.GetOnPlayerSpawned().Remove(OnPlayerSpawned);
    }

    protected void OnPlayerSpawned(SCR_SpawnRequestComponent requestComponent, IEntity player)
    {
        // Handler logic
    }
}

// EventHandlerManagerComponent subscription — by event NAME, not a GetOnXXX() getter
EventHandlerManagerComponent ehmc = EventHandlerManagerComponent.Cast(
    entity.FindComponent(EventHandlerManagerComponent));
if (ehmc)
    ehmc.RegisterScriptHandler("OnSomeNativeEvent", this, OnSomeNativeEvent);

// For damage events specifically, use SCR_DamageManagerComponent directly instead —
// see reforger-wiki-damage-system.
SCR_DamageManagerComponent dmgMgr = SCR_DamageManagerComponent.Cast(
    entity.FindComponent(SCR_DamageManagerComponent));
if (dmgMgr)
    dmgMgr.GetOnDamageStateChanged().Insert(OnEntityDamageStateChanged);

// ScriptInvoker getter lazy-init pattern (inside a class)
protected ref ScriptInvokerVoid m_OnSomethingHappened;

ScriptInvokerVoid GetOnSomethingHappened()
{
    if (!m_OnSomethingHappened)
        m_OnSomethingHappened = new ScriptInvokerVoid();
    return m_OnSomethingHappened;
}

// Firing the event
void TriggerSomething()
{
    if (m_OnSomethingHappened)
        m_OnSomethingHappened.Invoke();
}
```

## References

- PDF: `Event Handlers – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen coverage gap: this Doxygen build only documents the generic Enfusion engine (`Core`/`GameLib`/`WorkbenchCommon`). `EventHandlerManagerComponent` lives in the Arma Reforger game-specific layer (`Game/`), which is NOT included in the Doxygen HTML at all — it was verified instead against the raw generated source `scripts/Game/generated/Events/EventHandlerManagerComponent.c` (Reforger_1.6.0.119 snapshot, which at the time of this audit was the only local source with the full `Game/` tree). **Outdated as of 2026-08-18:** `Reforger_1.8.0.10` now ships engine *and* gameplay and is the base source for new lookups; the path above stands as the record of what was verified then. **Outdated as of 2026-09-03:** `Reforger_1.8.0.13` is now the base source for new lookups (matches the installed game build); it lacks `Core/`, `WorkbenchCommon/`, `WorkbenchGameCommon/` though, so fall back to `Reforger_1.8.0.10` for those.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Event_Handlers`
