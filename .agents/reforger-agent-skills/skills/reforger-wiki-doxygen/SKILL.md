---
name: reforger-wiki-doxygen
description: "Trigger: doxygen comment, @param, @return, @code, API documentation. Doxygen comment syntax and conventions for Enforce Script API documentation."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.0.0"
  triggers:
    - "doxygen comment"
    - "@param"
    - "@return"
    - "@code"
    - "API documentation"
---

## Activation Contract

Load this skill when writing or reviewing Doxygen-style API documentation comments in Enforce Script `.c` files. Applies to class-level, method-level, and property-level documentation. Do not activate for inline code comments (non-API); those follow `reforger-wiki-conventions`.

## Hard Rules

**Comment syntax**
- Doxygen blocks use `/*!` or `/**` as the opening delimiter and `*/` to close. Both are valid.
- Single-line Doxygen: `//!< brief description` (trailing — for member variable inline doc).
- Enforce Script Doxygen follows standard Doxygen syntax — no custom BI extensions.

**Comment placement**
- Class documentation: immediately BEFORE the `class` declaration.
- Method documentation: immediately BEFORE the method signature.
- Member variable documentation: trailing `//!<` on the same line, OR a block above the variable.
- Do NOT place Doxygen comments inside method bodies — they are ignored there.

**Mandatory tags for public API**
- `@brief` — one-sentence summary (or use the first sentence if no explicit `@brief`).
- `@param [in|out|in,out] name` — one entry per parameter; direction annotation is recommended.
- `@return` — describe the return value; omit for `void`.
- For `out` / `inout` parameters use `@param[out]` or `@param[in,out]`.

**Optional but encouraged tags**
- `@code` / `@endcode` — embed usage example. Wrap Enforce Script examples in these tags.
- `@note` — important side-effect or constraint.
- `@warning` — dangerous usage or common pitfall.
- `@see` — cross-reference to related class or method.
- `@deprecated` — mark obsolete API; include the replacement.

**What to document**
- All `proto`, `proto external`, `proto native` declarations (these form the engine API surface).
- All `public` and `protected` methods in game classes.
- All `[Attribute]` member variables (why the field exists, valid range if not in UIWidgets).
- Do NOT document `private` implementation details unless they are non-obvious.

**Formatting**
- Keep `@brief` to one line (~80 chars max).
- Long descriptions go in subsequent paragraphs after a blank Doxygen line.
- Consistency: use `@param` (not `\param`) — Enforce Script convention follows `@` prefix.
- Align `@param` entries when there are multiple parameters.

## Key APIs / Patterns

```enforce
/*!
 * @brief Manages the score for a single player during a treasure hunt session.
 *
 * Tracks collected items, total points, and notifies listeners when the score changes.
 * Attach to the player entity or game mode component.
 *
 * @note Score changes are replicated to all clients via RplProp.
 * @see SCR_TW_GameModeComponent
 */
class ARGA_ScoreManager : ScriptComponent
{
    [Attribute("0", UIWidgets.EditBox, "Starting score for this player")]
    protected int m_iStartScore; //!< Initial score value configured in Workbench.

    /*!
     * @brief Adds points to the player's total score.
     *
     * Must be called on the authority (server). Automatically triggers replication.
     *
     * @param[in]  points      Number of points to add. Must be >= 0.
     * @param[in]  instigator  Entity that caused the point gain (may be null for system awards).
     * @return                 New total score after the addition.
     * @warning Do NOT call from a proxy — only the authority should modify score.
     */
    int AddPoints(int points, IEntity instigator)
    {
        m_iScore += points;
        Replication.BumpMe();
        return m_iScore;
    }

    /*!
     * @brief Returns the player's current total score.
     * @return Current accumulated score value.
     */
    int GetScore() { return m_iScore; }

    /*!
     * @brief Resets score to the configured start value.
     *
     * @code
     * ARGA_ScoreManager sm = ARGA_ScoreManager.Cast(entity.FindComponent(ARGA_ScoreManager));
     * if (sm)
     *     sm.ResetScore();
     * @endcode
     *
     * @deprecated Use AddPoints with a negative delta instead. Will be removed in v2.0.
     */
    void ResetScore()
    {
        m_iScore = m_iStartScore;
        Replication.BumpMe();
    }

    [RplProp(onRplName: "OnScoreReplicated")]
    protected int m_iScore; //!< Current score, replicated to all clients.
}
```

**Minimal required block for any public method**
```enforce
/*!
 * @brief [One sentence what it does.]
 * @param[in] paramName [What it is and any constraints.]
 * @return [What is returned, or omit if void.]
 */
```

## References

- PDF: `Doxygen - Bohemia Interactive Community.pdf`
- External: `https://www.doxygen.nl/manual/docblocks.html`
- Wiki: `https://community.bistudio.com/wiki/Doxygen`
- Related spokes: `reforger-wiki-conventions` (code style), `reforger-wiki-best-practices`
