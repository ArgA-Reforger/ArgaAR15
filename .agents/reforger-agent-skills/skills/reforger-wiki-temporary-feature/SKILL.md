---
name: reforger-wiki-temporary-feature
description: "Trigger: temporary feature, feature flag, SCR_TemporaryFeature, temporary override. Pattern for safely shipping incomplete or experimental features behind a flag in Enforce Script."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.2.0"
  triggers:
    - "temporary feature"
    - "feature flag"
    - "SCR_TemporaryFeature"
    - "temporary override"
---

## Activation Contract

Load this skill when implementing, reviewing, or gating incomplete or experimental Enforce Script features using the temporary feature pattern. Applies to `SCR_TemporaryFeature` usage and any similar flag-based conditional activation pattern. Do not use for permanent configuration (`[Attribute]` properties) — those belong in `reforger-wiki-config-object`.

## Hard Rules

**CONFIRMED NOT TO EXIST — `SCR_TemporaryFeature`/`ETemporaryFeature` are not real**
A filename/content search across the entire `Game`/`GameCode` source tree (Reforger_1.6.0.119 snapshot) found zero matches for `SCR_TemporaryFeature` or `ETemporaryFeature`. A follow-up search on `arexplorer.zeroy.com` (which indexes `Game`/`GameCode` at the current 1.7.0.54 version and successfully found other classes this session) also found nothing under those names — independently confirmed by the user as well. Everything below describing the `SCR_TemporaryFeature`/`ETemporaryFeature` API is **fabricated and should not be used**. The general "feature flag gating experimental code" idea may still be a valid pattern in this codebase, but under a different (currently unknown) real class/enum name — do not reference `SCR_TemporaryFeature` in generated code; find and substitute the actual mechanism first.

**Purpose**
- The temporary feature pattern is for shipping work-in-progress code to QA or internal builds WITHOUT enabling it by default in release.
- It is NOT a permanent architectural pattern — remove the flag and clean up once the feature graduates to stable.
- Every temporary feature MUST have a removal plan (ticket / milestone noted in a comment).

**`SCR_TemporaryFeature` API**
- Wrap the experimental code block in a check: `if (SCR_TemporaryFeature.IsEnabled(ETemporaryFeature.MY_FEATURE))`.
- The feature identifier is a value in the `ETemporaryFeature` enum — add your feature there.
- Do NOT hard-code feature state in Workbench attributes; the flag is controlled at runtime/build level.
- Features are OFF by default. Enabling is explicit — never assume enabled.

**Adding a new temporary feature**
1. Add an entry to `ETemporaryFeature` (in the appropriate script file, not in your own addon).
2. Wrap experimental code with the `IsEnabled` check.
3. Add a comment with the tracking ticket/milestone for removal.
4. Do NOT ship permanent code that depends on a temporary feature flag staying enabled.

**Temporary override pattern**
- When a temporary feature requires overriding vanilla class behaviour, use `modded class` with an inner `IsEnabled` guard — do NOT `modded` unconditionally.
- If the feature is disabled, the modded method MUST call `super` so vanilla behaviour is preserved.

**Cleanup discipline**
- Once the feature is stable: remove the `ETemporaryFeature` entry, remove all `IsEnabled` checks, and delete any now-unconditional wrappers.
- Leaving dead feature flags in production code is a maintenance debt — treat it as a bug.

**Preprocessor alternative**
- For compile-time gating (not runtime), use `#ifdef DEVELOPER` instead of `SCR_TemporaryFeature`.
- Use `#ifdef` for debug-only helpers; use `SCR_TemporaryFeature` for QA-controlled runtime toggles.

## Key APIs / Patterns

```enforce
// ETemporaryFeature enum — add your entry here (in appropriate game script, not addon)
enum ETemporaryFeature
{
    NONE = 0,
    ARGA_MY_EXPERIMENTAL_FEATURE,   // TODO: remove after Sprint 12 — ticket ARGA-1234
}

// Usage — guard experimental code at runtime
void DoSomething(IEntity entity)
{
    if (!SCR_TemporaryFeature.IsEnabled(ETemporaryFeature.ARGA_MY_EXPERIMENTAL_FEATURE))
        return;  // feature disabled — early return

    // Experimental implementation here
}

// Temporary override in a modded class — preserve vanilla when disabled
modded class SCR_BaseGameMode
{
    override void OnPlayerSpawned(int playerId, IEntity controlledEntity)
    {
        super.OnPlayerSpawned(playerId, controlledEntity);  // ALWAYS call super first

        if (!SCR_TemporaryFeature.IsEnabled(ETemporaryFeature.ARGA_MY_EXPERIMENTAL_FEATURE))
            return;  // feature off — vanilla behaviour preserved via super

        // Experimental additional logic
        DoExperimentalSetup(playerId, controlledEntity);
    }
}

// Static helper — never instantiate SCR_TemporaryFeature
// bool enabled = SCR_TemporaryFeature.IsEnabled(ETemporaryFeature.MY_FEATURE);
```

## References

- PDF: `Scripting Temporary Feature – Arma Reforger - Bohemia Interactive Community.pdf`
- Confirmed absent: full-tree search of Reforger_1.6.0.119 `Game`/`GameCode` source, and `arexplorer.zeroy.com` (see reference memory `reforger/arexplorer-online-doxygen`) — `SCR_TemporaryFeature`/`ETemporaryFeature` not located in either; user independently confirmed the same conclusion.
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting_Temporary_Feature`
- Related spokes: `reforger-wiki-scripting-modding` (modded keyword), `reforger-wiki-preprocessor-directives` (#ifdef DEVELOPER)
