---
name: reforger-wiki-scripting-conf
description: "Trigger: .conf file, UserConfig, ResourceName approach, Object approach, conf loading, ParamString, ParamFloat, Resource.Load conf, SCR_ConfigHelper. Runtime loading and referencing of .conf files from script. Note: SCR_ConfigHelperT (generic) does not exist — the real class is SCR_ConfigHelper (non-generic, path/GUID string utility)."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.2.0"
  triggers:
    - ".conf file"
    - "UserConfig"
    - "ResourceName approach"
    - "Object approach"
    - "conf loading"
    - "ParamString"
    - "ParamFloat"
    - "SCR_ConfigHelper"
    - "SCR_ConfigHelperT"
---

## Activation Contract

Load this skill when the user asks about:
- Loading a `.conf` file at runtime from script
- The `ResourceName` approach vs. the `Object` (inline ref) approach for .conf references
- `Resource.Load` + `BaseContainerTools.CreateInstanceFromPrefab` pattern for conf files
- Restricting an `[Attribute]` ResourceName picker to `.conf` files of a specific class
- Trade-offs: ResourceName (default value possible, no preview) vs. Object ref (no default, instant preview / drag-drop)

Do NOT load for: creating the config class itself (→ reforger-wiki-config-class), `[Attribute]` / `[BaseContainerProps]` parameter reference (→ reforger-wiki-config-object), BaseContainer read/write API (→ reforger-wiki-base-container).

## Hard Rules

- **CORRECTED**: `SCR_ConfigHelperT<T>.GetConfigObject()` (mentioned in this skill's triggers, and used in `reforger-wiki-base-container`'s example) appears to be **fabricated**. The real, confirmed class is `SCR_ConfigHelper` — no `T`, NOT generic (`class_s_c_r___config_helper.html`, `scripts/Game/Helpers/SCR_ConfigHelper.c`) — and it only has 4 static protected methods for path/GUID string handling: `GetChildBaseContainerByPath()`, `GetChildFromList()`, `SplitConfigPath()`, `GetGUID()`. None of them load or return a typed config object. Use the `Resource.Load` + `BaseContainerTools.CreateInstanceFromPrefab` pattern shown below instead — that one is independently confirmed and safe to use.
- **ResourceName approach**: a default GUID value is possible; requires `Resource.Load` + cast in a loader method; the `params: "conf"` attribute filter restricts the picker to `.conf` files.
- **Object approach**: use `ref TAG_MyConfig m_Config` with `[Attribute()]`; no default value possible; the Config Editor shows a "set class" button and supports drag-and-drop.
- Always validate `resource.IsValid()` before calling `.GetResource().ToBaseContainer()`.
- Always cast the loaded object with a null-check: `TAG_MyConfig.Cast(...)` returns `null` on type mismatch.
- The `Resource` object must stay in scope for the `BaseContainer` to remain valid (see `reforger-wiki-resource-usage`).

## Key APIs / Patterns

```enforce
// --- ResourceName approach (default value supported) ---
class SCR_ExampleEntity : GenericEntity
{
    [Attribute(
        defvalue: "{EAD206A79A774696}Configs/MyConfig.conf",
        params: "conf class=TAG_MyConfig inheritedClasses=false"
    )]
    protected ResourceName m_sConfig;

    TAG_MyConfig LoadConfig()
    {
        if (m_sConfig.IsEmpty())
            return null;

        Resource resource = Resource.Load(m_sConfig);
        if (!resource.IsValid())
            return null;

        return TAG_MyConfig.Cast(
            BaseContainerTools.CreateInstanceFromPrefab(resource.GetResource().ToBaseContainer())
        );
    }
}

// --- Object approach (no default, instant Workbench preview) ---
class SCR_ExampleEntity : GenericEntity
{
    [Attribute()]
    protected ref TAG_MyConfig m_Config;   // drag-drop or "set class" in Config Editor
}
```

## References

- PDF: `Scripting Conf File Usage – Arma Reforger - Bohemia Interactive Community.pdf`
- Corrected via arexplorer.zeroy.com (see reference memory `reforger/arexplorer-online-doxygen`): the real class is `SCR_ConfigHelper` (`class_s_c_r___config_helper.html`, `scripts/Game/Helpers/SCR_ConfigHelper.c`) — non-generic, 4 static protected path/GUID string methods (`GetChildBaseContainerByPath`, `GetChildFromList`, `SplitConfigPath`, `GetGUID`). `SCR_ConfigHelperT<T>.GetConfigObject()` does not exist.
- See also: `reforger-wiki-config-class` (authoring the config class), `reforger-wiki-config-object` (decorator reference), `reforger-wiki-resource-usage` (Resource lifetime rules), `reforger-wiki-base-container` (BaseContainer model)
