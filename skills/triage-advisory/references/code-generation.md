# Code generation and escaping

Reports in this family claim a value is interpolated into generated source (JS/TS emitted at runtime or by a CLI, templates, config files) without proper escaping.

## Reading the escaping helper

Walk every branch of the helper against every metacharacter of the *target* grammar, not just the one the payload used:

- string delimiters for each quote style the generator emits (`'`, `"`, backtick)
- the backslash itself (escaping the quote but not the backslash re-opens the string)
- template interpolation (`${` inside backtick strings)
- line terminators, including `\u2028` / `\u2029` (legal in string literals since ES2019, but they still end a `//` comment and count for ASI)
- comment terminators (`*/`) when values land inside comments
- identifier rules when values become property, class, or variable names: an identifier cannot be "escaped", it has to be validated or quoted as a computed key

Then find every call site that interpolates the same kind of value and check that it calls the helper at all. A bypassing call site is the common worst case.

## Telling severity

Ask who supplies the interpolated value (gate 2 in SKILL.md). Names and definitions the developer writes are developer-trust input: injection through them is a correctness bug in the generator, not a vulnerability. The falsifier: the same value usually also becomes an identifier or a file name somewhere else, which no application would ever take from end users.

## Fix and tests

Fix as escaping correctness: the generated output must parse for every input, including inputs that contain each metacharacter above. Add a test that generates code from such values and actually compiles or evaluates the output; a snapshot alone just freezes whatever the generator produced.
