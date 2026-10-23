---
name: reforger-wiki-scripting-example
description: "Trigger: scripting example, end-to-end example, complete implementation, SCR_TW_. Full end-to-end Enforce Script example: entity + component + game mode integration."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "scripting example"
    - "end-to-end example"
    - "complete implementation"
    - "SCR_TW_"
---

## Activation Contract

Load this skill when you need a concrete, complete reference pattern that shows how all Enforce Script concepts integrate: entity creation, component attachment, game mode hook-up, and data configuration in a single walkthrough. Use this alongside domain-specific spokes for the specific APIs involved.

## Hard Rules

**Example structure — the canonical pattern**
The official Arma Reforger scripting example implements a "treasure hunt" (SCR_TW_) walkthrough that covers:
1. A configurable `SCR_TW_ItemComponent` (script component with `[Attribute]` properties)
2. A `SCR_TW_GameModeComponent` (game mode component that listens for pickups)
3. Data stored in prefab attributes, NOT in hard-coded values
4. Replication for multiplayer correctness (state broadcast via `RplProp` + `BumpMe`)
5. UI feedback via `SCR_HUDManagerComponent` and screen messages

Follow this layering for any complete feature:
```
GameMode (SCR_TW_GameModeComponent)   // orchestrator — knows global state
  └── listens to ScriptInvokers on ItemComponents
Entity + Component (SCR_TW_ItemComponent)   // owner of item state + notifier
  └── publishes events via ScriptInvoker
Config/Prefab (attributes on component)     // all tuneable values live here
```

**Configuration via attributes — mandatory**
- ALL tuneable values (counts, names, resource paths, radii) MUST be `[Attribute]` properties.
- Hard-coded magic values inside method bodies are a code smell — move them to attributes.
- Default values must be sensible without requiring Workbench editing.

**Component initialisation order**
- `EOnInit` fires for entity setup. Component `EOnPostInit` fires AFTER all components are attached.
- Subscribe to other components' ScriptInvokers in `EOnPostInit`, NOT in `EOnInit` (the other component may not exist yet).
- `EOnDelete` must unsubscribe all listeners to avoid dangling callbacks.

**Replication pattern (when needed)**
- Authority (server) owns state and modifies `[RplProp]` variables.
- After modifying a `[RplProp]`, call `Replication.BumpMe()`.
- The `onRplName` callback fires on proxies when the value is synchronised — use it to update the UI or local effects.
- Never modify `[RplProp]` from a proxy — only the authority writes.

**ScriptInvoker subscription pattern**
```enforce
// Subscribe in EOnPostInit (component is ready)
override void EOnPostInit(IEntity owner)
{
    super.EOnPostInit(owner);
    m_pGameMode = SCR_TW_GameModeComponent.Cast(
        GetGame().GetGameMode().FindComponent(SCR_TW_GameModeComponent));
    if (m_pGameMode)
        m_pGameMode.GetOnItemPickedUp().Insert(OnItemPickedUp);
}

// Unsubscribe in EOnDelete (avoid dangling callback)
override void EOnDelete(IEntity owner)
{
    if (m_pGameMode)
        m_pGameMode.GetOnItemPickedUp().Remove(OnItemPickedUp);
    super.EOnDelete(owner);
}
```

**FindComponent usage**
- Always null-check after `FindComponent`. The component may not be present on all variants.
- Cache the result in a member variable (`m_pComponent`) — do NOT call `FindComponent` in tight loops or per-frame.

**Print / debug output during development**
- Use `Print(msg, LogLevel.DEBUG)` for development output — it is stripped in release.
- Remove all `Print` calls before shipping — they are a performance and readability concern.
- For conditional debug: wrap in `#ifdef DEVELOPER` / `#endif` preprocessor guard.

## Key APIs / Patterns

```enforce
// Typed invoker for this component — declared once, outside the class.
// CORRECTED: `ScriptInvoker` is a fixed typedef (`ScriptInvokerBase<func>`), NOT a generic —
// `ScriptInvoker<SCR_TW_ItemComponent>` does not exist and would not compile. Neither engine core
// (scripts/GameLib/tools.c) nor the project helper (scripts/Game/Helpers/SCR_ScriptInvokerHelper.c)
// has a ready-made invoker for a component-typed argument, so define one the same way that file does.
// See reforger-wiki-script-invoker.
void ScriptInvokerItemComponentMethod(SCR_TW_ItemComponent item);
typedef func ScriptInvokerItemComponentMethod;
typedef ScriptInvokerBase<ScriptInvokerItemComponentMethod> ScriptInvokerItemComponent;

// --- SCR_TW_ItemComponent.c ---
class SCR_TW_ItemComponent : ScriptComponent
{
    [Attribute("", UIWidgets.EditBox, "Item display name")]
    protected string m_sItemName;

    [Attribute("1", UIWidgets.Slider, "Point value when collected", "1 10 1")]
    protected int m_iPoints;

    // Lazy-init ScriptInvoker getter pattern
    protected ref ScriptInvokerItemComponent m_OnCollected;
    ScriptInvokerItemComponent GetOnCollected()
    {
        if (!m_OnCollected)
            m_OnCollected = new ScriptInvokerItemComponent();
        return m_OnCollected;
    }

    int GetPoints() { return m_iPoints; }

    void Collect()
    {
        // Authority only
        if (!Replication.IsServer()) return;
        GetOnCollected().Invoke(this);
        SCR_EntityHelper.DeleteEntityAndChildren(GetOwner());
    }
}

// --- SCR_TW_GameModeComponent.c ---
class SCR_TW_GameModeComponent : SCR_BaseGameModeComponent
{
    [RplProp(onRplName: "OnScoreUpdated")]
    protected int m_iTotalScore;

    // ScriptInvokerInt is a real pre-defined typedef from SCR_ScriptInvokerHelper.c —
    // no need to invent one for a plain int argument (unlike the component case above).
    protected ref ScriptInvokerInt m_OnScoreUpdated;
    ScriptInvokerInt GetOnScoreUpdated()
    {
        if (!m_OnScoreUpdated)
            m_OnScoreUpdated = new ScriptInvokerInt();
        return m_OnScoreUpdated;
    }

    protected void OnItemCollected(SCR_TW_ItemComponent item)
    {
        m_iTotalScore += item.GetPoints();
        Replication.BumpMe();
    }

    // RplProp callback — fires on proxies
    protected void OnScoreUpdated()
    {
        GetOnScoreUpdated().Invoke(m_iTotalScore);
    }
}
```

## References

- PDF: `Scripting Example – Arma Reforger - Bohemia Interactive Community.pdf`
- Corrected: `ScriptInvoker<T>` generic syntax does not exist (`ScriptInvoker` is a fixed typedef, not a template) — see `reforger-wiki-script-invoker` for the real typedef family and how to add project-specific ones.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting_Example`
- Related spokes: `reforger-wiki-component`, `reforger-wiki-event-handlers`, `reforger-wiki-multiplayer`, `reforger-wiki-script-invoker`
