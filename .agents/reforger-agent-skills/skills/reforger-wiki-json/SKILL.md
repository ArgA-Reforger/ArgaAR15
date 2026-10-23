---
name: reforger-wiki-json
description: "Trigger: JsonSerializer, JsonObjectSerializer, JsonLoadFile, JsonSaveFile, json parsing. JSON format reference for Arma Reforger scripting — syntax rules, data types, and file I/O."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "JsonSerializer"
    - "JsonObjectSerializer"
    - "JsonLoadFile"
    - "JsonSaveFile"
    - "json parsing"
---

## Activation Contract

Load this skill when working with raw JSON format rules, JSON file I/O, or reviewing whether a JSON structure is valid Enfusion-compatible JSON. For object serialisation and struct-based JSON (encode/decode to Enforce Script objects), load `reforger-wiki-json-api-struct` instead.

## Hard Rules

**JSON format**
- The root of any JSON file MUST be an object `{ }`.
- Property names and string values MUST be wrapped in double-quotes `"`.
- The last property in an object or array must NOT have a trailing comma.
- Comments are NOT allowed in JSON. `//` and `/* */` will cause parse errors.
- Whitespace (spaces, tabs, newlines) outside of quotes is insignificant.
- Booleans are the literal values `true` or `false` (lowercase, no quotes).
- Numbers are unquoted. Integers and floats are both valid. Scientific notation is accepted. Numbers must NOT start with a period `.` (use `0.5` not `.5`).
- `null` is the null value (lowercase, no quotes).
- Arrays use `[ ]` and may contain any mix of values, objects, or nested arrays.
- String escape sequences: `\b`, `\f`, `\r`, `\n`, `\t`, `\"`, `\\`, `\/`, `\uXXXX`. Line breaks inside a JSON string are NOT allowed.

**Enfusion-specific limitations**
- Multi-type arrays (e.g. mixing integers and strings in the same array) are NOT supported by Enforce Script JSON tooling even though the JSON standard allows it.
- Maximum accepted data size for REST API calls is 1 MB.

## Key APIs / Patterns

```json
{
    "name": "Player 1",
    "health": 95,
    "isReady": true,
    "position": null,
    "tags": ["infantry", "medic"],
    "stats": {
        "kills": 3,
        "deaths": 1
    }
}
```

**Minimal valid JSON**
```json
{}
```

**Array of objects**
```json
{
    "squad": [
        { "name": "Alpha", "alive": true },
        { "name": "Bravo", "alive": false }
    ]
}
```

**Common mistakes**
```json
// WRONG — trailing comma
{ "a": 1, "b": 2, }

// WRONG — comment
{ "a": 1 /* comment */ }

// WRONG — unquoted key
{ name: "Player" }

// WRONG — starts with period
{ "ratio": .5 }
```

## References

- PDF: `Scripting_ JSON – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting:_JSON`
- See also: `reforger-wiki-json-api-struct` (struct-based encode/decode), `reforger-wiki-serialisation` (SCR_JsonSaveContext)
