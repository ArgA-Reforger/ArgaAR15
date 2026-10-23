---
name: reforger-guid-resolution
description: "Trigger: GUID, {GUID}, resource GUID, resolve GUID, GUID to path, path to GUID, resourceDatabase.rdb, rdb, braced GUID. Resolve Arma Reforger resource GUIDs to paths (and back) by parsing the game's own resourceDatabase.rdb, without opening any archive."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "GUID"
    - "{GUID}"
    - "resourceDatabase.rdb"
    - "resolve GUID"
    - "GUID to path"
    - "path to GUID"
---

## Activation Contract

Load this skill when a `.et`, `.conf`, `.ptc` or other project file contains a braced GUID such as `{39859EC13335D884}` and you need its resource path, or when you have a vanilla resource path and need the GUID to reference it. Needs only the installed game, not the modding Tools and not the `.pak` archives. For finding and reading the files themselves, see `reforger-source-recon`. For loading resources from script, see `reforger-wiki-resource-usage`.

## Hard Rules

**Source of truth**
- The game ships its own GUID index: `<SteamLibrary>\steamapps\common\Arma Reforger\addons\data\resourceDatabase.rdb` (~11.5 MB on build 1.8.0.13). It is not an archive; read it whole into memory.
- There are TWO `resourceDatabase.rdb` files (`addons\core\` and `addons\data\`). The one under `data\` is almost always the one wanted.
- Never guess or reconstruct a GUID from memory or from a similar-looking resource. Resolve it from the index.

**Record format** (verified on build 1.8.0.13)
- `<resource path, ASCII>` + `\x00\x06\x00\x00\x00\x00\x00` + 8 bytes GUID (little-endian).
- The braced GUID written in `.et`, `.conf` and `.ptc` files is those 8 bytes REVERSED, as uppercase hex.
- Paths are stored WITHOUT a leading slash (`Particles/Weapon/Explosion_M112.ptc`). A path with a leading slash returns no match. Paths printed by the `enfusion_tools` `glob` command DO start with a slash, so strip it before the lookup.

**Parse with a real program**
- Use Python (or another real language). NEVER use `strings`: it is absent from many Windows shells and FAILS SILENTLY when stderr is redirected, giving an empty result indistinguishable from a genuine "not found".
- A `None`/empty result means "not found in this index for this exact path", not "the resource does not exist". Re-check the path spelling, the missing leading slash and which `.rdb` was read.

**Pin the build**
- A resolved GUID belongs to the build whose `.rdb` produced it. State the build next to any GUID table you write down, and re-resolve when the game version changes.

## Key APIs / Patterns

```python
SEP = b"\x00\x06\x00\x00\x00\x00\x00"

def guid_of(rdb_path, resource_path):
    """Path (no leading slash) -> braced-GUID hex, or None."""
    data = open(rdb_path, "rb").read()
    b = resource_path.encode()
    i = data.find(b + SEP)
    if i < 0:
        return None
    return data[i + len(b) + len(SEP):][:8][::-1].hex().upper()
```

Usage: the result is written in resource references as `{<hex>}<path>`, for example `{8B9323E3EEFBFE99}Particles/Logistics/Explosion/TNT/Explosion_TNT_Small.ptc`.

Reverse lookup (GUID -> path) is the same scan backwards: take the braced hex, convert it to bytes, REVERSE them back to little-endian, locate that 8-byte pattern in the file, then read the ASCII path that precedes the separator.

```python
import re

def path_of(rdb_path, braced_hex):
    """16-hex-digit GUID (as written in files) -> resource path, or None."""
    data = open(rdb_path, "rb").read()
    g = bytes.fromhex(braced_hex)[::-1]
    j = data.find(g)
    if j < 0:
        return None
    k = data.rfind(SEP, 0, j + 1)
    m = None
    for m in re.finditer(rb"[\x20-\x7e]{6,}$", data[max(0, k - 300):k]):
        pass
    return m.group(0).decode() if m else None
```

To enumerate paths, run a `[\x20-\x7e]{6,}` regex over the bytes and filter by extension.

Verified on build 1.8.0.13: `guid_of` returns `8B9323E3EEFBFE99` for `Particles/Logistics/Explosion/TNT/Explosion_TNT_Small.ptc` and `9D8D204CCCC44A38` for `Particles/Weapon/Explosion_M112.ptc`; `path_of` resolves `8B9323E3EEFBFE99` back to the first path. Caveat: a raw 8-byte search can in principle hit unrelated bytes, so for a reverse lookup confirm the returned path ends in a plausible resource extension.

## References

- Related spokes: `reforger-source-recon` (reading the files that contain GUIDs), `reforger-wiki-resource-usage` (`ResourceName`, `Resource.Load`), `reforger-wiki-prefab-data`
- Index file: `<SteamLibrary>\steamapps\common\Arma Reforger\addons\data\resourceDatabase.rdb`
