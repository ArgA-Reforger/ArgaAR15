---
name: reforger-wiki-serialisation
description: "Trigger: JsonSaveContext, JsonLoadContext, BinarySaveContext, BinaryLoadContext, SCR_JsonSaveContext (deprecated alias), WriteValue, ReadValue, SerializationSave, SerializationLoad, [NonSerialized], SaveToString, LoadFromString, SaveToFile, LoadFromFile. JSON and binary serialisation of objects in Enforce Script."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "2.0.0"
  triggers:
    - "JsonSaveContext"
    - "JsonLoadContext"
    - "BinarySaveContext"
    - "BinaryLoadContext"
    - "SCR_JsonSaveContext"
    - "SCR_JsonLoadContext"
    - "SCR_BinSaveContext"
    - "SCR_BinLoadContext"
    - "WriteValue"
    - "ReadValue"
    - "SerializationSave"
    - "SerializationLoad"
    - "SaveToString"
    - "LoadFromString"
    - "ExportToString"
    - "ImportFromString"
    - "SaveToFile"
    - "LoadFromFile"
---

## Activation Contract

Load this skill when the user asks about:
- Serialising/deserialising data to/from JSON strings or binary files
- `JsonSaveContext` / `JsonLoadContext` (JSON, key-based, order-independent) — `SCR_JsonSaveContext`/`SCR_JsonLoadContext` are deprecated aliases for these, still real but should not be used in new code
- `BinarySaveContext` / `BinaryLoadContext` (binary, order-dependent) — `SCR_BinSaveContext`/`SCR_BinLoadContext` are deprecated aliases
- Automatic object serialisation (all properties serialised by default) vs. custom `SerializationSave`/`SerializationLoad` methods
- `[NonSerialized]` decorator to exclude a field from automatic serialisation
- Passing an empty string as the name parameter for top-level complex struct read/write

Do NOT load for: config class decoration (→ reforger-wiki-config-object), BaseContainer property read/write (→ reforger-wiki-base-container), `[JsonEntry]` / `JsonApiStruct` (→ separate JSON API docs).

## Hard Rules

**MAJOR CORRECTION (verified against arexplorer.zeroy.com, which indexes `Game`/`GameCode` — not covered by the local Doxygen dump)**

`SCR_JsonSaveContext`, `SCR_JsonLoadContext`, `SCR_BinSaveContext`, `SCR_BinLoadContext` are real, but they are **deprecated backwards-compatibility aliases**, declared in `scripts/Game/Plugins/Serialization/BackwardsCompatiblity.c` (file name misspelling is in the actual source) alongside other `[Obsolete]`-marked classes. They did not even exist yet in the older Reforger_1.6.0.119 snapshot in this repo — they were added later specifically as compatibility shims. The CURRENT classes to use in new code:

