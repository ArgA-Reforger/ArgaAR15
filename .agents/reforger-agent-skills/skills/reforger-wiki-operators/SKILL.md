---
name: reforger-wiki-operators
description: "Trigger: &&, ||, <<, >>, %, ==, !=, +=, -=, *=, /=, bitwise, precedence, modulo. Enforce Script operators, precedence, arithmetic, relational, logical, bitwise, and string concatenation."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "&&"
    - "||"
    - "<<"
    - ">>"
    - "%"
    - "=="
    - "!="
    - "+="
    - "-="
    - "*="
    - "/="
    - "bitwise"
    - "precedence"
    - "modulo"
---

## Activation Contract

Load this skill when the context involves Enforce Script operator usage, operator precedence questions, bitwise operations, sign-extension with `>>`, integer division rounding, or string concatenation rules. Do NOT load for class structure or type declaration work.

## Hard Rules

**Precedence**
- Enforce Script operator precedence is identical to C. When in doubt, add parentheses.
- Reference: https://en.wikipedia.org/wiki/Operators_in_C_and_C%2B%2B#Operator_precedence

**Assignment vs. Equality**
- `=` assigns; it is NOT a comparison. Use `==` for equality checks.
- Confusing `=` with `==` inside `if` conditions is a compilation error in Enforce Script (unlike C).

**Arithmetic**
- Integer division truncates toward zero: `5 / 3 = 1`, `-5 / 3 = -1` (NOT -2). This differs from mathematical floor for negative values. Mix one `float` operand to get float result: `5 / 3.0 = 1.666...`
- `%` (modulo) applies to `int` only. For `float` modulo use `Math.Repeat()`.
- Pre-increment (`++i`) returns the incremented value. Post-increment (`i++`) returns the original value before increment.

**Relational**
- `==` on objects compares **references** (same instance), not values. Two `new` instances of the same class are NOT `==`.
- `==` on `vector` compares values (not references).
- `string` comparison is case-sensitive.
- Float equality with `==` is unreliable due to floating-point precision — use `float.AlmostEqual()`.

**Logical**
- `&&` (AND) and `||` (OR) short-circuit: right operand is not evaluated if result is determined by left operand.
- `!` (NOT) also works on `int`/`float` (0 test), `string` (empty string test), and objects (null test) — but explicit checks (`== 0`, `.IsEmpty()`, `== null`) are preferred for clarity.

**Bitwise**
- `&` — bitwise AND, `|` — bitwise OR, `~` — bitwise NOT (inverts all bits).
- `<<` — left shift (multiply by power of 2), `>>` — right shift, `<<=` — left shift-assign, `>>=` — right shift-assign.
- BEWARE of sign extension with `>>` on negative integers: `0x80000000 >> 8 = 0xFF800000` (sign bit fills from left).

**String Concatenation**
- `+` with `string` left operand stringifies the right operand automatically (`"val: " + 42` → `"val: 42"`).
- The left operand MUST be a `string`; `3 + "test"` is a parsing error.
- Chained concatenation: `"a" + 3 + true + false + 4` → `"a3104"` (each appended left-to-right).

**Indexing**
- `array[i]` is equivalent to `.Get(i)` on dynamic arrays — it is a method call with overhead.
- On static arrays `T arr[N]`, `arr[i]` is constant-time with no method call — prefer static arrays in hot paths.
- Cache the element if accessed multiple times inside a loop; do not call `myArray[i]` repeatedly.

## Key APIs / Patterns

```c
// Integer division — floor, not round
int result = 20 / 3; // result is 6, not 6.67

// Float division
float fResult = 20 / 3.0; // 6.666...

// Modulo (int only)
int rem = 10 % 3; // 2

// Bit flags (combine with |, test with &)
int flags = ARGA_EStatusFlags.HEALTHY | ARGA_EStatusFlags.HAS_AMMO;
if (flags & ARGA_EStatusFlags.HEALTHY)
    Print("Healthy");

// Sign-extension guard on right shift
int val = 0x80000000;
int shifted = val >> 8; // 0xFF800000 — NOT 0x00800000!

// String concatenation — left must be string
string msg = "Health: " + 75;      // OK: "Health: 75"
// string err = 75 + " health";    // parse error

// Object reference comparison
ARGA_MyClass a = new ARGA_MyClass();
ARGA_MyClass b = new ARGA_MyClass();
bool sameRef = (a == b); // false — different instances
ARGA_MyClass c = a;
bool sameRef2 = (a == c); // true — same instance

// Float comparison
float x = 0.1 + 0.1 + 0.1;
if (float.AlmostEqual(x, 0.3))
    Print("close enough");

// Array indexing — cache the element
foreach (int i, string item : m_aItems)
{
    // avoid calling m_aItems[i] multiple times
    if (item.IsEmpty())
        continue;
    PrintFormat("%1: %2", i, item.Trim());
}
```

## References

- Doxygen: `float.AlmostEqual(a, b, epsilon=0.0001)` confirmed real (`float_8c_source.html`). No bugs found.
- PDF: `Scripting_ Operators – Arma Reforger - Bohemia Interactive Community.pdf`
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:Scripting:_Operators`
