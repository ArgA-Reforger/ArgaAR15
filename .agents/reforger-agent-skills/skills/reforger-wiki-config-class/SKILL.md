---
name: reforger-wiki-config-class
description: "Trigger: config asset, config file creation, Workbench config, editable property, SCR_BaseContainerHolder, Resource Manager Create Config. Step-by-step tutorial for authoring a .conf root class and creating the asset in Workbench."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "config asset"
    - "config file creation"
    - "Workbench config"
    - "editable property"
    - "SCR_BaseContainerHolder"
    - "Resource Manager Create Config"
---

## Activation Contract

Load this skill when the user asks about:
- How to create a `.conf` file in the Workbench Resource Manager
- Making a class appear as a selectable root in the "Create Config" dialog
- The end-to-end workflow: write class → decorate with `[BaseContainerProps(configRoot: true)]` → create asset in Workbench
- `SCR_BaseContainerHolder` or similar config-holder base classes used in the game code

Do NOT load for: the detailed `[BaseContainerProps]` / `[Attribute]` parameter reference (→ reforger-wiki-config-object), loading a .conf at runtime (→ reforger-wiki-scripting-conf), BaseContainer read/write API (→ reforger-wiki-base-container).

## Hard Rules

- A class only appears in the "Create Config" dialog when `[BaseContainerProps(configRoot: true)]` is present.
- Member variables without `[Attribute()]` are NOT visible or editable in the Config Editor.
- The class name should follow the project TAG convention: `TAG_MyConfig`.
- `defvalue` is always a string literal, even for numeric types.
- Non-attributed members are compiled and functional in script but invisible to Workbench.

## Key APIs / Patterns

```enforce
// 1. Define the config class with editable properties
[BaseContainerProps(configRoot: true)]
class TAG_SuperConfig
{
    [Attribute(defvalue: "DEFAULT VALUE", category: "Personal Details")]
    string m_sName;

    [Attribute(defvalue: "100", params: "1 500", category: "Damage")]
    int m_iTotalHealth;

    // Not decorated — invisible in Config Editor, usable in script
    bool m_bOtherValue;
}
```

**Workbench workflow to create a .conf asset:**
1. In Resource Browser, navigate to the target directory.
2. Click **Create** → **Config File**.
3. In the class-picker dialog, find and select `TAG_SuperConfig`.
4. After clicking the class name the `.conf` file is created.
5. Double-click the file to open it in the Config Editor and set property values.

## References

- PDF: `Create a Config Class – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (verified accurate, no bugs found): `configRoot` confirmed as a real `BaseContainerProps` constructor parameter, defaulting to `false` (`_base_container_props_8c_source.html`). Full signature and the `NamingConvention` parameter are detailed in `reforger-wiki-config-object`.
- See also: `reforger-wiki-config-object` (full `[BaseContainerProps]`/`[Attribute]` parameter reference), `reforger-wiki-scripting-conf` (loading .conf in script), `reforger-wiki-base-container` (BaseContainer model)
