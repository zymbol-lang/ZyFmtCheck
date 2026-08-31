# ZyFmtCheck

*Named for the command it guards: `zymbol fmt`, so the project, its runner
`zyfmtcheck` and the subcommand all spell the same three letters.*

**The formatter moves things. It does not change them. This proves it, or names
what it changed.**

`zymbol fmt` is a layout tool. It may alter whitespace, indentation and where a
line breaks. It may not alter the code, the comments, or the spelling of a
literal. When a program is wrong, the formatter is not the thing that fixes it —
the **LSP reports it and the programmer decides**. A formatter that repairs,
tidies, normalises or improves is a formatter that edits your program without
being asked, and no amount of good intent makes that acceptable.

This project exists because that rule was written down and broken anyway.

## Why it is a project and not a test

`interpreter/FORMATTER_RULES.md` states the contract in prose, and the formatter
carries its own safety gate. Neither was enough:

- **The gate compares tokens**, and a `TokenKind` carries the *value*, not the
  spelling. `४२` and `42` are one `Integer(42)`. Printing `42` for `४२` passed
  the gate **for four releases** — a program written in Devanagari stopped being
  one the moment it was formatted, and nothing anywhere went red.
- **P1–P4** in `zyquality/fmt/` ask whether the output reparses, is idempotent,
  runs the same, and keeps its comment count. All four pass on that rewrite. The
  output reparses, is stable, runs identically and keeps every comment. It is
  simply not the program the author wrote.
- **The prose is prose.** `FORMATTER_RULES.md` § 10 lists the allowed
  normalizations and nothing executes it, so the list and the behaviour drift
  apart in silence — which is how § 10 came to describe a formatter that had
  moved on.

So the list lives here, in [`normalizations.toml`](normalizations.toml), and it
is read by the runner and by nothing else. **Adding a normalization means adding
an entry with a reason, in a file a reviewer opens.** A formatter cannot grow a
new power quietly. That is the whole design.

## The pins — five findings, five minimal files

`cases/pin/` holds one file per named finding, and the assertion is the sharpest
a formatter allows: **each is already formatted, so `zymbol fmt` on it must be a
no-op — byte for byte.** No goldens, no tolerance, nothing to keep in step. If a
behaviour comes back, the file changes or the gate refuses it, and either is red.

They exist because a body cannot name anything. Running over GO answers
"something changed in GO"; a pin answers "the interpolated-string arm is gone",
which is the difference between a report and a diagnosis.

**Every one was verified by reverting its fix**, and that mattered: two of the
first drafts held while the fix was gone.

| pin | holds |
|---|---|
| `FMT-001` | an interpolated string reprints as written |
| `FMT_002` | the export block stays where the author put it |
| `FMT-004` | a bare non-ASCII key stays bare |
| `FMT-005` | a quoted key stays quoted |
| `FMT-006` | an output statement is never wrapped |

Three things they taught, written in their own headers so the next editor does
not repeat them:

- **`FMT-001` must use interpolated strings.** The first draft used plain ones,
  which take the `String` arm — never broken — so it passed with the fix
  reverted.
- **`FMT-006` needs a long ARGUMENT LIST, not a long line.** The wrap decision
  measures the arguments; a 116-character line with a short call wraps nothing.
- **There is no `FMT-003`.** It held the Rust-`{:?}` escaping, and once
  `NamedTupleExpr` records `quoted` and the identifier rule is the lexer's, that
  branch is unreachable from parsed input — reverting the escaping alone changes
  nothing, measured. The fix stays as defence in depth; the pin was retired,
  because a pin that cannot fail is not a pin. `FMT-004`'s header says so.

`FMT_002` spells its ID with underscores: a module's name must be an identifier
and must match its file stem, so that one pin cannot spell its own ID the way
the rest do.

## What it does

```bash
./bin/zyfmtcheck                 # the LDV applications
./bin/zyfmtcheck --body corpus   # the corpus instead
./bin/zyfmtcheck --only go       # one application
./bin/zyfmtcheck -v              # show the layout categories too
```

