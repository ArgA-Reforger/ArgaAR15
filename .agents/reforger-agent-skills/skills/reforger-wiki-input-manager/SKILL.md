---
name: reforger-wiki-input-manager
description: "Trigger: InputManager, AddActionListener, RemoveActionListener, EActionTrigger, GetInputManager. Scripting input actions and contexts via InputManager."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "InputManager"
    - "AddActionListener"
    - "RemoveActionListener"
    - "EActionTrigger"
    - "GetInputManager"
---

## Activation Contract

Load this skill when code reads player input through the `InputManager` (action values, listeners, context activation), configures `.conf` input files, or discusses input action types and filters. Do NOT load for generic gameplay logic that happens to call an action result.

## Hard Rules

**InputManager is a Sub-Class of ActionManager**
- `InputManager` extends `ActionManager` and adds the physical input layer (mouse, keyboard, gamepad, joystick).
- Access it via: `GetGame().GetInputManager()` — returns `InputManager`.
- Actions are defined in `chimeraInputCommon.conf`. Never reference raw keys in script — always use named actions.

**Action Types**
- `Digital` — binary (on/off): keyboard, mouse buttons, gamepad digital buttons.
- `Analog` — continuous value: thumbstick, trigger.
- `AnalogRelative` — relative/differential: mouse wheel, relative mouse position.
- `Motion` — absolute position: VR motion controllers, absolute mouse position.
- Action type determines how conflicting inputs are resolved (sum, diff, or value mapping).

**Input Source Classes**
- `InputSourceValue` — single key/input with optional `Filter`.
- `InputSourceSum` — sums values from multiple child `InputSource` items (multiple keys for same action).
- `InputSourceCombo` — combo (all children must be active): blocks other inputs that share a child key while combo is active (e.g., Ctrl+A vs A).
- `InputSourceEmpty` — placeholder source (no input).

**Filters (InputFilter subclasses)**
- `InputFilterValue` — pass through normalised value.
- `InputFilterPressed` — 1.0 when value exceeds threshold, else 0.0.
- `InputFilterDown` — 1.0 ONLY on rising edge (key-down frame).
- `InputFilterUp` — 1.0 ONLY on falling edge (key-up frame).
- `InputFilterClick` — 1.0 on key-up if held less than `HOLD_DURATION`.
- `InputFilterHold` — ramps from 0.0→1.0 over `HOLD_DURATION`; returns 1.0 while held beyond that.
- `InputFilterHoldOnce` — fires once when `HOLD_DURATION` exceeded.
- `InputFilterToggle` — toggles 0.0/1.0 on rising edge.
- `InputFilterDoubleClick` — 1.0 on second rising edge within `DOUBLE_CLICK_DURATION`.
- `InputFilterRepeat` — 1.0 on rising edge, then repeats every `REPEAT_INTERVAL` after `INITIAL_INTERVAL`.

**Contexts**
- Actions are grouped in contexts. A context has a `priority` and flags.
- Context flags: `Overlay` (allows lower-priority contexts to also activate), `CursorVisible` (shows cursor), `Exclusive` (disables other contexts), `ForceCursor`, `CaptureCursor`.
- `InputManager` activates contexts per frame; if two contexts have the same priority and neither has `Overlay`, only the higher-priority one processes.
- Activate with `GetGame().GetInputManager().ActivateContext("ContextName")`.

**Listeners**
- `AddActionListener(actionName, EActionTrigger, callback)` — subscribe a method to an action trigger event.
- `RemoveActionListener(actionName, EActionTrigger, callback)` — unsubscribe. Always remove on cleanup.
- `EActionTrigger` values (verified against `_e_action_trigger_8c_source.html`): `DOWN`, `PRESSED`, `UP`, `VALUE`. CORRECTED: there is no `RELEASED` value — the previous version of this skill listed one that doesn't exist.

**Polling (vs. listeners)**
- `GetActionValue(actionName)` — normalised float value.
- `GetActionTriggered(actionName)` — true if value is above threshold (0.99) and active.
- `IsActionActive(actionName)` — true if action is currently active.

## Key APIs / Patterns

```c
// Access InputManager
InputManager im = GetGame().GetInputManager();

// Activate a context for this frame (call every frame to keep it active)
im.ActivateContext("CharacterMovementContext");

// Subscribe to an action listener
im.AddActionListener("CharacterFire", EActionTrigger.DOWN, OnFirePressed);

// Listener callback signature
protected void OnFirePressed(float value, EActionTrigger trigger)
{
    // value: normalised input value; trigger: what triggered this callback
}

// Remove listener on cleanup
im.RemoveActionListener("CharacterFire", EActionTrigger.DOWN, OnFirePressed);

// Poll action value each frame
float moveX = im.GetActionValue("CharacterMoveX");

// Check if action triggered (above threshold)
if (im.GetActionTriggered("CharacterJump"))
    DoJump();

// Check context active
bool isMoving = im.IsContextActive("CharacterMovementContext");
```

## References

- PDF: `Input Manager – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (source of truth for `EActionTrigger` and inherited `ActionManager` API in this skill): `_e_action_trigger_8c_source.html`, `class_input_manager.html`, `class_action_manager.html` (see `reforger-wiki-action-manager` for the fully-verified method list).
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Input_Manager`
- Config: `chimeraInputCommon.conf` (ArmaReforger:Configs/System/)
