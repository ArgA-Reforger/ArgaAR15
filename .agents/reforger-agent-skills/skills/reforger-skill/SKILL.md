---
name: reforger-skill
description: "Trigger: *.c, Enforce, Enfusion. Hub router for Arma Reforger Enforce scripting — applies ARGA naming conventions and delegates to spoke skills for specific APIs."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "2.0.0"
  triggers:
    - "*.c"
    - "Enforce"
    - "Enfusion"
---

## Activation Contract

Load this skill when editing or generating `.c` files inside an Enforce/Reforger project. Do not apply to `.json`, `.txt`, or non-Enforce files. This hub applies only global naming and file conventions; all specific API, language, and domain rules are handled by spoke skills loaded alongside this one.

## Hard Rules

**Class & File Naming**
- Classes: `ARGA_` prefix, PascalCase — e.g. `ARGA_MyComponent`.
- Files: `ARGA_` prefix, PascalCase, `.c` extension — matches class name exactly — e.g. `ARGA_MyComponent.c`.
- Class body closing brace MUST end with `;`.
- Enum names: `ARGA_E` prefix, PascalCase — e.g. `ARGA_EMyState`.
- File must be in `scripts\Game\` or the appropriate subdirectory matching its class type.

## Spoke Skills

The following spoke skills cover specific domains. They are loaded alongside this hub when their exclusive trigger keywords appear in context:

| Spoke skill | Exclusive triggers | Domain |
|---|---|---|
| `reforger-wiki-oop-basics` | `class`, `extends`, `modded`, `sealed`, inheritance | OOP class model, constructors, visibility |
| `reforger-wiki-scripting-first-steps` | `Print`, `PrintFormat`, `Remote Console`, Workbench setup | Scripting environment and debug output |
| `reforger-wiki-values` | `int`, `float`, `bool`, `string`, `vector`, `array`, `map`, `set`, `typename`, `enum`, `const`, `ref` | Types, values, scope, casting |
| `reforger-wiki-operators` | `&&`, `\|\|`, `<<`, `>>`, `%`, `==`, `!=`, `+=`, bitwise | Operators and precedence |
| `reforger-wiki-keywords` | `override`, `out`, `inout`, `notnull`, `owned`, `auto`, `new`, `delete`, `thread`, `super`, `vanilla`, `proto`, `native`, `volatile`, `event`, `typedef` | Language keywords |
| `reforger-wiki-conventions` | naming conventions, `m_`, `s_`, `g_`, `SCR_`, `TAG_`, Allman style, `camelCase`, `PascalCase` | Code style and order conventions |
| `reforger-wiki-sqf-to-enforce` | SQF, `forEach`, `while`, `switch`, `hint`, migration, Arma 3 | SQF → Enforce Script migration |
| `reforger-wiki-oop-advanced` | `Cast`, `upcasting`, `downcasting`, `class hierarchy` | Advanced OOP: type casting, class hierarchies, mod load order |
| `reforger-wiki-arc` | `ARC`, `Managed`, `weak reference`, `memory leak`, `cyclic reference`, `reference counting`, `autoptr` | Automatic Reference Counting memory model |
| `reforger-wiki-class-template` | `class template`, `generic class`, `<Class T>`, `Cast helper` | Generic / template class syntax |
| `reforger-wiki-preprocessor-directives` | `#define`, `#ifdef`, `#ifndef`, `#endif`, `#include`, preprocessor | Preprocessor directives |
| `reforger-wiki-preprocessor-macros` | `__FILE__`, `__LINE__`, debug context macros | Built-in preprocessor macros |
| `reforger-wiki-script-invoker` | `ScriptInvoker`, `ScriptInvokerVoid`, `ScriptInvokerBase`, `event handler`, `event subscription` | ScriptInvoker event subscription system |
| `reforger-wiki-base-container` | `BaseContainer`, `GetOwner`, `BaseContainerList`, `BaseContainerTools`, `SCR_BaseContainerTools`, `BaseContainerProps` | BaseContainer data model (Prefab/Config/IEntitySource) |
| `reforger-wiki-config-object` | `ConfigObject`, `[BaseContainerProps]`, `[Attribute]`, `configRoot`, `NamingConvention`, `uiwidget`, `ParamEnum` | Config Editor decorators and widget reference |
| `reforger-wiki-config-class` | `config asset`, `config file creation`, `Workbench config`, `editable property`, `SCR_BaseContainerHolder` | Creating a .conf root class and Workbench asset |
| `reforger-wiki-scripting-conf` | `.conf file`, `UserConfig`, `ResourceName approach`, `Object approach`, `ParamString`, `ParamFloat`, `SCR_ConfigHelperT` | Loading .conf files from script at runtime |
| `reforger-wiki-prefab-data` | `EntityPrefabData`, `GetPrefabData`, `SCR_EntityPrefabData`, `prefab data`, `ComponentData`, `GetComponentData` | Shared prefab data pattern (Class class) |
| `reforger-wiki-resource-usage` | `ResourceName`, `Resource.Load`, `resource lifetime`, `keep resource reference`, `BaseResourceObject` | Safe resource loading and reference lifetime |
| `reforger-wiki-serialisation` | `SCR_JsonSaveContext`, `SCR_JsonLoadContext`, `SCR_BinSaveContext`, `SerializationSave`, `SerializationLoad`, `WriteValue`, `ReadValue` | JSON and binary serialisation |
| `reforger-wiki-entity` | `GenericEntity`, `IEntity`, `CreateEntity`, `SpawnEntityPrefab`, `EntityManager` | Creating and placing scripted entities in World Editor |
| `reforger-wiki-i-entity` | `IEntity interface`, `QueryEntitiesBySphere`, `FindEntityByName`, `GetChildren`, `GetParent` | IEntity base class API: hierarchy, transforms, component lookup |
| `reforger-wiki-entity-lifecycle` | `EOnInit`, `EOnFrame`, `EOnDelete`, `SetEventMask`, `EntityEvent`, `ClearEventMask` | Full entity and component lifecycle event sequence |
| `reforger-wiki-entity-activeness` | `EOnActivate`, `EOnDeactivate`, `ActiveState`, `SetActiveState`, `entity activeness` | FRAME event and entity active-state management (post-0.9.8) |
| `reforger-wiki-component` | `ScriptComponent`, `GameComponent`, `FindComponent`, `EOnPostInit` | Creating and attaching World Editor components |
| `reforger-wiki-event-handlers` | `GetOnPlayerSpawned`, `SCR_BaseGameMode`, `event mask`, `AddEventMask`, `GetGame` | ScriptInvoker getters, EventHandlerManagerComponent, IEntity events |
| `reforger-wiki-input-manager` | `InputManager`, `AddActionListener`, `RemoveActionListener`, `EActionTrigger`, `GetInputManager` | Scripting input actions and contexts via InputManager |
| `reforger-wiki-action-manager` | `ActionManager`, `AddAction`, `RemoveAction`, `PerformAction`, `SCR_ActionsManagerComponent` | ActionManager API for context and action state management |
| `reforger-wiki-multiplayer` | `RplRpc`, `RplChannel`, `RpcAsk_`, `RpcDo_`, `BumpMe`, `JIP`, `Replication.IsServer`, `IsMaster`, `IsProxy` | Multiplayer replication: RPC patterns, authority/proxy/owner model, BumpMe, JIP |
| `reforger-wiki-rpl-component` | `RplComponent`, `RplProp`, `Replication.BumpMe`, `replication component`, `RplIdentity` | RplComponent / BaseRplComponent API: role queries, ownership transfer, streaming control |
| `reforger-wiki-json` | `JsonSerializer`, `JsonObjectSerializer`, `JsonLoadFile`, `JsonSaveFile`, `json parsing` | Raw JSON format rules: syntax, data types, file I/O, Enfusion limitations |
| `reforger-wiki-json-api-struct` | `JsonApiStruct`, `SCR_JsonApiStruct`, `SerializeToJson`, `DeserializeFromJson`, `json struct` | JsonApiStruct encode/decode, file I/O, REST callback payload, error handling |
| `reforger-wiki-rest-api` | `RestCallback`, `RestContext`, `GET request`, `POST request`, `HTTP header` | RestApi scripting: context, GET/POST requests, async callbacks, lifetime rules |
| `reforger-wiki-damage-system` | `SCR_DamageManagerComponent`, `DamageType`, `HitZone`, `EDamageType`, `OnDamage` | Damage system hierarchy, logic flow, DamageEffects, SetHealth caveats |
| `reforger-wiki-damage-effects` | `SCR_DamageEffectComponent`, `damage particle`, `damage sound`, `OnDamageStateChanged`, `DamageEffect` | Visual and audio damage effects tied to entity damage states |
| `reforger-wiki-scripting-modding` | `modded keyword`, `mod priority`, `mod load order`, `addon`, `modded class override` | modded keyword integration, mod load ordering, addon dependency graph |
| `reforger-wiki-scripting-example` | `scripting example`, `end-to-end example`, `complete implementation`, `SCR_TW_` | Full end-to-end Enforce Script example: entity + component + game mode |
| `reforger-wiki-temporary-feature` | `temporary feature`, `feature flag`, `SCR_TemporaryFeature`, `temporary override` | Gating incomplete features behind a runtime flag |
| `reforger-wiki-best-practices` | `best practices`, `code quality`, `avoid null check omission`, `single responsibility` | Design-level coding guidance: null safety, SRP, encapsulation, mod-friendliness |
| `reforger-wiki-dos-donts` | `do's and don'ts`, `forbidden pattern`, `anti-pattern`, `scripting pitfall` | Explicit do/don't checklist for common Enforce Script mistakes |
| `reforger-wiki-performance` | `CallLater`, `GetTickCount`, `performance optimization`, `GC pressure`, `profiling` | Frame budget, deferred calls, allocation avoidance, spatial query throttling |
| `reforger-wiki-doxygen` | `doxygen comment`, `@param`, `@return`, `@code`, `API documentation` | Doxygen comment syntax and conventions for Enforce Script API docs |
| `reforger-wiki-workbench-plugin` | `WorkbenchPlugin`, `WorldEditor`, `editor script`, `OnMenuOpened`, `EditorMenu` | Workbench editor plugins: menus, tools, undo/redo, editor-side script utilities |
| `reforger-source-recon` | `vanilla class`, `vanilla prefab`, `does vanilla already`, `enfusion_tools`, `.pak`, `data007` | Method for reading real vanilla scripts and assets from packed archives with citations |
| `reforger-guid-resolution` | `GUID`, `{GUID}`, `resourceDatabase.rdb`, `GUID to path`, `path to GUID` | Resolve resource GUIDs to paths (and back) from the game's resourceDatabase.rdb |

## References

- `scripts\` — Reforger scripts source of truth
- `reforger-source-recon` — how to read vanilla scripts and assets from the packed archives when no extracted tree is available
- `reforger-guid-resolution` — how to resolve resource GUIDs to paths from the game's own index
- `https://community.bistudio.com/wiki/Category:Arma_Reforger/Modding` — Bohemia Modding
- `https://community.bistudio.com/wikidata/external-data/arma-reforger/ArmaReforgerScriptAPIPublic/` — Arma Reforger Script API
- `https://community.bistudio.com/wikidata/external-data/arma-reforger/EnfusionScriptAPIPublic/` — Enfusion Script API

