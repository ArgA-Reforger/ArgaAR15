---
name: reforger-wiki-workbench-plugin
description: "Trigger: WorkbenchPlugin, WorldEditor, editor script, OnMenuOpened, EditorMenu. Creating Workbench editor plugins: menus, tools, and editor-side script utilities."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.3.0"
  triggers:
    - "WorkbenchPlugin"
    - "WorldEditor"
    - "editor script"
    - "OnMenuOpened"
    - "EditorMenu"
---

## Activation Contract

Load this skill when writing or reviewing Workbench editor plugin scripts: custom menus, editor-side utilities, `WorkbenchPlugin` subclasses, `WorldEditor` API usage, or any script that runs inside the editor (not the game). Do NOT apply to game-runtime scripts.

## Hard Rules

**Plugin vs game script**
- Workbench plugins run ONLY inside the editor — never at game runtime.
- Plugin scripts live in `scripts/WorkbenchGame/` (game-side editor scripts) or `scripts/Workbench/` (pure editor tools).
- NEVER import or depend on gameplay classes (like `SCR_BaseGameMode`) inside a plugin class — they may not exist in the editor context.
- Use `[WorkbenchPluginAttribute]` to register a class as a Workbench plugin; without it the class is invisible to the editor.

**`WorkbenchPlugin` anatomy — CONFIRMED against the full generated class body (arexplorer.zeroy.com, `_workbench_plugin_8c_source.html`, `scripts/GameLib/generated/WorkbenchAPI/Plugins/WorkbenchPlugin.c`)**
- The complete engine-generated class is: `class WorkbenchPlugin : Managed { private void WorkbenchPlugin(); private void ~WorkbenchPlugin(); event void Run(); event void RunCommandline(); event void Configure(); event void OnResourceContextMenu(notnull array<ResourceName> resources); }` — nothing else.
- `CanRun()` and `GetAttributes()` are CONFIRMED NOT TO EXIST — this is not a search gap, the full class body above is exhaustive (generated file, "Do not modify"). Do not use them to gate menu visibility or plugin availability; there is no such hook.
- `Run()` is the entry point — called when the user triggers the plugin via menu or shortcut. Confirmed real.
- `RunCommandline()` — override for command-line-invoked execution (seen in real plugins like `FlowmapTool`).
- `Configure()` — override for a plugin's settings/configuration entry point (seen in `ViewOrientationTool`).
- `OnResourceContextMenu(array<ResourceName> resources)` — override to add entries to a resource's right-click context menu.
- Subclass `WorkbenchPlugin` and annotate with `[WorkbenchPluginAttribute(...)]` — confirmed real and widely used (e.g. `WorkbenchPluginAttribute("Regenerate river flow-maps", "Generate and save/overwrite river flow-maps", "", "", {"WorldEditor"}, "", 0xf773)`), though its exact named-parameter list could not be independently confirmed from this Doxygen build (its own class declaration wasn't indexed) — treat parameter names in examples as best-effort, verify against a real usage in the codebase before copying blindly.

**Menu registration**
- Plugins appear in Workbench under the module they declare in `wbModules`.
- `WorldEditor` module: appears in World Editor menus.
- `ResourceManager` module: appears in Resource Manager menus.
- Use `OnMenuOpened(string name)` to initialise plugin state when the menu containing it is opened.

**Editor API access — CORRECTED using arexplorer.zeroy.com (indexes `scripts/Game/`/`GameCode`, which the local Doxygen dump does not cover)**
- Inside `Run()`, get the `WorldEditor` module via `WorldEditor editor = Workbench.GetModule(WorldEditor)`, then call `editor.GetApi()` to obtain the actual `WorldEditorAPI` instance — this is the object that carries selection/action/property methods (mirrors the pattern seen in `SCR_AutotestHarness`: `we.GetApi()`).
- Null-check both the module and `GetApi()` result — the module may not be active or the API may be unavailable outside a loaded world.
- Selection is NOT a single fill-array call. Real confirmed API on `WorldEditorAPI` (`_world_editor_a_p_i_8c_source.html`): `GetSelectedEntitiesCount()` (int) and `GetSelectedEntity(int n = 0)` (returns `IEntitySource`) — iterate by index. Also available: `ClearEntitySelection()`, `AddToEntitySelection(notnull IEntitySource ent)`, `SetEntitySelection(notnull IEntitySource ent)`, `RemoveFromEntitySelection(notnull IEntitySource ent)`, `IsEntitySelected(IEntitySource entity)`, `IsEntitySelectedAsMain(IEntitySource entity)`.

**Undo / redo — CORRECTED**
- The real undo-wrapping methods are `BeginEntityAction(string historyPointName = "", string historyPointIcon = "", void userData = NULL)` and `EndEntityAction(string historyPointName = "")` (both `proto external bool`, confirmed in `_world_editor_a_p_i_8c_source.html`) — NOT `BeginAction()`/`EndAction()` as an earlier version of this skill guessed (those names do not exist anywhere in the engine). There is also a sequence-scoped variant: `BeginEditSequence(IEntitySource entSrc)` / `EndEditSequence(IEntitySource entSrc)`, and `IsDoingEditAction()` to check current state.
- Property edits go through `SetVariableValue(notnull BaseContainer topLevel, array<ref ContainerIdPathEntry> containerIdPath, string key, string value)` — the same method already documented in `reforger-wiki-base-container` — NOT a `ModifyProperty()` call (that name does not exist).

**Error handling in plugins**
- Use `Print(message, LogLevel.ERROR)` for plugin errors — they appear in the Workbench Output tab.
- Never throw unhandled exceptions in `Run()` — it may crash the editor session.
- Always validate preconditions in `CanRun()` before allowing `Run()` to execute.

**Prefab and resource operations**
- Use `ResourceManager.GetModule(ResourceManager)` to access the resource manager in plugin context.
- `editor.Save()` flushes changes to disk — call explicitly after modifications if autosave is not sufficient. (Not independently re-verified against `WorldEditorAPI`'s member list — carried over from the original skill.)
- Batch modifications: open one `BeginEntityAction`, make all changes, then one matching `EndEntityAction`. Do NOT nest actions.

## Key APIs / Patterns

```enforce
// Plugin class with attribute registration
[WorkbenchPluginAttribute(
    name: "ARGA_SetupTool",
    shortcut: "",
    description: "ARGA setup utility for batch entity configuration.",
    wbModules: { "WorldEditor" },
    awesomeFontCode: 0xf013  // gear icon
)]
class ARGA_SetupPlugin : WorkbenchPlugin
{
    // Called when the user clicks the menu item or presses the shortcut
    override void Run()
    {
        WorldEditor editorModule = Workbench.GetModule(WorldEditor);
        if (!editorModule)
        {
            Print("ARGA_SetupPlugin: WorldEditor not available", LogLevel.ERROR);
            return;
        }

        WorldEditorAPI api = editorModule.GetApi();
        if (!api)
        {
            Print("ARGA_SetupPlugin: WorldEditorAPI not available", LogLevel.ERROR);
            return;
        }

        int count = api.GetSelectedEntitiesCount();
        if (count == 0)
        {
            Print("ARGA_SetupPlugin: No entities selected", LogLevel.WARNING);
            return;
        }

        // Wrap all modifications in a single undoable action
        api.BeginEntityAction("ARGA_SetupPlugin_Configure");
        for (int i = 0; i < count; i++)
        {
            IEntitySource src = api.GetSelectedEntity(i);
            ConfigureEntity(api, src);
        }
        api.EndEntityAction("ARGA_SetupPlugin_Configure");
    }

    // NOTE: CanRun() does NOT exist on WorkbenchPlugin — confirmed against the full generated
    // class body (arexplorer.zeroy.com). There is no such hook; do not use it.
    // override bool CanRun() { ... }

    protected void ConfigureEntity(WorldEditorAPI api, IEntitySource src)
    {
        // Modify a property inside the open action
        array<ref ContainerIdPathEntry> path = {};
        api.SetVariableValue(src, path, "propertyName", "newValue");
    }
}
```

**Access World Editor from script**
```enforce
WorldEditor editorModule = Workbench.GetModule(WorldEditor);
if (!editorModule) return;

WorldEditorAPI api = editorModule.GetApi();
if (!api) return;

// Get selected entities — by index, not a fill-array call
for (int i = 0, count = api.GetSelectedEntitiesCount(); i < count; i++)
{
    IEntitySource src = api.GetSelectedEntity(i);
}

// Save world changes
editorModule.Save();
```

## References

- PDF: `Workbench Plugin – Arma Reforger - Bohemia Interactive Community.pdf`
- arexplorer.zeroy.com: `_workbench_plugin_8c_source.html` (`scripts/GameLib/generated/WorkbenchAPI/Plugins/WorkbenchPlugin.c`) — full generated class body confirms `Run`/`RunCommandline`/`Configure`/`OnResourceContextMenu` are the only members; `CanRun`/`GetAttributes` confirmed absent (not a search gap — exhaustive class listing).
- arexplorer.zeroy.com (covers `scripts/Game`/`GameCode`, unlike the local dump — see reference memory `reforger/arexplorer-online-doxygen`): `_world_editor_a_p_i_8c_source.html` — real `WorldEditorAPI` selection/action/property API (`BeginEntityAction`/`EndEntityAction`, `GetSelectedEntity`/`GetSelectedEntitiesCount`, `SetVariableValue`, etc.) that replaced the previously-guessed `BeginAction`/`EndAction`/`GetSelection`/`ModifyProperty` names.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Workbench_Plugin`
- Related spokes: `reforger-wiki-scripting-first-steps` (Workbench setup), `reforger-wiki-entity` (entity API), `reforger-wiki-base-container` (`SetVariableValue` already documented there)