| Deprecated (this skill's old names) | Current class | Notes |
|---|---|---|
| `SCR_JsonSaveContext` | `JsonSaveContext` | `.ExportToString()` is a deprecated wrapper that just calls `.SaveToString()` — use `SaveToString()` directly. |
| `SCR_JsonLoadContext` | `JsonLoadContext` | `.ImportFromString(data)` is a deprecated wrapper for `.LoadFromString(data)` — use `LoadFromString()` directly. |
| `SCR_BinSaveContext` | `BinarySaveContext` | Note the full word "Binary", not "Bin". |
| `SCR_BinLoadContext` | `BinaryLoadContext` | |

- Real inheritance chain (verified in `_json_save_context_8c_source.html`, `_save_container_context_8c_source.html`, `_save_context_8c_source.html`): `JsonSaveContext : SaveContainerContext : SaveContext : SerializationContext`.
- Confirmed real methods on `SaveContext` (shared base, applies to both JSON and Binary save contexts): `WriteValue(string name, void value)`, `Write(void value)`, `WriteDefault(void value, void defaultValue)`, `WriteValueDefault(string name, void value, void defaultValue)`, `WriteMapKey(string key)`.
- Confirmed real methods on `JsonSaveContext` itself: `SaveToString()`, `SaveToFile(string fileName)`, `SetMaxDecimalPlaces(int)`/`GetMaxDecimalPlaces()` (controls JSON float precision — not previously documented in this skill).
- The load-side equivalents (`ReadValue`, `LoadFromString`, `LoadFromFile` on `JsonLoadContext`/`BinaryLoadContext`) were not independently re-derived in this pass — assumed symmetric with the save side by naming convention, but verify before relying on exact signatures.
- **JSON** uses key names → `ReadValue` order does NOT matter.
- **Binary** ignores key names → `ReadValue` order MUST exactly match `WriteValue` order.
- Objects that will be deserialised must NOT have constructor parameters — the deserialiser cannot pass arguments; the object is instantiated via default constructor.
- `[NonSerialized]` only skips the field during **automatic** (simple) object serialisation. If the class defines `SerializationSave`/`SerializationLoad`, those methods take full control and `[NonSerialized]` is ignored.
- If `SerializationSave` is defined, the save context calls it exclusively — automatic property processing does NOT happen.
- If `SerializationLoad` is defined, the load context calls it exclusively — automatic property processing does NOT happen.
- Always call `context.IsValid()` at the start of `SerializationSave`/`SerializationLoad`.
- Passing an empty string `""` as the name parameter to `WriteValue`/`ReadValue` allows writing/reading a complex top-level struct without a named wrapper key.

## Key APIs / Patterns

```enforce
// JSON save — current class names, not the deprecated SCR_ aliases
JsonSaveContext saveCtx = new JsonSaveContext();
saveCtx.WriteValue("key1", someString);
saveCtx.WriteValue("key2", someInt);
string json = saveCtx.SaveToString();   // SaveToString(), not the deprecated ExportToString()

// JSON load (order-independent)
JsonLoadContext loadCtx = new JsonLoadContext();
loadCtx.LoadFromString(json);   // LoadFromString(), not the deprecated ImportFromString()
string s;
int i;
loadCtx.ReadValue("key2", i);   // order does not matter for JSON
loadCtx.ReadValue("key1", s);

// Binary save — "Binary", not "Bin"
BinarySaveContext binSave = new BinarySaveContext();
binSave.WriteValue("k1", someString);
binSave.WriteValue("k2", someInt);
binSave.SaveToFile("data.bin");

// Binary load (MUST match WriteValue order)
BinaryLoadContext binLoad = new BinaryLoadContext();
binLoad.LoadFromFile("data.bin");
binLoad.ReadValue("k1", s);   // same order as WriteValue
binLoad.ReadValue("k2", i);

// Simple automatic object serialisation (all members serialised)
class MyData : Managed
{
    protected int m_iValue = 42;
    protected string m_sLabel;

    [NonSerialized]
    protected float m_fIgnored = 3.14;   // excluded from auto-serialisation
}

// Custom serialisation — full control, NonSerialized ignored here
class MyData : Managed
{
    protected int m_iValue = 42;
    protected string m_sLabel = "Hello";
    protected float m_fExtra = 3.14;

    bool SerializationSave(BaseSerializationSaveContext context)
    {
        if (!context.IsValid())
            return false;
        context.WriteValue("val", m_iValue);
        context.WriteValue("label", m_sLabel);
        // m_fExtra intentionally omitted
        return true;
    }

    bool SerializationLoad(BaseSerializationLoadContext context)
    {
        if (!context.IsValid())
            return false;
        context.ReadValue("val", m_iValue);
        context.ReadValue("label", m_sLabel);
        return true;
    }
}
```

## References

- PDF: `Serialisation – Arma Reforger - Bohemia Interactive Community.pdf` (uses the now-deprecated `SCR_*` class names; superseded by the corrections above)
- Doxygen coverage gap: this serialization layer lives in `scripts/Game/Plugins/Serialization/` — not covered by the local Doxygen dump. Verified against `arexplorer.zeroy.com`: `_json_save_context_8c_source.html`, `_save_container_context_8c_source.html`, `_save_context_8c_source.html`, and the deprecated-alias file `_backwards_compatiblity_8c_source.html` (see reference memory `reforger/arexplorer-online-doxygen`).
- See also: `reforger-wiki-config-object` (config class decoration), `reforger-wiki-base-container` (BaseContainer property access)
