![version](https://img.shields.io/badge/version-20%2B-E23089)
![platform](https://img.shields.io/static/v1?label=platform&message=mac-intel%20|%20mac-arm%20|%20win-64&color=blue) [![license](https://img.shields.io/github/license/miyako/4d-syntax-vscode)](LICENSE)
![downloads](https://img.shields.io/github/downloads/miyako/4d-syntax-vscode/total)

# 4d-syntax-vscode
A Visual Studio Code syntax extension for 4D.

based on [4D.tmbundle](https://github.com/miyako/4D.tmbundle)

normally you should use the LSP aware [4D-Analyzer](https://marketplace.visualstudio.com/items?itemName=4D.4d-analyzer).

# 4D Syntax Highlighting — Implementation Specification

Two TextMate-grammar-based syntax highlighters for the 4D language (`.4dm` / `.c4d`
files), targeting two different hosts:

| | Host | Format | Scope name |
|---|---|---|---|
| **A** | TextMate 2 (macOS editor, bundle system) | `.tmLanguage` (XML plist) | `source.4dm` |
| **B** | VS Code extension | `.tmLanguage.json` (JSON) + `package.json` + `language-configuration.json` | `source.4dm` |

Both are hand-derived ports of a separate, authoritative **tree-sitter grammar**
for 4D (`grammar.js` + `scanner.c` + `builtins.h` + `highlights.scm`, targeting
4D 21 R4), used as the source of truth for what the language actually accepts.
Neither grammar is a parser: both are flat, position-independent regex
(oniguruma) pattern lists with no concept of a parse tree, so every design
decision below exists to work around that constraint while staying faithful to
what the tree-sitter grammar treats as a single token vs. several.

Both files currently define **87 patterns** and share an identical scope
taxonomy (below) — B was built directly from A's design, adapted only where the
JSON host format or its pre-existing structure required it.

---

## 1. Core design principle: one token, one scope

The tree-sitter scanner treats several constructs as **single atomic tokens**
(e.g. `New collection:C1472` is one `command` leaf; `[Employees:1]Name:2` is one
`field_reference` leaf). A parser hands the whole node to the highlighter at
once, so it can never be styled inconsistently.

A flat regex grammar has no such guarantee: if a pattern only *peeks* at part of
a token (via a zero-width lookahead) instead of *consuming* it, the un-consumed
remainder is left exposed to whatever unrelated pattern happens to match it
next — producing a token that renders in two or three different colors instead
of one. Both grammars are written to avoid this:

- Classic command/constant names (`Name:C1234`, `Name:K12:3`) consume the
  `:C`/`:K` suffix as part of the same match, rather than using
  `(?=(:C[0-9]+))` to only color the name and leave the suffix to fall through
  to the generic operator/number rules.
- `field_reference` (`[Table:1]Field:2`) is matched and scoped as one span,
  not split into a bracket-match plus a separate field-name-match.
- Where two different real distinctions exist at the same lexical position
  (an ORDA member access is either a method **call** or a **property**
  access), the two patterns are ordered so the more specific one (call,
  requiring a following `(`) is tried first, and the second only fires when
  the first didn't match — rather than both being written identically and one
  silently shadowing the other as dead code.

## 2. Scope taxonomy

Scopes are deliberately kept **shallow** (2 segments deep, e.g.
`support.function.4d`, not `entity.name.function.classic.4d`) and mapped onto
scope families that virtually every existing TextMate/VS Code theme already
assigns a distinct color to. TextMate scope matching resolves by specificity:
if a theme has no rule for a deep custom leaf, it falls back to the nearest
matching ancestor — so two different deep scopes sharing an undecorated
ancestor (e.g. both falling back to `entity.name.function`) become visually
identical under any theme that doesn't specifically know about them. Shallow,
standard scopes avoid this collapse.

| Scope | Used for |
|---|---|
| `comment.block.4d` / `comment.line.double-slash.4d` | `/* */` and `//` comments |
| `meta.attributes.4d` | `// %attributes = {...}` header |
| `meta.embedded.block.sql.4d` | `Begin SQL … End SQL` body (raw, unparsed) |
| `keyword.other.sql-block.4d` | the `Begin/End SQL` delimiters themselves |
| `constant.numeric.4d` (+ `.decimal`, `.exponential`, `.hexadecimal`) | number literals |
| `constant.other.time.4d` / `constant.other.date.4d` | `?HH:MM:SS?` / `!date!` literals |
| `support.constant.4d` | tokenized built-in constants (`Is text:K...`) |
| `string.quoted.double.4d` | string literal contents |
| `constant.character.escape.line-continuation.4d` | trailing `\` line-continuation |
| `punctuation.definition.char-reference.4d` | `[[` `]]` char-reference brackets |
| `punctuation.definition.variable.indirection.4d` | `${` parameter indirection |
| `variable.parameter.numeric.4d` | `$1`, `$2`, … |
| `variable.other.local.4d` | `$name` local variables |
| `variable.other.interprocess.4d` | `<>name` interprocess variables |
| `variable.other.property.4d` | ORDA property access (`.name`) **and** DB field references (`[T:1]Field:2`) — unified: both are "named member of a record" |
| `support.class.4d` | bare table reference (`[Table:1]`, no field) |
| `entity.name.function.4d` | ORDA method calls (`.push(`, `.join(`) and declared function names |
| `support.function.4d` (+ `.deprecated`) | tokenized built-in commands (`Name:C1234`), including the `_o_`-prefixed deprecated form |
| `support.type.4d` | `c_ClassName:C...` classic types, and `: Type` annotations |
| `variable.language.4d` | `this` / `super` / `form` / `null` / `4D` / `ds` / `cs` |
| `storage.modifier.4d` | `shared session singleton exposed local server onHTTPGet` |
| `storage.type.4d` | `var`, `property` |
| `keyword.control.4d` | `If/Else/While/Repeat/Until/For/Try/Catch`, English + French forms, `return/break/continue/throw/defer` |
| `keyword.control.directive.4d` | `#DECLARE` |
| `keyword.other.4d` | `Use`, `Alias`, `Class extends`, `Function`/`Class constructor` keyword itself, English/French `End …` closers not otherwise categorized |
| `support.variable.classic.4d` | reserved system variables (`OK`, `Error`, `Document`, `MouseDown`, …) |
| `keyword.operator.4d` / `keyword.operator.variadic.4d` | full operator set (see §4) + `...` |

## 3. Literal-syntax fixes (matching `scanner.c`, not the language docs)

- **Time literal**: hours field is `{2,3}` digits, not a fixed 2 — 4D's scanner
  explicitly allows hours to exceed 24.
- **Date literal**: single unified pattern using a backreference so the same
  delimiter (`-`, `/`, or `.`) is required on both sides —
  `!([0-9]{1,4})([-/.])([0-9]{1,4})\2([0-9]{1,4})!` — replacing two rigid
  `YYYY-MM-DD` / `DD-MM-YYYY`-only patterns that couldn't represent the
  regional `/`/`.` delimiter forms 4D actually accepts.
- Leading-character classes for tokenized command/constant names were widened
  to permit a leading digit or underscore (`[\p{Letter}_0-9]`), matching the
  tree-sitter `command`/`constant` token's actual character class (methods
  named `00_Start`, the tokenized `4D:C1709` namespace, etc.).

## 4. Operator set

Both grammars use one alternation for the full operator set, ordered so
multi-character operators are tried before their single-character prefixes
(`:=` before `:`, `->` before `-`, `??`/`?+`/`?-` before bare `?`, etc.):

```
:=  +=  -=  *=  /=   ||  &&  |  ^|  &   =  #   <  >  <=  >=
<<  >>  ??  ?+  ?-   ->   +  -  *  /   %  \  ^  ?  :
```

plus a separate `...` (variadic parameter marker) pattern. An earlier draft
carried a stray empty alternation branch (`\|\|||\|`, a typo) and was missing
`->` and `^|` entirely — both fixed.

## 5. Additions not present in the pre-existing grammars

Cross-referenced against `scanner.c` / `grammar.js`, the following were added
because they were real, missing syntax rather than cosmetic gaps:

- `throw` / `defer` statement keywords
- `${...}` parameter indirection
- `[[` / `]]` char-reference brackets
- Database field/table references (`[Table:1]Field:2`, bare `[Table:1]`)
- `...` variadic parameter ellipsis
- `->` pointer-creation/dereference and `^|` bitwise-OR-variant operators
- All 7 function/class-constructor modifiers (`shared session singleton
  exposed local server onHTTPGet`) — earlier versions recognized only 2–3
- `Function` computed-attribute accessors (`get set event query orderBy`)
- `// %attributes = {...}` header as its own scope
- Trailing-backslash line continuation as its own scope, distinct from the
  `\` integer-division operator
- `Begin SQL … End SQL` as a proper raw region (see §6) instead of four flat
  keyword matches

## 6. `Begin SQL … End SQL` handling

Implemented as a `begin`/`end` block (not a flat keyword match), body left
unparsed (empty `patterns: []`), matching `scanner.c`'s `scan_sql`: the
terminator must be `End SQL` / `Fin SQL` **alone on its line** (`^\s*(end
sql|fin sql)\s*$`), so SQL text that merely contains the words "End SQL"
inside a string or comment can't falsely close the block.

## 7. Known limitations (not addressed)

- **Builtin command/constant names are not enumerated.** The tree-sitter
  scanner's builtin table (`builtins.h`) covers ~1300 commands and several
  thousand constants; both grammars rely entirely on the `:C`/`:K` tokenized
  suffix to recognize these, which only works for *tokenized* source. This is
  the same strategy the pre-existing grammars used and is a deliberate,
  correct tradeoff — enumerating the table as literal regex alternatives isn't
  practical in a TextMate grammar.
- **No rich structural code folding.** The reference tree-sitter grammar's
  `folds.scm` defines per-construct folding (`if`/`while`/`for`/`try`/function
  bodies each fold independently). TextMate/VS Code's single
  `foldingStartMarker`/`foldingStopMarker` pair (or VS Code's simple
  `folding.markers`) can only express one block-comment-style region; matching
  `folds.scm` would require restructuring the entire pattern list into nested
  `begin`/`end` blocks, which is a substantially larger undertaking than a
  regex-level update.
- **French keyword forms are retained** (`si`, `sinon`, `tant que`, `boucle`,
  `pour chaque`, …) even though the current tree-sitter grammar (`grammar.js`)
  only encodes English-only literal tokens for these statements. Kept for
  compatibility with older 4D source, per the intended use case of these
  grammars.
- **Theme dependency.** Scope names are chosen to be as broadly compatible as
  possible (§2), but any theme is still free to not style a given scope at
  all; visual differentiation between e.g. `support.function.4d` and
  `entity.name.function.4d` depends on the active theme actually defining
  different colors for `support.*` vs `entity.name.*`.

---

## Appendix A — TextMate 2 bundle (`4D.tmLanguage`)

- Format: XML property list (`plist`), root `patterns` array.
- Regex engine: oniguruma, matched per-line except inside explicit
  `begin`/`end` blocks (block comment, line comment, SQL region).
- Line comments (`//`) are implemented as a simple `match`-to-end-of-line rule.
- Folding: `foldingStartMarker` / `foldingStopMarker` cover only `/* */` block
  comments.
- Install path: `~/Library/Application Support/TextMate/Bundles/
  <Bundle>.tmbundle/Syntaxes/4D.tmLanguage`, or via Bundle Editor
  (⌥⌘B) → drag file onto a bundle.
- Theme selection is a separate, per-window concern: **View → Theme** (and
  **View → Theme for Light/Dark Appearance**), not Preferences.

## Appendix B — VS Code extension

Three files, per VS Code's extension manifest conventions:

- **`4D.tmLanguage.json`** — same grammar, JSON-serialized. Differs from the
  TextMate plist in one structural respect carried over from the pre-existing
  file: the line-comment rule is a continuation-aware `begin`/`end` pair
  (`end: (?<=\n)(?<!\\\n)`) rather than a plain match-to-newline, so a `//`
  comment whose line ends in `\` continues onto the next line, mirroring
  `grammar.js`'s `line_comment` token.
- **`package.json`** — extension manifest. `contributes.languages[].id` is
  `fourd` (an internal identifier — must be a valid non-numeric-leading token,
  which `4d` is not), with `aliases: ["4D"]` providing the user-facing display
  name shown in the language picker. `contributes.grammars[].language` /
  `scopeName` must match the language `id` and the grammar's own `scopeName`
  respectively — both do (`fourd` / `source.4dm`). Version bumped to `0.0.3`
  to reflect the grammar update.
- **`language-configuration.json`** — bracket-matching/auto-close rules.
  Already listed `[[`/`]]` as its own bracket pair (ahead of `[`/`]`), which
  aligns with the char-reference token added to the grammar; no changes were
  needed here.

One bug specific to this file's pre-existing form (not present in the
TextMate plist) was also fixed: most patterns used a leading `\s*` where the
plist used `(?<!\w)`. `\s*` is satisfied by zero whitespace, so it doesn't
actually anchor the match to a word boundary — patterns could fire
**mid-identifier** (e.g. `0xFF` lighting up as a hex constant inside a longer
name like `myVar0xFF`). All such patterns were changed to the equivalent
negative-lookbehind form used throughout the TextMate plist.
