---
name: reforger-wiki-config-object
description: "Trigger: ConfigObject, [BaseContainerProps], [Attribute], configRoot, NamingConvention, NC_MUST_HAVE_NAME, NC_MUST_HAVE_GUID, uiwidget, ParamEnum, NonSerialized. Decorating config classes and member variables for the Workbench Config Editor. uiwidget values verified against Doxygen: CheckBox (not Checkbox), SpinBox, EditBox, ColorPicker, ResourceNamePicker, Flags, ComboBox, Slider, Object, Coords, Button, Script (there is no 'Callback' widget)."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "ConfigObject"
    - "[BaseContainerProps]"
    - "[Attribute]"
    - "configRoot"
    - "NamingConvention"
    - "NC_MUST_HAVE_NAME"
    - "NC_MUST_HAVE_GUID"
    - "uiwidget"
    - "ParamEnum"
---

## Activation Contract

Load this skill when the user asks about:
- Decorating a class with `[BaseContainerProps]` to make it visible in the Config Editor
- Decorating member variables with `[Attribute]` to expose them in the Config Editor
- `configRoot` parameter, `NamingConvention` enum, `NC_MUST_HAVE_NAME`, `NC_MUST_HAVE_GUID`
- `uiwidget` values: `Checkbox`, `SpinBox`, `EditBox`, `ColorPicker`, `ResourceNamePicker`, `Flags`, `ComboBox`, `Slider`, `Object`, `Coords`, `Callback`
- `ParamEnum` for `ComboBox` / `Flags` widgets
- `[NonSerialized]` decorator on member variables
- `defvalue`, `params`, `desc`, `precision`, `category` parameters of `[Attribute]`

Do NOT load for: creating a .conf file on disk (→ reforger-wiki-config-class), loading a .conf at runtime (→ reforger-wiki-scripting-conf), BaseContainer data structures (→ reforger-wiki-base-container).

## Hard Rules

- Every class used as a Config blueprint **must** have `[BaseContainerProps]`; a class inheriting from a decorated class must also be decorated.
- The minimum viable class decorator is `[BaseContainerProps]` (no parameters).
- The minimum viable member decorator is `[Attribute()]` (no parameters).
- `defvalue` is always a `string` even for `bool` ("0"/"1", not "false"/"true") and numbers.
- There is **no way** to set a default value for an object-type member or for an array (only for the array's new-item template).
- `[NonSerialized]` only affects **simple** automatic object serialisation — it is ignored by custom `SerializationSave`/`SerializationLoad` methods.
- `params` for Number/SpinBox format: `"minValue maxValue step"` (e.g. `"0 100 5"`).
- `params` for Array: `"MaxSize=10"` limits element count.
- `params` for ResourcePicker: `"ext1 ext2 class=ClassName inheritedClasses=false"`.
- Use `enumType` **or** `enums` for ComboBox, not both.

**CORRECTED against Doxygen (`_base_container_props_8c_source.html`, `_core_2attributes_8c_source.html`, `_attribute_8c_source.html`)**
- Full real `BaseContainerProps` constructor: `(string category = "", string description = "", string color = "255 0 0 255", bool visible = true, bool insertable = true, bool configRoot = false, string icon = "", NamingConvention namingConvention = NamingConvention.NC_MUST_HAVE_GUID)`.
- `NamingConvention` (a real, separate parameter — previously listed only as a trigger keyword, never explained) controls what identifier the object must have to be valid in the Config Editor. Its 3 real values: `NC_CAN_HAVE_NAME` (a name is optional), `NC_MUST_HAVE_NAME` (a name is required), `NC_MUST_HAVE_GUID` (the default — the object is identified by a GUID instead of a name). Set it explicitly when your config object should be nameable/must-be-named rather than GUID-identified: `[BaseContainerProps(configRoot: true, namingConvention: NamingConvention.NC_MUST_HAVE_NAME)]`.
- Full real `Attribute` constructor: `(string defvalue = "", string uiwidget = "auto", string desc = "", string params = "", ParamEnumArray enums = NULL, string category = "", int precision = 3, typename enumType = void, bool prefabbed = false)` — confirms `defvalue`/`uiwidget`/`desc`/`params`/`category`/`precision`/`enumType` are all real and match this skill's usage.
- `uiwidget` values are class constants on `UIWidgets` (`Core/attributes.c`) — the real, verified list includes `Auto`, `Hidden`/`None`, `ColorPicker`, `ResourceNamePicker`, `ResourcePickerThumbnail`, `FileNamePicker`, `ResourceAssignArray`, `Date`, `Graph`, `Font`, `SpinBox`, `ComboBox`, `EditComboBox`, `SearchComboBox`, `LocaleEditBox`, `EditBox`, `CheckBox` (capital B — NOT `Checkbox`), `Slider`, `Flags`, `Button`, `Script`, `EditBoxWithButton`, `EditBoxMultiline`, `LODFactorsEdit`, `Object`, `Coords`, `Range`, `TopLevelObject`, `CurveDialog`, `BoundingVolumeEditor`. There is **no `Callback` widget** — that name doesn't exist on `UIWidgets`; the closest real ones for invoking script are `Button` and `Script`.

## Key APIs / Patterns

```enforce
// Minimal config class
[BaseContainerProps()]
class TAG_MyConfig
{
    [Attribute()]
    protected string m_sLabel;
}

// Config root (selectable when creating a .conf file)
[BaseContainerProps(configRoot: true)]
class TAG_MyRootConfig
{
    [Attribute(defvalue: "100", params: "1 500", category: "Stats")]
    int m_iTotalHealth;

    [Attribute(defvalue: "0", desc: "Enable debug output")]
    bool m_bDebug;
}

// Flags widget (power-of-two enums recommended)
[Attribute(uiwidget: UIWidgets.Flags, enums: {
    ParamEnum("Visible", "1"),
    ParamEnum("Traceable", "2"),
    ParamEnum("Relative Y", "4")
})]
int m_iFlags;

// ResourceName picker limited to .conf files of a specific class
[Attribute(params: "conf class=TAG_MyRootConfig inheritedClasses=false")]
ResourceName m_sConfigPath;

// ComboBox driven by an enum typename
[Attribute(uiwidget: UIWidgets.ComboBox, enumType: EMyEnum)]
EMyEnum m_eMode;
```

## References

- PDF: `Scripting_ Config Object – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (source of truth for `[BaseContainerProps]`/`[Attribute]`/`NamingConvention`/`UIWidgets` in this skill): `_base_container_props_8c_source.html`, `_attribute_8c_source.html`, `_core_2attributes_8c_source.html`, `class_u_i_widgets.html`
- See also: `reforger-wiki-config-class` (tutorial for creating .conf root classes), `reforger-wiki-base-container` (BaseContainer data model), `reforger-wiki-scripting-conf` (loading .conf at runtime)
