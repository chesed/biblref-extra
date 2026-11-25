# bibleref-extra

Non-canonical scripture citations for the **bibleref** stack. Lets an author
cite pseudepigraphal, **Qumran**, and patristic texts inline with the same
`\bref` entry point they already use for canonical books, and have them indexed
under proper headings (e.g. **Qumran**) exactly like any other book.

This is a small, self-contained extension package. It depends only on the stock
`bibleref`, `bibleref-parse`, and `bibleref-sbl` packages (already in TeX Live)
and is intended to be maintainable as a candidate for upstreaming.

---

## 1. What it does

| Need | How it works |
|---|---|
| Cite a **named** non-canonical book (`1 Clement`, `1 Enoch`, `1QS`, `11QTempl`) with its SBL abbreviation | Registered once via `\brx@newbook`; `\bref` and `\ibibleverse` then treat it like a canonical book. |
| Cite an **anonymous Qumran shelf mark** (`4Q413`, `4Q521`, `11Q5`, …) with no per-document setup | Detected by rule (digit + capital `Q`) and **auto-registered on the fly**; cited via `\qref`; indexed under **Qumran**. |
| **Index** all of the above under stable, auto-numbered headings | Each book is filed under a category (`qumran`, `stl`, `cf`) with an auto-assigned entry number. |

**The author never registers anything.** `\bref` is the single entry point; Qumran
marks just work.

---

## 2. The two entry points

### `\bref{…}` — the router (lives in this package)

`\bref` is the author's entry point. It **routes** each citation item:

- **Standard books** (OT / NT / Deuterocanonical, and registered named books)
  flow through the stock `bibleref-parse` + `bibleref-sbl` pipeline, unchanged.
- **Qumran items** are handed to `\qref`.

It preserves the stock `\bref` dispatch (`s` → `\pibiblechvs`, `o` →
`\pibiblevs`, plain → `\pibibleverse`) for the standard runs, and inserts the
book's `\BRchsep` between items.

```latex
\bref{Gen. 1:1}                 % standard
\bref{1QS 23:1}                 % Qumran -> \qref
\bref{Gen. 1:1; 1QS 23:1; Num 17:12}   % mixed: standard run, Qumran, standard run
```

### `\qref{…}` — the Qumran command (lives in this package)

`\qref` holds **all** Qumran-specific logic. Given a shelf mark with an optional
reference, it:

1. **Splits** the argument into *shelf mark* + *reference* (e.g. `4Q413 32:1`
   → shelf `4Q413`, ref `32:1`; bare `4Q521` → shelf `4Q521`, no ref) — via
   **xstring** (see §7, item 8).
2. **Registers** the shelf mark on the fly if it is not already known
   (idempotent; named texts like `1QS`/`11QTempl` keep their aliases).
3. **Renders** as `\ibibleverse{<shelf>}(<ref>)` (bare: `\ibibleverse{<shelf>}`).
4. **Indexes** it under **Qumran** via the existing `\@bible@verse` hook.

```latex
\qref{4Q413 32:1}   % ->  4Q413 32:1
\qref{1QS 23:1}     % ->  1QS 23:1
\qref{4Q521}        % ->  4Q521
```

`(\bref)` only ever *calls* `\qref` for Qumran items; it contains no Qumran
rendering or registration logic itself.

---

## 3. Architecture

```
\bref{ARG}
  └─ router: split ARG on ";", walk items in reading order
       ├─ standard run ──> \pibibleverse / \pibiblevs / \pibiblechvs   (stock pipeline)
       └─ Qumran item  ──> \qref{item}
                              ├─ split shelf | ref        (xstring: \IfSubStr + \StrBefore/\StrBehind)
                              ├─ \ibibleverse{shelf}(ref)
                              │     └─ \@bible@verse hook ──> \brx@qumrantry
                              │                                   ├─ register (if unknown)
                              │                                   ├─ render
                              │                                   └─ index  (28@Qumran)
```

**Why pre-route instead of hooking the parser?** The stock parser
(`\brp@parse@book`) **truncates digit-letter-digit names** and its numeric-prefix
table mis-maps shelf marks (`4Q17 12:1` → "Numbers 1712:1"). So Qumran items
**bypass the stock parser entirely**; `\qref` emits the already-parsed
`\ibibleverse{…}(…)` form directly.

