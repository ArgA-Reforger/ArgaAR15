---
name: reforger-wiki-action-manager
description: "Trigger: ActionManager, AddAction, RemoveAction, PerformAction, SCR_ActionsManagerComponent. ActionManager API for context and action state management."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "ActionManager"
    - "AddAction"
    - "RemoveAction"
    - "PerformAction"
    - "SCR_ActionsManagerComponent"
---

## Activation Contract

Load this skill when code uses `ActionManager` directly to activate contexts/actions, read action values, or register listeners — distinct from the physical `InputManager` layer. Also load when working with `SCR_ActionsManagerComponent` for entity-level action registration. Do NOT load for `.conf` file input configuration (use `reforger-wiki-input-manager` for that layer).

## Hard Rules

**ActionManager is `sealed`**
- `ActionManager` is `sealed` — it cannot be extended. Use composition or access via `GetGame().GetInputManager()` (which returns an `InputManager`, a subclass of `ActionManager`).
- The engine creates and owns the `ActionManager` instance. Never instantiate it manually.

**Core Responsibilities**
- `ActionManager` holds state for registered **Contexts** and **Actions**.
- Contexts group actions by gameplay state (movement, inventory, menu). Only actions in an active context are processed.
- Actions are named strings matching entries in the `.conf` input file.

**Context Activation**
- `ActivateContext(contextName, duration)` — activates a context for `duration` milliseconds (0 = single frame, -1 = permanent until deactivated).
- `IsContextActive(contextName)` — polls current active state.
- To keep a context active across frames, call `ActivateContext` every frame (or use a non-zero duration).

**Action State**
- `GetActionValue(actionName)` — normalised `float` (0.0–1.0) value of the action.
- `GetActionTriggered(actionName)` — true if action value is above threshold (0.99) and the action is active.
- `IsActionActive(actionName)` — true if action is currently in active state.
- `ActivateAction(actionName, duration)` — programmatically activate an action (simulate input).
- `SetActionValue(actionName, float)` — force-set action value (useful for testing or AI).
- `GetActionInputType(actionName)` — returns the `EActionValueType` of the action (missing from earlier version of this skill; confirmed real in `_action_manager_8c_source.html`).

**Action Listeners**
- `AddActionListener(actionName, EActionTrigger, ActionListenerCallback callback)` — register a callback for when the action crosses a trigger threshold.
- `RemoveActionListener(actionName, EActionTrigger, ActionListenerCallback callback)` — always remove on owner deletion to prevent dangling callbacks.
- `ActionListenerCallback` signature: `void Callback(float value, EActionTrigger trigger)`.

**Iteration**
- `GetActionCount()` — total number of registered actions.
- `GetActionName(int index)` — get action name by index (useful for debug enumeration).

**Debug**
- `SetDebug(int debugMode)` — enables DbgUI overlay for action state visualisation. Useful during development.
- `SetContextDebug(contextName, bool)` — per-context debug toggle.

## Key APIs / Patterns

```c
// Access via InputManager (ActionManager subclass)
ActionManager am = GetGame().GetInputManager();

// Activate a context permanently
am.ActivateContext("CharacterMovementContext", -1);

// Activate a context for one frame (call every frame)
am.ActivateContext("InventoryContext");

// Poll action value
float aimValue = am.GetActionValue("CharacterAim");

// Check trigger
if (am.GetActionTriggered("CharacterJump"))
    Jump();

// Register listener
am.AddActionListener("CharacterFire", EActionTrigger.DOWN, OnFire);

protected void OnFire(float value, EActionTrigger trigger)
{
    // Fired on key-down of CharacterFire action
}

// Cleanup — always remove listeners
void OnEntityDelete()
{
    ActionManager am = GetGame().GetInputManager();
    if (am)
        am.RemoveActionListener("CharacterFire", EActionTrigger.DOWN, OnFire);
}

// Programmatic action activation (e.g., AI, testing)
am.ActivateAction("CharacterFire", 100); // active for 100 ms

// Debug all registered actions
int count = am.GetActionCount();
for (int i = 0; i < count; i++)
    Print(am.GetActionName(i));
```

## References

- Doxygen: `class_action_manager.html`, `_action_manager_8c_source.html` — re-verified against 1.7.0.54. Every method in this skill (`SetDebug`, `ActivateContext`, `IsContextActive`, `SetContextDebug`, `ActivateAction`, `IsActionActive`, `GetActionValue`, `GetActionTriggered`, `SetActionValue`, `GetActionCount`, `GetActionName`, `AddActionListener`, `RemoveActionListener`) matched the real `sealed class ActionManager` exactly, including default values (`duration = 0`). Only `GetActionInputType` was missing and has been added.
- Source: `scripts/GameLib/generated/Input/ActionManager.c`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Input_Manager`
