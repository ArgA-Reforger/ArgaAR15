---
name: reforger-wiki-base-container
description: "Trigger: BaseContainer, GetOwner, BaseContainerList, BaseContainerTools, SCR_BaseContainerTools, BaseContainerProps. Data container hierarchy, Config/Prefab/IEntitySource reading and writing patterns."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.3.0"
  triggers:
    - "BaseContainer"
    - "GetOwner"
    - "BaseContainerList"
    - "BaseContainerTools"
    - "SCR_BaseContainerTools"
    - "BaseContainerProps"
---

## Activation Contract

Load this skill when the user asks about:
- `BaseContainer` data holders (Prefab, Config, IEntitySource, WidgetSource, UserSettings, MetaFile)
- Reading/writing BaseContainer properties with `Get`/`Set`
- `BaseContainerList` iteration
- `BaseContainerTools` or `SCR_BaseContainerTools` utility methods
- Creating a container from script: `BaseContainerTools.CreateContainer`
- Editing entity sources via `WorldEditorAPI.SetVariableValue`

Do NOT load for: generic class/inheritance patterns (→ reforger-wiki-oop-basics), resource loading safety (→ reforger-wiki-resource-usage), config class decoration (→ reforger-wiki-config-object).

## Hard Rules

- A `BaseContainer` **cannot** be strongly referenced in script — store the parent `Resource` object to keep the container alive.
- `BaseContainerList` is iterated by index with `.Count()` / `.Get(i)` — it is NOT an `array<BaseContainer>`.
- `IEntitySource` must be edited via `WorldEditorAPI` for changes to persist on save; editing the in-memory `BaseContainer` directly does NOT write back to the Prefab/World file.
- `BaseContainerTools.SaveContainer` accepts either a `ResourceName` or an absolute file path as second argument.
- For non-root-level property access via `WorldEditorAPI.SetVariableValue`, build a `ContainerIdPathEntry` path array first.

## Key APIs / Patterns

```enforce
// Create a container from class name
Resource resource = BaseContainerTools.CreateContainer("GenericEntity");
BaseContainer baseContainer = resource.GetResource().ToBaseContainer();

// Read a property (returns bool — check it)
int value;
if (baseContainer.Get("m_iValue", value))
    Print("read ok");

// Write a property
if (baseContainer.Set("m_iValue", 42))
    Print("write ok");

// Iterate a BaseContainerList
for (int i, count = baseContainerList.Count(); i < count; ++i)
{
    BaseContainer child = baseContainerList.Get(i);
}

// Save container to resource
BaseContainerTools.SaveContainer(baseContainer, resourceName);

// Edit entity source via WorldEditorAPI (persists on save)
worldEditorAPI.SetVariableValue(entitySource, null, "m_iValue", "42");

// Edit a nested property (non-root)
array<ref ContainerIdPathEntry> path = { new ContainerIdPathEntry("m_SubObject") };
worldEditorAPI.SetVariableValue(entitySource, path, "m_iValue", "42");

// CORRECTED: "SCR_ConfigHelperT<T>.GetConfigObject()" appears to be FABRICATED — the real
// class is SCR_ConfigHelper (no "T", NOT generic, scripts/Game/Helpers/SCR_ConfigHelper.c),
// and it only has GetChildBaseContainerByPath(), GetChildFromList(), SplitConfigPath(), GetGUID()
// — path/GUID string utilities, nothing that loads or returns a typed config object. To load a
// typed .conf object, use the confirmed Resource.Load + BaseContainerTools.CreateInstanceFromPrefab
// pattern shown in reforger-wiki-scripting-conf instead:
Resource resource = Resource.Load(resourceName);
if (resource.IsValid())
{
    SCR_ConfigClass configInstance = SCR_ConfigClass.Cast(
        BaseContainerTools.CreateInstanceFromPrefab(resource.GetResource().ToBaseContainer())
    );
}
```

## References

- PDF: `BaseContainer Usage – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (verified accurate, no bugs found): `BaseContainer.Get(string varName, out void val)` / `.Set(string varName, void val)` confirmed (`_base_container_8c_source.html`); `BaseContainerList.Get(int)`/`Set(int, BaseContainer)`/`Insert`/`Remove`/`Count()` confirmed (`_base_container_list_8c_source.html`); `BaseContainerTools.CreateContainer(string typeClass)` returns `ref Resource`, `SaveContainer(BaseContainer cont, ResourceName resourceName, string fileName = "")` confirmed (`_base_container_tools_8c_source.html`).
- Corrected via arexplorer.zeroy.com: `SCR_ConfigHelperT<T>.GetConfigObject()` does not match the real `SCR_ConfigHelper` class (`class_s_c_r___config_helper.html`, `scripts/Game/Helpers/SCR_ConfigHelper.c`) — that class is non-generic and only offers `GetChildBaseContainerByPath()`/`GetChildFromList()`/`SplitConfigPath()`/`GetGUID()`, unrelated to loading a typed config object.
- See also: `reforger-wiki-resource-usage` (Resource lifetime), `reforger-wiki-config-object` (BaseContainerProps decorator), `reforger-wiki-config-class` (creating .conf root classes)
