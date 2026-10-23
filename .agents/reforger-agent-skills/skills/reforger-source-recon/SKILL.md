---
name: reforger-source-recon
description: "Trigger: vanilla class, vanilla component, vanilla prefab, vanilla config, does vanilla already, enfusion_tools, .pak, data007, archive search, find in game files. Method for reading real vanilla Arma Reforger scripts and assets from packed archives instead of answering from memory."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "vanilla class"
    - "vanilla component"
    - "vanilla prefab"
    - "vanilla config"
    - "does vanilla already"
    - "enfusion_tools"
    - ".pak"
    - "data007"
    - "archive search"
---

## Activation Contract

Load this skill when a question is about how vanilla Arma Reforger classes, components, prefabs, configs, particles or sounds look or behave, or when you must check whether vanilla already implements something. Do not answer such questions from memory: read the real file and cite it. Does NOT cover writing Enforce Script (see the other spokes). Resolving `{GUID}` values to paths is covered by `reforger-guid-resolution`.

## Hard Rules

**Source priority**
- If an extracted script tree for the SAME game build already exists locally, read it first with normal grep/read tools. Confirm its build number against the installed game before relying on it.
- Otherwise read the packed archives in place with the `enfusion_tools` CLI. It reads PAK archives by index (list, glob, grep, cat, extract) without unpacking.
- Doxygen gives signatures and inheritance, never method bodies. Online mirrors may be a different build than the installed one. Neither replaces reading the file.

**Never brute-force the install directory**
- NEVER run recursive `grep`, `rg` or `find` over `addons\`. It is ~26 GB of binary archives, hangs or times out, and cannot read archive contents anyway.
- Run the CLI from PowerShell on Windows. Under Git Bash, paths containing spaces (`Arma Reforger`) failed.
- If an expected tool fails or returns empty, report what was tried and stop; do not improvise a costly scan.

**Pick the right archive** (`<SteamLibrary>\steamapps\common\Arma Reforger\addons\data\`, game build 1.8.0.13)
- `data007.pak` holds all vanilla scripts (5809 `.c`), plus Configs, Prefabs, PrefabsEditable, PrefabLibrary, Particles, Missions, Language. For code questions it is the only archive needed.
- Sounds: `data008.pak`, `data009.pak` (weapon audio). Terrains/UI/WBData: `data010.pak`. `data.pak`, `data001`-`data006`: mostly Assets (`data.pak` also AI, anims). Worlds: `worlds*.pak`.
- Script tree top-level folders verified in `data007.pak`: `Autotest`, `Game`, `GameCode`, `GameLib`, `Workbench`, `WorkbenchGame`. There is no `Core` folder in this archive.
- The install has TWO `resourceDatabase.rdb` files (`addons\core\` and `addons\data\`). The one under `data\` is almost always the one wanted.

**Verification discipline**
- Cite `file:line` for every claim about vanilla behaviour. If you cannot cite it, you did not verify it.
- Never state inheritance, an access modifier or the inclusivity of a numeric range from memory. Grep the declaration, read the parent, grep the parent, repeat until you reach an engine type.
- Read the body, not the signature. Behaviour lives in the implementation.
- Absence in one archive is not absence. Say which archives were searched.
- Report mechanism (what the code does, with citation) separately from opinion (label it as opinion).
- State the game build the answer was verified against.

**Config inheritance**
- Enfusion configs chain to an ancestor through a `parent` field. A value can be missing from the file you read because it is defined higher up (e.g. a weapon `.acp` often lacks its shot event; it comes from a shared base config).
- Finding the file is not finding the value: follow `parent` to the root before concluding a field is missing. Applies to every config type.

**Reading the result**
- Read the distribution of `glob` hits, not just the count. Several files clustered under one subsystem (behavior, activity, waypoint, volume, helper nodes) signal a complete vanilla system to build on; two scattered unrelated hits do not.

## Key APIs / Patterns

Setup (PowerShell). `enfusion.exe` is the CLI from `https://github.com/landaire/enfusion_tools` (MIT, Rust). It requires only the installed game, not the modding Tools. Get it from prebuilt binaries in the repository's Releases tab, or build it with `cargo build --release`:

```powershell
$exe = "<path-to>\enfusion.exe"
$pak = "<SteamLibrary>\steamapps\common\Arma Reforger\addons\data\data007.pak"
$dir = "<SteamLibrary>\steamapps\common\Arma Reforger\addons\data"
```

Commands: `list` (`-f` flat, `-g` glob, `-l` sizes), `glob` (`-l`), `grep` (`-i`, `-g`, `-e <ext>`, `-l` files only, `-C <n>` context), `cat` (`-p/--path` is REQUIRED, not positional), `info`, `extract` (`-o`, `-f`). Every command takes one or more `.pak` files, or a directory containing them, as the LAST argument.

```powershell
# 1. Concept -> candidate files (cheapest query; "does vanilla already do this?")
& $exe glob "**/*Suppress*.c" $pak

# 2. Class declaration with file, line and parent class
& $exe grep "class SCR_AISuppressBehavior" $pak
#   /scripts/Game/AI/Behavior/SCR_AISuppressBehavior.c:
#   >     5: class SCR_AISuppressBehavior : SCR_AIBehaviorBase

# 3. Read the real source (copy the path verbatim from glob output, leading slash included)
& $exe cat --path "/scripts/Game/AI/Behavior/SCR_AISuppressBehavior.c" $pak

# 4. Search the whole codebase (narrow to scripts with -e c)
& $exe grep -e c "class \w+ : SCR_AIBehaviorBase" $pak     # everything deriving from a base
& $exe grep -l -e c "GetSuppressionVolume" $pak            # who references it: file names only
& $exe grep -C 3 -e c "RplComponent" $pak                  # with surrounding lines

# 5. Assets (prefabs .et, configs .conf, sounds .acp, particles .ptc): aim at the right archive
& $exe glob "**/*PKM*.acp" "$dir\data008.pak" "$dir\data009.pak"

# 6. Sustained work on one subsystem: extract once, then use ordinary local tools
& $exe extract "scripts/Game/AI/**" $pak -o .\vanilla-ai
```

Re-tested on a current build: recursive `**/` globs DO work (`**/*.c` matched all 5809 script files) and passing a directory DOES work (scans every archive in it; slower than targeting one). `list` prints an indented tree unless `--flat` is passed.

Worked example (3 commands): "does vanilla implement AI suppressive fire?" -> `glob "**/*Suppress*.c"` (clustered AI behavior/activity/waypoint/volume files = a complete system) -> `grep "class SCR_AISuppress"` (declarations with parents) -> `cat` the two or three that matter. The design question becomes "how do we drive the suppression that exists", not "how do we build it".

**Optional: modding Tools installed**
- Supplement only. Offline Doxygen (`Workbench\docs\`) gives inheritance diagrams and full member lists but signatures only. The Enfusion guides add prose the archives lack (EnforceScript syntax, reference counting, event system, replication pages).

## References

- `enfusion_tools` CLI: `https://github.com/landaire/enfusion_tools`
- Related spokes: `reforger-guid-resolution` (GUID <-> path), `reforger-wiki-doxygen` (documenting API, not reading it), `reforger-wiki-config-class`, `reforger-wiki-resource-usage`, `reforger-wiki-multiplayer`
- Verification note: `data007.pak` script count (5809), folder breakdown, the `**/*Suppress*.c` glob (15 hits) and the `SCR_AISuppressBehavior` declaration at line 5 were confirmed against build 1.8.0.13.