**The split.** The shelf mark is always the **first word** (`1QS`, `4Q413`,
`11QTempl`, `11Q5`); the reference is the rest. We split on the **first space**
with **xstring** — an `\IfSubStr` presence gate plus `\StrBefore`/`\StrBehind` —
not with a character scan: every hand-rolled space test in plain LaTeX/expl3 is
a dead end (§7, item 7). xstring is already loaded by bibleref and handles
spaces in strings correctly.

---

## 4. The Qumran detection rule

An item is a Qumran shelf mark **iff its first token starts with a digit AND
contains a capital `Q`**. Implemented as two bounded, **non-recursive** `\ifx`
chains (see §7 for why they must stay non-recursive):

- `\brx@testfisd{token}` → true iff the first character is a digit.
- `\brx@testhasq{token}` → true iff the token contains a `Q`.

Both are no-ops once a book is already known, so named texts keep their
registered identities and nothing is double-registered.

---

## 5. Index categories

Categories are registered once; books auto-file and auto-number under them.

| Key      | Sort | Heading                    |
|----------|------|----------------------------|
| `stl`    | 25   | Second Temple Literature   |
| `qumran` | 28   | Qumran                     |
| `cf`     | 40   | Church Fathers             |

These slot between the stock headings owned by `bibleref-sbl`
(OT `10@`, Deuterocanonical `20@`, NT `30@`). Entries land in
`<jobname>-ancient-sources.idx` via **imakeidx** (not `makeindex`).

---

## 6. API

### Add a book — one line
```latex
\brx@newbook {canonical} {SBL abbreviation} {aliases,comma,list} {category}
\brx@newbook {1Clem} {1~Clem.} {1Clem,1Clement} {cf}
\brx@newbook {1QS}   {1QS}     {1QS}            {qumran}
```

### Add a category — one line
```latex
\brx@newcategory{key}{sort-number}{Heading}
\brx@newcategory{qumran}{28}{Qumran}
```

Books are numbered automatically in the order added; you never hand-maintain
`\biblerefmap` lines or book numbers.

---

## 7. Constraints & known issues (read before editing)

1. **No recursive `\ifx` self-scans.** A recursive `\ifx#1\relax …` scan trips a
   **LuaTeX 1.24.0 engine bug** ("there is a problem in getting a node, case 3")
   in this build. Keep `\brx@testfisd` / `\brx@testhasq` as **bounded,
   non-recursive** `\ifx` chains.
2. **`\ExplSyntaxOn` drops spaces at read time** (space catcode → ignore). Any
   content the splitter reads must be read in **LaTeX catcodes** (the `m`
   argument of `\bref`/`\qref`, called from the document body). Do not wrap the
   splitter input in an explicit `\ExplSyntaxOn…\ExplSyntaxOff`.
3. **`\ibibleverse{…}(<ref>)` — the `(` must be adjacent** (no space) or
   bibleref's `\new@ifnextchar(` misses it.
4. **Expand the shelf mark to literal characters before it becomes the
   bookname**, or on-the-fly registration silently fails.
5. **`\@ifundefined{br@<name>}`** is the correct idiom to test "is this book
   known?"; do **not** use `\ifdefined\csname br@<name>\endcsname` (it consumes
   the `\csname` primitive and leaks `! Extra \endcsname`).
6. **`l3regex2` is not installed** → the whole regex route is dead. Do not use it.
7. **`\ifcat` space-detection is dead — do not retry.** Three independent
   failures: a 15-parameter `\def` is invalid (TeX caps definitions at 9
   parameters); `\ifcat` with a space operand → `! Incomplete \ifcat`; and TeX
   **skips spaces** when reading single-token parameters, so an unbraced
   `\ifcat#N <space>` scan can never see the space. Same verdict for the older
   attempts: `\seq_set_split` on a space, `\tl_if_head_eq_charcode:nTF`,
   `\ifcat#1 ^^20` on a parameter-held space, and expl3 space-token tests in
   general.
