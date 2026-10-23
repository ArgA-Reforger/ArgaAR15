---
name: reforger-wiki-performance
description: "Trigger: CallLater, GetTickCount, performance optimization, GC pressure, profiling. Enforce Script performance patterns: frame budget, deferred calls, allocation avoidance, and profiling."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.2.0"
  triggers:
    - "CallLater"
    - "GetTickCount"
    - "performance optimization"
    - "GC pressure"
    - "profiling"
---

## Activation Contract

Load this skill when writing, reviewing, or optimising Enforce Script code for runtime performance: `EOnFrame` budgets, deferred execution with `CallLater`, allocation patterns, string operations, and profiling hooks. Do not activate for game-design or gameplay logic concerns.

## Hard Rules

**Frame budget**
- `EOnFrame` runs EVERY game frame (default ~60 Hz). Any work you do there is multiplied across all entities.
- Keep `EOnFrame` bodies as lean as possible: threshold checks, early returns, cached lookups.
- If a per-frame operation is NOT needed every frame, use a time accumulator or `CallLater` to throttle it.

**`CallLater` — deferred and repeated calls**
- `ScriptCallQueue` is a confirmed real class (`scripts/GameLib/tools.c`) with `CallLater(func fn, int delay = 0, bool repeat = false, void param1=NULL, ...)`, `Call(...)`, `Remove(func fn)`, `GetRemainingTime(func fn)`, `Clear()`, `Dump()` — matches this skill's usage in spirit.
- `GetGame().GetCallqueue()` IS confirmed real: `protected ScriptCallQueue GetCallqueue()`, declared in `scripts/Game/game.c` (the Arma Reforger-specific `Game` extension — this is why it wasn't found in the local Doxygen dump, which doesn't cover `scripts/Game/`; confirmed instead via arexplorer.zeroy.com's source view of that file). This skill's original accessor call was correct.
- Use it for: polling loops, timers, deferred initialisation, periodic state checks.
- `repeat = true` creates a recurring call — MUST be explicitly cancelled via the queue's `Remove(callback)` in `EOnDelete`.
- `delay = 0` defers to the next frame (useful for escaping init-order issues) — but this is NOT free; avoid in tight loops.
- Avoid scheduling `CallLater` from inside another `CallLater` callback in a tight loop — stack overflow risk.

**`GetTickCount` — timing**
- `System.GetTickCount(int prev = 0)` returns milliseconds since engine start (int) — confirmed real (`generated_2_system_2_system_8c_source.html`).
- Use it to measure elapsed time for throttling: `if (System.GetTickCount() - m_iLastUpdate < UPDATE_INTERVAL_MS) return;`
- Do NOT use it for game-time (paused game ≠ real time); use `GetGame().GetTickTime()` for game time delta.

**Allocation and GC pressure**
- Avoid `new` inside `EOnFrame` or tight loops — GC runs are unpredictable and cause frame spikes.
- Pre-allocate buffers/arrays in `EOnPostInit` and clear/reuse them.
- `string` concatenation with `+` allocates a new string each time — use `string.Format()` or a `StringBuilder`-style approach for multi-segment strings.
- Avoid creating short-lived `ref`-counted objects in hot paths; the ARC release also incurs cost.

**Array and map performance**
- `array<T>.Find(value)` is O(n) linear scan — avoid in per-frame code on large arrays.
- Prefer `map<K,V>` for frequent key lookups.
- `array<T>.Clear()` then refill is cheaper than creating a new array if you already hold a `ref`.

**`FindComponent` cost**
- `FindComponent` iterates the entity's component list — O(number of components).
- NEVER call it in `EOnFrame`. Cache in `EOnPostInit` (see `reforger-wiki-best-practices`).

**String operations in hot paths**
- `string` comparison is cheap; `string` construction (concatenation, `string.Format`) is not.
- For log/debug messages in hot paths: guard with `#ifdef DEVELOPER` so they compile out in release.

**Spatial queries**
- `QueryEntitiesBySphere` / `QueryEntitiesByOBB` iterate spatial index — cost depends on entity density.
- Throttle spatial queries to no more than a few times per second unless physics accuracy requires it.
- Cache query results when the answer is unlikely to change between frames.

**Profiling**
- Use `DiagMenu.SetValue(DiagMenuItems.GAME_PROFILE, 1)` to enable in-game profiling overlay (dev builds).
- `Profile` / `ProfileEnd` script macros are available in developer builds for function-level timing.
- Always profile BEFORE optimising — never guess the hot path.

## Key APIs / Patterns

```enforce
// Throttled per-frame update using time accumulator
class ARGA_PeriodicComponent : ScriptComponent
{
    protected static const int UPDATE_INTERVAL_MS = 500;  // 2 Hz
    protected int m_iLastUpdateTick;

    override void EOnFrame(IEntity owner, float timeSlice)
    {
        int now = System.GetTickCount();
        if (now - m_iLastUpdateTick < UPDATE_INTERVAL_MS) return;
        m_iLastUpdateTick = now;

        // Expensive work here (max 2/sec)
        DoPeriodicWork(owner);
    }
}

// CallLater recurring timer — must cancel in EOnDelete
class ARGA_TimerComponent : ScriptComponent
{
    protected static const int TICK_MS = 1000;

    override void EOnPostInit(IEntity owner)
    {
        super.EOnPostInit(owner);
        GetGame().GetCallqueue().CallLater(OnTick, TICK_MS, true);  // repeat = true
    }

    override void EOnDelete(IEntity owner)
    {
        GetGame().GetCallqueue().Remove(OnTick);  // REQUIRED — prevents dangling timer
        super.EOnDelete(owner);
    }

    protected void OnTick()
    {
        // Runs every TICK_MS milliseconds
    }
}

// Pre-allocated reusable array
class ARGA_QueryComponent : ScriptComponent
{
    protected ref array<IEntity> m_aQueryResults = new array<IEntity>();

    void QueryNearby(vector origin, float radius)
    {
        m_aQueryResults.Clear();  // reuse allocation
        GetGame().GetWorld().QueryEntitiesBySphere(origin, radius, OnEntityFound, EQueryEntitiesFlags.ALL);
    }

    protected bool OnEntityFound(IEntity entity)
    {
        m_aQueryResults.Insert(entity);
        return true;  // continue query
    }
}

// CallLater deferred init (escape init-order without EOnPostInit)
GetGame().GetCallqueue().CallLater(DeferredSetup, 0, false);  // next frame, once
```

## References

- PDF: `Scripting_ Performance – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen: `System.GetTickCount(int prev=0)` confirmed real (`generated_2_system_2_system_8c_source.html`); `ScriptCallQueue` class confirmed real (`tools_8c_source.html`).
- arexplorer.zeroy.com (covers `scripts/Game/`, unlike the local dump): `Game.GetCallqueue()` confirmed real — `protected ScriptCallQueue GetCallqueue()` in `scripts/Game/game.c` (`_game_2game_8c.html`). The earlier version of this skill's caveat about this being unconfirmed was wrong — the local dump simply doesn't index this file.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting_Performance`
- Related spokes: `reforger-wiki-best-practices`, `reforger-wiki-dos-donts`, `reforger-wiki-entity-lifecycle`
