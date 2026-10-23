---
name: reforger-wiki-prefab-data
description: "Trigger: EntityPrefabData, GetPrefabData, SCR_EntityPrefabData, prefab data, ComponentData, GetComponentData, shared prefab variable. Moving shared read-only data from instance variables to the prefab data class."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.3.0"
  triggers:
    - "EntityPrefabData"
    - "GetPrefabData"
    - "SCR_EntityPrefabData"
    - "prefab data"
    - "ComponentData"
    - "GetComponentData"
    - "shared prefab variable"
---

## Activation Contract

Load this skill when the user asks about:
- Moving member variables from a functional class (`SCR_MyEntity`) to the paired class class (`SCR_MyEntityClass`) to share data across all instances of a prefab
- `GetPrefabData()` for entities and `GetComponentData()` for components
- When it is (and is not) worth paying the performance cost of prefab data lookup
- The naming convention for class classes (e.g. `SCR_AIGroupClass` paired with `SCR_AIGroup`)

Do NOT load for: generic class/inheritance syntax (→ reforger-wiki-oop-basics), BaseContainer reading (→ reforger-wiki-base-container), config-class authoring (→ reforger-wiki-config-class).

## Hard Rules

- The "Class class" (e.g. `SCR_MyEntityClass`) is the prefab data holder; it inherits from the engine Class class (e.g. `ChimeraAIGroupClass`).
- Variables moved to the Class class must be `[Attribute]`-decorated so they are editable per-prefab in Workbench.
- Access prefab data with `GetPrefabData()` (entities) or `GetComponentData()` (components) — always cast and null-check. `GetPrefabData()` is confirmed real on `IEntity` (`proto external EntityPrefabData GetPrefabData()`). `GetComponentData()` is NOT a `GenericComponent` base method (confirmed absent from its full member list) — it's declared per concrete component subtype instead. Confirmed real example: `SCR_CharacterSoundComponent` declares `protected SCR_CharacterSoundComponentClass GetComponentData()` itself (`scripts/Game/Components/SCR_CharacterSoundComponent.c:132`, verified via arexplorer.zeroy.com), returning its OWN specific `...Class` type rather than a generic one. Expect to declare (or find) a `GetComponentData()` override on the specific component you're working with, typed to that component's own `...Class` pair — do not assume it's inherited for free.
- Returning an invalid sentinel (e.g. `-1`) on null prefab data is the recommended pattern.
- **Performance rule**: use prefab data only for variables that are truly shared and not accessed every frame on many instances; if accessed every frame, cache the prefab data pointer as a member variable (8-byte pointer cost) only when the total memory saved exceeds that overhead.
- Use case: 100+ instances with multiple shared variables → prefab data is worth it. ~10 instances → probably not.

## Key APIs / Patterns

```enforce
// Prefab data class (variables shared across all instances of the prefab)
class SCR_AIGroupClass : ChimeraAIGroupClass
{
    [Attribute("42")]
    int m_iValue;
}

// Functional class — reads shared value via GetPrefabData()
class SCR_AIGroup : ChimeraAIGroup
{
    int GetValue()
    {
        SCR_AIGroupClass prefabData = SCR_AIGroupClass.Cast(GetPrefabData());
        if (!prefabData)
            return -1;   // -1 signals invalid / not set

        return prefabData.m_iValue;
    }
}

// Components declare their own GetComponentData() — confirmed real example:
// SCR_CharacterSoundComponent declares "protected SCR_CharacterSoundComponentClass GetComponentData()"
// with NO parameters (scripts/Game/Components/SCR_CharacterSoundComponent.c:132). Its internal
// implementation was not independently re-derived — only the signature and return type were
// confirmed. Verify the method body against a real component before copying it.
class SCR_MyComponent : ScriptedGameComponent
{
    protected SCR_MyComponentClass GetComponentData();   // declare/find the real implementation

    int GetMaxAmmo()
    {
        SCR_MyComponentClass data = GetComponentData();
        if (!data)
            return 0;
        return data.m_iMaxAmmo;
    }
}
```

## References

- PDF: `Prefab Data – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen: `IEntity.GetPrefabData()` confirmed real, returns `EntityPrefabData` (`proto external EntityPrefabData GetPrefabData()` in `_i_entity_8c_source.html`) — the base mechanism this skill's `SCR_AIGroupClass`/`GetPrefabData()` pattern relies on.
- arexplorer.zeroy.com (covers `Game`/`GameCode`, unlike the local dump): confirmed `GetComponentData()` is declared per concrete component (`SCR_CharacterSoundComponent.GetComponentData()`, `_s_c_r___character_sound_component_8c.html`), not inherited generically from `GenericComponent`.
- See also: `reforger-wiki-oop-basics` (class/inheritance basics), `reforger-wiki-base-container` (BaseContainer model for prefabs)
