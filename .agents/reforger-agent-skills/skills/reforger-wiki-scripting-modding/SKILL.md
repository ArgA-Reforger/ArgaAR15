---
name: reforger-wiki-scripting-modding
description: "Trigger: modded keyword, mod priority, mod load order, addon, modded class override. How the modded keyword integrates into Enforce Script class inheritance and mod load ordering."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "modded keyword"
    - "mod priority"
    - "mod load order"
    - "addon"
    - "modded class override"
---

## Activation Contract

Load this skill when writing, reviewing, or debugging code that uses the `modded` keyword to extend vanilla or other mods' classes, or when reasoning about mod load order, addon dependencies, and class override conflicts. Do NOT activate for plain `class` / `extends` patterns without modding context — use `reforger-wiki-oop-basics` for that.

## Hard Rules

**What `modded` does**
- `modded class Foo` replaces the global definition of `Foo` with an extended version.
- Unlike `extends`, `modded` does NOT create a new type name — the class is still called `Foo`.
- Any code that previously instantiated or used `Foo` now gets the modded version transparently.
- `modded` must target a class that already exists (vanilla or earlier mod). You cannot `modded` a class that hasn't been defined yet.

**`super` chain**
- Inside a `modded` method, `super.MethodName()` calls the PREVIOUS definition of that method (vanilla or a lower-priority mod).
- Always call `super` unless you intentionally want to block the base behaviour.
- Failing to call `super` breaks the chain for all lower-priority mods — this is a mod compatibility anti-pattern.

**Mod load order and priority**
- Mods load in order defined by the addon dependency graph and the game's mod priority list.
- A later-loading mod's `modded` class wraps the earlier-loading mod's version.
- `super` in the later mod calls the earlier mod's version.
- Priority is set in the addon manifest (`addon.gproj` / `mod.gproj`), NOT in code.
- When two mods both `modded` the same class, the one with higher priority (loaded later) becomes the outermost wrapper.

**Limits and restrictions**
- You can only `modded` existing classes — you cannot `modded` interfaces or structs that don't exist.
- `modded` cannot change a class's base class (`extends` target). The inheritance chain is fixed by the original definition.
- Adding new member variables inside `modded` is valid. Removing or renaming existing ones is NOT (breaks binary compatibility).
- `modded sealed` is contradictory — a `sealed` class cannot be modded. You will get a compile error if you try.

**Addon / mod structure**
- Each addon is a folder containing a `addon.gproj` file that declares its `name`, `author`, `dependencies`, and `requiredGameVersion`.
- Scripts from an addon are loaded after all scripts from its dependencies are loaded.
- Circular addon dependencies are forbidden and cause load failure.
- Use `dependencies` in `addon.gproj` to express explicit load order requirements between your addons.

**File placement**
- Modded scripts must live inside your addon's `scripts/` folder.
- The file name should match the modded class name (e.g. `SCR_BaseGameMode.c` inside your addon's `scripts/GameLib/`).
- Do NOT place your modded class files inside the vanilla game's script folder.

**Compatibility guidelines**
- Always call `super` in every overridden method.
- Avoid re-implementing large blocks of vanilla logic — prefer calling super and patching only what differs.
- If you need to block vanilla behaviour, document WHY in a comment adjacent to the missing `super` call.
- Test with and without other mods enabled to catch priority conflicts early.

## Key APIs / Patterns

```enforce
// addon.gproj — dependency declaration
{
    "name": "ARGA_MyMod",
    "author": "ARGA Team",
    "dependencies": ["ArmaReforger"],
    "requiredGameVersion": "1.0.0"
}

// Modded class — same name as vanilla, added behaviour
modded class SCR_BaseGameMode
{
    // New member variable — valid
    protected bool m_bARGA_CustomMode = false;

    // Override method — MUST call super
    override void OnPlayerSpawned(int playerId, IEntity controlledEntity)
    {
        super.OnPlayerSpawned(playerId, controlledEntity);  // REQUIRED
        // Custom logic
        if (m_bARGA_CustomMode)
            DoCustomSetup(playerId, controlledEntity);
    }
}

// BAD — missing super breaks the mod chain for any lower-priority mod
modded class SCR_BaseGameMode
{
    override void OnPlayerSpawned(int playerId, IEntity controlledEntity)
    {
        // MISSING super.OnPlayerSpawned(playerId, controlledEntity);
        // This silently breaks all mods loaded before this one
    }
}
```

## References

- PDF: `Scripting Modding – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting_Modding`
- Related spokes: `reforger-wiki-oop-basics` (class/extends), `reforger-wiki-oop-advanced` (casting in mod chains)
