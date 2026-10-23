---
name: reforger-wiki-resource-usage
description: "Trigger: ResourceName, Resource.Load, IsLoaded, resource path, IsValid, resource lifetime, keep resource reference, BaseResourceObject, resource null pointer. Safe resource loading patterns and reference lifetime rules."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "ResourceName"
    - "Resource.Load"
    - "IsLoaded"
    - "IsValid"
    - "resource path"
    - "resource lifetime"
    - "keep resource reference"
    - "BaseResourceObject"
    - "resource null pointer"
---

## Activation Contract

Load this skill when the user asks about:
- `Resource.Load(resourceName)` and `resource.IsValid()` checks
- Why a returned `BaseContainer` or `IEntitySource` suddenly becomes `null`
- How to safely load a resource in a loop without dropping references mid-loop
- The rule that a `BaseContainer` cannot be strongly referenced — only the parent `Resource` can
- `BaseContainerTools.CreateInstanceFromPrefab` safe usage

Do NOT load for: editing a BaseContainer's properties (→ reforger-wiki-base-container), loading a .conf file specifically (→ reforger-wiki-scripting-conf), ResourceName picker in Workbench (→ reforger-wiki-config-object).

## Hard Rules

- A `Resource` object is managed **outside** script. When the last script reference to a `Resource` is dropped (end of scope, loop iteration end, etc.), the engine **may dispose** of it and all associated `BaseContainer`/`IEntitySource` objects immediately.
- A `BaseContainer` **cannot** be strongly referenced in script — you must keep the parent `Resource` alive.
- Always call `resource.IsValid()` before using the resource.
- In loops: accumulate `Resource` references in a `array<ref Resource>` alongside the `BaseContainer` array — never let the `Resource` go out of scope while the `BaseContainer` is still in use.
- `BaseContainerTools.CreateInstanceFromPrefab` internally manages the resource; returning its result is safe as long as no external code drops the resource.

## Key APIs / Patterns

```enforce
// WRONG — resource is dropped at scope end, returned BaseContainer may be null
static BaseContainer GetBaseContainer(ResourceName resourceName)
{
    Resource resource = Resource.Load(resourceName);
    if (!resource.IsValid())
        return null;
    return resource.GetResource().ToBaseContainer();  // resource dropped here!
}

// CORRECT — caller must keep 'resource' alive
static BaseContainer GetBaseContainerSafe(ResourceName resourceName, out Resource resource)
{
    resource = Resource.Load(resourceName);
    if (!resource.IsValid())
        return null;
    return resource.GetResource().ToBaseContainer();
}

// WRONG — resource lost at end of each loop iteration
array<BaseContainer> baseContainers = {};
Resource resource;
foreach (ResourceName rn : resourceNames)
{
    resource = Resource.Load(rn);
    if (!resource.IsValid()) continue;
    baseContainers.Insert(resource.GetResource().ToBaseContainer());
    // resource overwritten next iteration — previous BaseContainer may become null
}

// CORRECT — keep all Resource references
array<ref Resource> resources = {};
array<BaseContainer> baseContainers = {};
foreach (ResourceName rn : resourceNames)
{
    Resource res = Resource.Load(rn);
    if (!res.IsValid()) continue;
    resources.Insert(res);
    baseContainers.Insert(res.GetResource().ToBaseContainer());
}
Process(baseContainers);    // all BaseContainers are valid
resources = null;           // release after done

// CreateInstanceFromPrefab pattern (resource managed internally)
static Managed CreateInstanceFromPrefab(ResourceName resourceName)
{
    Resource resource = Resource.Load(resourceName);
    if (!resource.IsValid())
        return null;
    BaseContainer baseContainer = resource.GetResource().ToBaseContainer();
    if (!baseContainer)
        return null;
    return BaseContainerTools.CreateInstanceFromPrefab(baseContainer);
    // resource reference is dropped — safe here because CreateInstanceFromPrefab retains it
}
```

## References

- PDF: `Resource Usage – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (verified accurate, no bugs found): `class Resource : Managed` with `GetResource()` returning `BaseResourceObject`, `IsValid()`, and static `Load(ResourceName name)` all confirmed in `_resource_8c_source.html` — matches this skill's usage exactly.
- See also: `reforger-wiki-base-container` (BaseContainer model), `reforger-wiki-arc` (ARC memory safety, weak references), `reforger-wiki-scripting-conf` (loading .conf files)