Pins first, then three phases over each body:

1. **Format** — copy the tree to a *temporary directory* and format every `.zy`
   there. Never in place: the formatter writes, and a validator that can damage
   what it validates is worse than none.
2. **Verify** — the code skeleton and the multiset of comments must be
   unchanged, and every remaining difference must be one `normalizations.toml`
   declares. An unexplained one names the file and quotes both sides.
3. **Run** — execute the application's own suite **from the copy** and require
   the same output. Everything above is a proxy for this question, and a
   skeleton comparison can be fooled by a reordering it cannot see. Running
   cannot.

Exit codes, the contract every runner in this workspace honours: **0** green,
**1** red, **2** no verdict.

## Why the LDV applications

Because they are the richest Zymbol there is — real nesting, hand-aligned
tables, comments in awkward places, five writing systems, modules importing each
other. A formatter's damage lives in exactly what a short file does not have,
and the corpus is 661 files that are mostly a few lines long. `--body corpus`
asks the wider, shallower question; it is worth running and it is not the same
question.

## What it found on its first run

Two behaviours, both invisible to the gate and to P1–P4, both now fixed:

| | |
|---|---|
| **Interpolated strings were re-spelled** | a newline written inside `"…{x}…"` came back as `\n`, and `\'` came back as `'`. `literal_source_form` had an arm for `String` and none for `InterpolatedString`. Same family as `४२` → `42`. |
| **The export block was moved** | a module opening with `#> { … }` and imports underneath came back with the two swapped. |
| **A dictionary key was re-spelled, twice over** | the quoted form went through Rust's `{:?}`, writing a combining mark as `\u{941}` — syntax Zymbol does not have, which its own lexer then read as a broken interpolation. And the bare/quoted decision used `is_alphanumeric()`, narrower than the lexer's rule, so `मुद्रा` was judged to need quoting. |
| **A quoted key was un-quoted** | a key arrives as a bare `String` either way, so the formatter had to guess and printed every key that *could* be bare bare — `String("k")` became `Ident("k")`. Fixed by recording `quoted` on the node, the remedy §12 names. |
| **An output statement was wrapped** | `>>` and `>>~` end at the line, so breaking a long call inside one ends the statement and leaves a fragment. |

All five are fixed and all five have a pin. The common shape is worth naming:
each was the formatter **deciding** how something ought to be written — quotes,
escapes, order, line length — instead of reproducing what was there.

## What is still open

Nothing in the applications: **refusals went from fourteen to zero**, and every
suite is byte-identical after formatting. The corpus's thirteen refusals are
files that do not parse — error fixtures — which is the correct refusal.

## Its place among the other layers

| | asks |
|---|---|
| **LDV** (`interpreter/LDV.md`) | can the language express this domain? |
| **ZyQuality** (`zyquality/`) | do the engines agree, and match their goldens? |
| **ZyDDT** (`ZyDDT/`) | do all engines still answer the same, correctly? |
| **ZyFmtCheck** (here) | did the formatter change anything it was not allowed to? |

It borrows ZyQuality's bodies — the corpus and `project/apps.toml` — and owns
nothing but the question. `ZYQUALITY_ROOT` and `ZYMBOL_BIN` point it elsewhere.

## Authorship & AI collaboration

Zymbol is designed by
**[OscarE.EspinozaB](https://github.com/zymbol-lang/interpreter/commits?author=OscarEEspinozaB)**.
This project is built with **[Claude Code](https://claude.ai/code)** (Anthropic)
as the engineering team, under the author's direction.

It is worth saying plainly what this one is for. The formatter kept acquiring
transformations — conversions, tidying, reordering — that nobody asked for and
that the existing checks could not see. Several were written by an assistant
that judged them improvements. They were not improvements: they were edits to
somebody else's program, made by a tool whose entire remit is whitespace.

This project makes that class of change impossible to land silently. The list of
what the formatter may do is small, declared, and requires a human edit to grow.
