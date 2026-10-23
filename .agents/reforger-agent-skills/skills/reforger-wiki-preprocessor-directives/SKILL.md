---
name: reforger-wiki-preprocessor-directives
description: "Trigger: #define, #ifdef, #ifndef, #endif, #else, #include, conditional compilation. Preprocessor directives for Enforce Script conditional compilation and file inclusion."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "#define"
    - "#ifdef"
    - "#ifndef"
    - "#endif"
    - "#else"
    - "#include"
    - "conditional compilation"
    - "preprocessor directive"
    - "scrDefine"
---

## Activation Contract

Load this skill when the context involves preprocessor directives (`#define`, `#ifdef`, `#ifndef`, `#else`, `#endif`, `#include`) for conditional compilation or file inclusion in Enforce Script. Do NOT load for regular code without preprocessor tokens.

## Hard Rules

**Directive Reference**

| Directive | Purpose |
|---|---|
| `#define MY_FLAG` | Defines a flag. A flag is either defined or not — it has no value, only presence. |
| `#ifdef MY_FLAG` | Opens a block compiled only if `MY_FLAG` is defined. MUST be closed with `#endif`. |
| `#ifndef MY_FLAG` | Opens a block compiled only if `MY_FLAG` is NOT defined. MUST be closed with `#endif`. |
| `#else` | Opens the opposite-condition block within a `#ifdef`/`#ifndef` scope. |
| `#endif` | Closes a `#ifdef`/`#ifndef`/`#else` scope. Every open scope MUST have exactly one `#endif`. |
| `#include "path/to/file.c"` | Inlines the target file's content at the `#include` location — equivalent to copy-pasting the file. |

**Flags**
- A flag defined via `#define` is binary — it cannot carry a value, only existence.
- Flags can also be defined OUTSIDE of code via startup parameters (`-scrDefine=MY_FLAG`), enabling build-time feature toggles without source changes.

**#include**
- Path is relative and must point to a valid `.c` file.
- The entire content of the included file is substituted at the `#include` location — constants, classes, and methods become part of the including file's scope.
- Use `#include` to share constants or small utilities across multiple classes without duplicating code.

**Scoping**
- Preprocessor directives apply at the file/compile level, NOT at runtime — excluded blocks are never compiled.
- Directives MUST start at the beginning of a line (no inline usage inside expressions).
- Nesting `#ifdef`/`#ifndef` blocks is allowed; each level needs its own `#endif`.

## Key APIs / Patterns

```c
// --- Flag definition and guarded block ---
#define ARGA_DEBUG_MODE

#ifdef ARGA_DEBUG_MODE
    Print("Debug mode is active");
#else
    Print("Release mode");
#endif

// --- ifndef guard (common for include guards) ---
#ifndef ARGA_MY_CONSTANTS_INCLUDED
#define ARGA_MY_CONSTANTS_INCLUDED

static const int ARGA_MAX_PLAYERS = 64;

#endif  // ARGA_MY_CONSTANTS_INCLUDED

// --- #include to share constants ---
// FileToInclude.c:
//   protected static const string MY_PRINT = "Hello there";

class SCR_ScriptedClass
{
#include "scripts/Game/FileToInclude.c"

    void ShowMessage()
    {
        Print(MY_PRINT);  // MY_PRINT is in scope via #include
    }
}
```

## References

- PDF: `Scripting_ Preprocessor Directives – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting:_Preprocessor_Directives`
