---
name: reforger-wiki-scripting-first-steps
description: "Trigger: Print, PrintFormat, Remote Console, Workbench, Script Editor, Output tab. Enforce Script setup, debug output, and basic language constructs for first-time scripting."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "Print"
    - "PrintFormat"
    - "Remote Console"
    - "Workbench"
    - "Script Editor"
    - "Output tab"
---

## Activation Contract

Load this skill when the context involves debugging output (`Print`, `PrintFormat`), the Enforce Script development environment (Workbench, Script Editor, Remote Console), or introductory scripting constructs. Do NOT load for purely architectural or networking work.

## Hard Rules

**Debug Output**
- `Print(value)` — prints any value to the log console. Works with any type.
- `PrintFormat(format, arg1, arg2, ...)` — prints a formatted string. Placeholders are `%1`, `%2`, etc.
- A literal `%` in `PrintFormat` must be doubled: `"5%%"` prints `5%`. Using `Print("5%")` is safe and needs no doubling.
- `PrintFormat` without placeholders still works but is wasteful — prefer `Print` for plain strings.

**Environment**
- Enforce Script is edited in the Enfusion Workbench (Arma Reforger Tools via Steam).
- The Script Editor provides: source editing, Remote Console (execute code at runtime), Output panel (log).
- The Remote Console can execute standalone code snippets without loading a world — ideal for quick tests.
- Simple scripts do not require a world or game session; the Remote Console runs them directly.

**Basic Language Constructs**
- Variable declaration: `type identifier = value;` — e.g., `int myAge = 25;`
- Common types: `int` (whole), `float` (decimal), `string` (text), `bool` (true/false).
- Arrays: `array<string> soldiers = { "Alpha", "Bravo", "Charlie" };` — zero-indexed.
- `if/else`: no `then` keyword — `if (condition) { } else { }`.
- `for` loop: `for (int i; i < count; i++) { }` — `i` auto-initialises to 0.
- `foreach`: `foreach (string item : myArray) { }` or `foreach (int i, string item : myArray) { }`.
- String concatenation: `"I am " + myAge + " years old"` — non-string values are auto-stringified when the left operand is a string.

## Key APIs / Patterns

```c
// Debug output
Print("Hello there!");
PrintFormat("Hello %1, welcome to %2!", "Soldier", "Arma Reforger");
Print("5%");          // safe: prints "5%"
PrintFormat("5%%");   // prints "5%"
PrintFormat("%1%%", 5); // prints "5%"

// Variables
int myAge = 25;
float distance = 150.5;
string playerName = "Soldier";
bool isAlive = true;

// Array and loops
array<string> soldiers = { "Alpha", "Bravo", "Charlie" };
Print(soldiers[0]); // "Alpha"

for (int i, count = soldiers.Count(); i < count; i++)
{
    Print(soldiers[i]);
}

foreach (string soldier : soldiers)
{
    Print(soldier);
}

foreach (int i, string soldier : soldiers)
{
    PrintFormat("%1: %2", i, soldier);
}

// Decisions
int health = 75;
if (health > 50)
    Print("Healthy!");
else
    Print("Need medical attention");
```

## References

- Doxygen: `Print`/`PrintFormat` usage patterns confirmed consistent across essentially every real source file examined this session. No bugs found.
- PDF: `Scripting First Steps – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting_First_Steps`
