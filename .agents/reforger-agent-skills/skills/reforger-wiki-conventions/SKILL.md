---
name: reforger-wiki-conventions
description: "Trigger: naming conventions, m_, s_, g_, SCR_, TAG_, Allman style, camelCase, PascalCase, member order. Enforce Script code style, naming, and structural conventions."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "naming conventions"
    - "m_"
    - "s_"
    - "g_"
    - "SCR_"
    - "TAG_"
    - "Allman style"
    - "camelCase"
    - "PascalCase"
    - "member order"
---

## Activation Contract

Load this skill when the context involves code style decisions: naming conventions for classes, files, variables, methods, enums; visibility and ordering of class members; brace style; use of getters/setters; or the `[Attribute]` decorator. Do NOT load for runtime API or networking work.

## Hard Rules

**Case Styles**
- `camelCase` — local variables and method parameters: `int healthValue`, `string playerName`.
- `PascalCase` — class names, method names, enum names: `ARGA_MyComponent`, `GetHealth()`, `ARGA_EMyState`.
- `UPPER_SNAKE_CASE` — constants only: `const int MAX_HEALTH = 100`.
- `snake_case` is NEVER used in Enforce Script conventions.

**Tags (Unique Prefixes)**
- Every class and global function MUST carry a unique tag to prevent conflicts with other mods.
- BIS uses `SCR_`. ARGA uses `ARGA_`. Do NOT use `SCR_` or another team's tag.
- The tag prefix applies to: class names, enum names, modded class content (all methods and variables inside a modded class must carry the tag).
- Enums: `ARGA_E` prefix — e.g. `ARGA_EMyState`. Values use `UPPER_SNAKE_CASE`.

**File and Class Naming**
- File: `ARGA_MyObject.c` — matches the class name exactly (without `.c`).
- Component file/class must end with `Component`: `ARGA_ExampleComponent`.
- Entity file/class must end with `Entity`: `ARGA_ExampleEntity`.
- Singular form only — plural suffix is forbidden: `ARGA_NotificationComponent`, NOT `ARGA_NotificationsComponent`.
- File location: `scripts\Game\` directory (or relevant subdirectory by type).

**Variable Naming Prefixes**

| Scope | Prefix | Example |
|---|---|---|
| Member | `m_` + type prefix | `m_iHealth`, `m_bIsAlive`, `m_aItems` |
| Static | `s_` + type prefix | `s_iCount`, `s_bEnabled` |
| Global | `g_` | `g_MyGlobalVar` (bad practice — avoid) |
| Local / param | none (camelCase) | `int health`, `string name` |
| Constant | none (UPPER_SNAKE_CASE) | `const int MAX_VALUE = 9999` |
| Static constant | none (UPPER_SNAKE_CASE) | `static const int TOTAL = 10` |

Type prefixes: `b` bool, `i` int, `f` float, `s` string/ResourceName/LocalizedString, `e` enum, `v` vector, `a` array/Curve, `m` map. No prefix for class instances, typename, set, Color.

**Method Naming**
- Methods use `PascalCase`: `GetHealth()`, `SetAmmo(int count)`, `OnPlayerSpawned()`.
- Parameters use `camelCase`: `void SetHealth(int health)`, `void Move(vector direction)`.
- Getters: `Get` prefix — `GetHealth()`. Setters: `Set` prefix — `SetHealth(int health)`.

**Brace Style — Allman**
- Opening brace ALWAYS on a new line. No K&R or same-line braces.
- Applies to classes, methods, `if`, `for`, `foreach`, `while`, `switch`.

```c
// correct
void MyMethod()
{
    if (condition)
    {
        DoSomething();
    }
}

// wrong
void MyMethod() {
    if (condition) {
        DoSomething();
    }
}
```

**Member Order (inside a class)**
1. `[Attribute]` decorators and annotated public/protected member variables
2. Public member variables
3. Protected member variables
4. Private member variables
5. Public static variables
6. Protected static variables
7. Private static variables
8. Public constants
9. Protected constants
10. Private constants
11. Methods (constructor first, then public, protected, private)

**Visibility**
- Prefer `protected` for member variables over public.
- Use `private` when the value must not be accessed even by derived classes.
- Never expose a field publicly when a getter/setter can serve the purpose.
- Global variables (`g_`) are bad practice and must not be used outside absolute necessity.

**Getters and Setters**
- Always expose `protected` member variables through getter and setter methods.
- This allows internal implementation to change (e.g., computed health from head + body) without breaking callers.

## Key APIs / Patterns

```c
// Correct class structure
class ARGA_SoldierComponent : ScriptComponent
{
    [Attribute(defvalue: "100", desc: "Initial health value")]
    protected int m_iMaxHealth;

    protected int m_iHealth;
    protected bool m_bIsAlive;

    static protected int s_iSpawnedCount;

    const int HEALTH_MIN = 0;

    void ARGA_SoldierComponent()
    {
        m_iHealth = m_iMaxHealth;
        m_bIsAlive = true;
        s_iSpawnedCount++;
    }

    int GetHealth()
    {
        return m_iHealth;
    }

    void SetHealth(int health)
    {
        if (health < HEALTH_MIN)
            health = HEALTH_MIN;
        m_iHealth = health;
        m_bIsAlive = (m_iHealth > 0);
    }

    static int GetSpawnedCount()
    {
        return s_iSpawnedCount;
    }
};

// Enum convention
enum ARGA_ECombatState
{
    IDLE    = 0,
    PATROL  = 1,
    COMBAT  = 2,
    FLEEING = 3,
}
```

## References

- PDF: `Scripting_ Conventions – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting:_Conventions`
