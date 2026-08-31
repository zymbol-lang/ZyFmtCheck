# ZyFormatCheck

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

## What it does

```bash
./bin/zyfmtcheck                 # the LDV applications
./bin/zyfmtcheck --body corpus   # the corpus instead
./bin/zyfmtcheck --only go       # one application
./bin/zyfmtcheck -v              # show the layout categories too
```

Three phases:

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
| **Interpolated strings were re-spelled** | a newline written inside `"…{x}…"` came back as `\n`, and `\'` came back as `'`. `literal_source_form` had an arm for `String` and none for `InterpolatedString`, so every interpolated literal fell through to the re-writer. Same family as `४२` → `42`. |
| **The export block was moved** | a module that opens with `#> { … }` and imports underneath came back with the two swapped. The gate caught it and refused the file — the right failure, and still a failure: **fourteen** application files could not be formatted at all, including every `api/` re-export layer in GO and Chaturanga. |

After both: the seven applications format clean and their suites are unchanged,
and refusals went from fourteen to two.

## What is still open

Two files the gate refuses, both string handling in the formatter:

- `Chaturanga/दर्शनम्/चित्रणम्.zy` — `formatted output no longer parses:
  unexpected token: String("│"`
- `ZyBank/configuración/preferencias.zy` — `no longer lexes: invalid character
  in string interpolation`, in a file with no interpolation in it

The gate refusing is correct behaviour and the reason the files are intact. They
are still two real programs `zymbol fmt` cannot format.

## Its place among the other layers

| | asks |
|---|---|
| **LDV** (`interpreter/LDV.md`) | can the language express this domain? |
| **ZyQuality** (`zyquality/`) | do the engines agree, and match their goldens? |
| **ZyDDT** (`ZyDDT/`) | do all engines still answer the same, correctly? |
| **ZyFormatCheck** (here) | did the formatter change anything it was not allowed to? |

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