8. **xstring (the split mechanism) — verified from `xstring.tex` (2026-11-22).**
   Already loaded by `bibleref` (the package should still
   `\RequirePackage{xstring}` explicitly). **The `Str*` functions ASSIGN a
   result; the result is the third argument — they are NOT expandable, never
   `\edef`-capture them:** `\StrBefore{<string>}{<sub>}{\result}` (L718) ·
   `\StrBehind{…}{…}{\result}` (L724) · `\StrPosition{…}{…}{\result}`
   (L959, returns `0` when absent). `\IfSubStr{<string>}{<sub>}{<then>}{<else>}`
   (L444) branches. (The `*` in xstring's internal `\xs_newmacro*` means an
   **optional occurrence-number** argument — it does NOT mean expandable.)
   A space is a first-class, matchable syntax unit (L148–180: the scanner
   explicitly tests "does #1 begin with a space?"; L186–190 detokenize
   spaces to catcode 10). When the delimiter is **absent**:
   `\StrBefore`/`\StrBehind` both return **empty** (not the whole string,
   L410) → gate on `\IfSubStr{<str>}{ }{…}{…}` for bare shelf marks.
   **xstring result syntax note:** The assigned result argument in xstring is an
   **optional bracket argument** `[<macro>]` (e.g. `\StrBefore{str}{ }[\res]`),
   not a curly-brace argument. Calling `\StrBefore{str}{ }` without brackets
   prints directly to PDF.
   The correct split:
   ```latex
   \newcommand{\brx@split}[1]{%
     \IfSubStr{#1}{ }{%
       \StrBefore{#1}{ }[\brx@shelf]%
       \StrBehind{#1}{ }[\brx@ref]%
     }{%
       \edef\brx@shelf{#1}\edef\brx@ref{}%
     }%
   }
   ```

---

## 8. Status (2026-11-22)

- **Completed & Proven:**
  1. Shelf/ref split via presence-gated `xstring` (`\brx@split` with `\IfSubStr`, `\StrBefore[...][\res]`, `\StrBehind[...][\res]`).
  2. `\qref` implemented and verified: produces `\ibibleverse{shelf}(ref)` with adjacent parenthesis for with-ref cases and `\ibibleverse{shelf}` for bare shelf marks.
  3. `\bref{s o m}` router implemented: splits on `;` using `\DeclareListParser*` from `etoolbox`, walks items in reading order, groups contiguous standard runs (`\pibibleverse`, `\pibiblechvs`, `\pibiblevs`), dispatches Qumran items via `\qref`, separates units with `\BRchsep`, and strips/prints `cf.` prefixes as plain text.
  4. End-to-end verified with test harness `test-split.tex`: tested single standard, multi-standard, single Qumran, bare Qumran, mixed lists, `cf.` prefixes, star (`*`), and optional argument (`[o]`). Indexing verified under `28@Qumran` and `10@Old Testament`.
  5. `book_peter/main.tex` updated: deleted duplicate local `\bref` macro, leaving package `\bref` active.

## 9. Changelog

- **2026-11-22 (late afternoon)** — Implemented and proven end-to-end:
  - Correct xstring bracket-assignment API documented and verified (`[macro]`, not `{macro}`).
  - `\qref` and `\bref` router added to `bibleref-extra.sty`.
  - Contiguous standard run grouping + `cf.` handling + `s`/`o` dispatch verified.
  - Harness `test-split.tex` compiles clean with LuaLaTeX; PDF and index confirmed.
  - `book_peter/main.tex` local `\bref` removed in favor of package implementation.
- **2026-11-22** — xstring API verified from source: `Str*` functions take the
  result as an optional bracket argument and assign it; confirmed a space is a matchable syntax unit in xstring's scanner (L148–180).
- **2026-11-22 (morning)** — Split mechanism settled on **xstring** (already
  loaded by bibleref); `\ifcat` space-detection retired as a dead end (§7.7);
  project doc written; `PLAN.md` superseded.
- **2026-11-19** — Modular non-canonical book registration
  (`\brx@newcategory` + `\brx@newbook`); registered 1 Clement (`cf`),
  1 Enoch + 1QS (`stl`), `11QTempl` (`qumran`).
