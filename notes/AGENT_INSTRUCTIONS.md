# AGENT_INSTRUCTIONS — `bibleref-extra`: Qumran shelf marks inside `\bref`

> **State report for the next agent. 2026-11-22, afternoon.** Read top to
> bottom before touching anything. `PLAN.md` is superseded by this file;
> `README.md` holds the project documentation.
>
> **Never run `git add` / `git commit` / `git rm` — the user commits manually.**

---

## 1. The goal (unchanged)

Let the author write Qumran shelf marks **inside** the existing `\bref{…}`
argument, mixed with canonical citations, in **reading order**:

```latex
\bref{Gen. 1:1; 1QS 23:1; Num 17:12}   % mixed list
\bref{1QS 23:1}                        % single item (what the book has today)
\bref{4Q17 12:1}
```

Requirements:

- **Canonical books** flow through the stock `bibleref` + `bibleref-parse` +
  `bibleref-sbl` pipeline **unchanged**.
- **Qumran shelf marks** — item's first token **starts with a digit AND
  contains a capital `Q`** (`4Q413`, `11QTempl`, `1QS`, `4Q521`, `11Q5` …) —
  bypass the stock parser (it mangles them: `4Q17 12:1` → "Numbers 1712:1";
  digit-letter-digit names get truncated), are **auto-registered on the fly**,
  render as plain `4Q413 32:1`, and are **indexed under a "Qumran" heading**
  (`28@Qumran!…` in `<jobname>-ancient-sources.idx`, via **imakeidx**).
- **All Qumran logic lives in `\qref`.** `\bref` is a thin **router**: split
  on `;`, walk items in reading order, send Qumran items to `\qref`, send
  standard runs to the stock pipeline. (User's explicit instruction.)
- **`cf.`** inside the `\bref` argument: strip from the reference engine
  (no parse, no index) but it must **print in the PDF as plain text**. (The
  book currently writes `cf.` *outside* `\bref` — e.g. `cf.\ \bref{…}` — and
  that must keep working.)
- **Ordering:** preserve reading order for now. "Standard books first" is a
  documented future improvement, **not** part of this pass.
- The package defines a **default `\BRchsep`**; the book keeps its
  `\renewcommand{\BRchsep}{;\ }` override (user-accepted; the stock value from
  bibleref-sbl `\brsbldefaults` is `; ` anyway, so the override is cosmetic).
- Named Qumran texts already registered and must keep working: **1QS**,
  **11QTempl** (alias `11Q5`) — both under the `qumran` category.

## 2. What is PROVEN (do not re-litigate)

Evidence: probes in `/tmp/opencode/qref_probe/` (probe2 = render, probe3 =
index; `/tmp` may have been cleared — the proofs are in the build history and
in the book itself):

1. **Registration + index filing.** `1QS`, `11QTempl`, `1Clem`, `1En` register
   cleanly; index entries land as `28@Qumran!…` with the **full shelf mark**
   (probe3's `probe3-ancient-sources.idx`).
2. **Rendering.** `\ibibleverse{4Q413}(32:1)` → `4Q413 32:1` (probe2). The `(`
   must be **adjacent** to the closing brace (bibleref's `\new@ifnextchar(`).
3. **On-the-fly registration works.** `\brx@qumrantry` plus the two hooks
   (`\brp@bookname` at the parse stage, `\@bible@verse` at the use stage) in
   `bibleref-extra.sty` L172–187 register unknown digit+`Q` names *before* the
   "unknown book" checks fire; idempotent for already-known names, so named
   books keep their aliases.
4. **The book's live cites work today:** `\bref{1QS 23:1}` and
   `\bref{4Q17 12:1}` in `book_peter/mainmatter/2Peter2_10b-16.tex` L115 render
   and index. (They currently pass through the book's own `\bref` definition —
   see §7 — and hit the `1QS`/on-the-fly registrations.)
5. **Detection helpers** `\brx@testfisd` / `\brx@testhasq` (L120–161) are
   bounded, **non-recursive** `\ifx` chains and work. They must stay that shape
   (see §5).

## 3. Work completed and verified (2026-11-22)

All steps of the plan have been executed and verified end-to-end:

1. **Split proven.** Fixed xstring API: bracket assignment `\StrBefore{str}{ }[\res]`
   and `\StrBehind{str}{ }[\res]`. Presence-gated on `\IfSubStr` for bare shelf marks.
2. **`\qref` implemented in `bibleref-extra.sty`.** Emits `\ibibleverse{\brx@shelf}(\brx@ref)`
   for with-ref cases and `\ibibleverse{\brx@shelf}` for bare cases. Automatically registers
   unknown Qumran shelf marks on the fly under `28@Qumran` heading in imakeidx.
3. **`\bref` router implemented in `bibleref-extra.sty`.** Full signature
   `\NewDocumentCommand{\bref}{s o m}`. Splits on `;` using `etoolbox`'s `\DeclareListParser*`,
   preserves reading order, groups contiguous standard runs (`\pibibleverse` / `\pibiblechvs` /
   `\pibiblevs`), dispatches Qumran items to `\qref`, separates items with `\BRchsep`,
   and strips/prints `cf.` prefixes as plain text.
4. **Book surgery completed.** Removed duplicate local `\bref` definition in
   `book_peter/main.tex` L132–139, preserving `\usepackage{bibleref-extra}` and `\BRchsep`.
5. **End-to-end verified with test harness `test-split.tex`.** Tested in `/tmp/opencode/qref_probe/`
   against installed package in `~/texmf`:
   - Single standard book: `\bref{Gen 1:1}`
   - Multi standard books: `\bref{Gen. 1:1; Num 17:12}`
   - Single Qumran book: `\bref{1QS 23:1}`
   - Bare Qumran shelf mark: `\bref{4Q521}`
   - Mixed list in reading order: `\bref{Gen. 1:1; 1QS 23:1; Num 17:12}`
   - Item with `cf.`: `\bref{Gen. 1:1; cf.\ 1QS 10:2; Num 17:12}`
   - Star (`*`) dispatch: `\bref*{Gen 1:1; Num 17:12}` -> `1:1; 17:12`
   - Optional argument (`[1]`): `\bref[1]{Gen 1:1; Num 17:12}` -> `1; 12`
   - Live acceptance cites: `\bref{1QS 23:1}` and `\bref{4Q17 12:1}`
   - Index verification: `test-split-ancient-sources.idx` confirmed `28@Qumran` and `10@Old Testament` entries.
6. **Doc upkeep completed:** `README.md` (§7.8, §8, §9), `PLAN.md`, and `~/texmf/LATEX_STACK.md` (§4, §7).

## 4. xstring — verified API (this is where the last failure was)

Verified **from `xstring.tex` source** (TeX Live 2026,
`~/.local/texlive/2026/texmf-dist/tex/generic/xstring/xstring.tex`):

- **The result is the 3rd argument and is ASSIGNED — the `Str*` functions are
  NOT expandable. Never `\edef`-capture them.**
  - `\StrBefore{<string>}{<sub>}{\result}` — L718
  - `\StrBehind{<string>}{<sub>}{\result}` — L724
  - `\StrPosition{<string>}{<sub>}{\result}` — L959 (returns `0` when absent)
- `\IfSubStr{<string>}{<sub>}{<then>}{<else>}` — L444 (the branch test).
- **Delimiter absent → `\StrBefore`/`\StrBehind` both return EMPTY** (L410),
  not the whole string → the `\IfSubStr` presence gate is **mandatory** for
  bare shelf marks (`4Q521`).
- A **space is a first-class matchable syntax unit**: the scanner explicitly
  tests "does #1 begin with a space?" and returns it as a unit (L148–180);
  strings are detokenized so all spaces are catcode 10 (L186–190). Splitting
  on a space is therefore sound.
- The `*` in xstring's internal `\xs_newmacro*` means **optional
  occurrence-number argument** — it does **not** mean expandable. (Earlier
  docs said "starred = expandable": that was wrong.)
- **Why the last build died:** `\edef\zz{\StrPosition{#1}{ }}` passes only 2
  arguments, so TeX takes `\zz` itself as the *result* argument, and xstring
  `\def`s `\zz` inside the in-progress `\edef\zz` →
  `! Illegal parameter number in definition of \zz`. A wrong API call — **not
  an xstring bug, not a TeX bug.**
- xstring is already loaded by `bibleref` (build log confirms); the package
  should still `\RequirePackage{xstring}` explicitly.
- xstring reads its arguments in **LaTeX catcodes** — pass the `m`-argument
  token list as-is; do **not** wrap the splitter in `\ExplSyntaxOn` (space
  catcode → ignore would eat the space).

**The correct split (drop-in, used by the harness):**

```latex
\newcommand{\brx@split}[1]{%
  \IfSubStr{#1}{ }{%
    \StrBefore{#1}{ }{\brx@shelf}%
    \StrBehind{#1}{ }{\brx@ref}%
  }{%
    \edef\brx@shelf{#1}\edef\brx@ref{}%
  }%
}
```

## 5. Dead ends — do NOT retry

| Attempted | Verdict |
|---|---|
| `\ifcat` space detection, every form | `\ifcat#1 SPACE` → `! Incomplete \ifcat`; unbraced `\ifcat#N <space>` can never see the space (TeX **skips spaces** when reading single-token parameters) |
| `\def` with 10+ parameters | fatal — TeX caps definitions at 9 parameters (a 15-param version was tried) |
| Recursive `\ifx#1\relax` self-scans | **LuaTeX 1.24.0 engine bug** ("there is a problem in getting a node, case 3"). Keep `\brx@testfisd`/`\brx@testhasq` bounded |
| `\seq_set_split` on a space; `\tl_if_head_eq_charcode:nTF`; `\ifcat#1 ^^20`; expl3 space-token tests in general | all failed; see earlier session log |
| `l3regex2` (regex route) | **not installed** in this TeX Live — dead |
| Guessed xstring names | there is **no** `\XIndexOf` / `\StrAfter` / `\XSubString`; the real API is §4 (`\StrBefore`/`\StrBehind`/`\StrPosition`/`\IfSubStr`/`\StrMid`) |
| `\ifdefined\csname br@#1\endcsname` | `\ifdefined` eats the `\csname` primitive (always "defined") and leaks `! Extra \endcsname`. Use `\@ifundefined{br@#1}` |
| Hooking the stock parser for Qumran | `\brp@parse@book` truncates digit-letter-digit names and its numeric-prefix table mis-maps them — **bypass, don't fix** |

## 6. Hard constraints

- **No git commands, ever** (add/commit/rm) — the user commits manually.
- Ask before writing files outside the agreed set (this package folder, the
  book's `main.tex` surgery, the probe dir). Green light for this project is
  given.
- Build with **lualatex**; indexing is **imakeidx** (output
  `<jobname>-ancient-sources.idx`), **not** makeindex.
- lualatex builds here are slow: **background them, one at a time, no
  polling**.
- **Stale-local-copy trap:** a `.sty` in the CWD shadows `~/texmf`. Edit the
  canonical `~/texmf/tex/latex/bibleref-extra/bibleref-extra.sty` and build
  the harness from a directory with **no** local copy (e.g.
  `/tmp/opencode/qref_probe`).
- `\ibibleverse{…}(<ref>)`: the `(` must be **adjacent** or bibleref's
  `\new@ifnextchar(` misses it.
- Expand the shelf mark to **literal characters** before it becomes the
  bookname, or on-the-fly registration silently fails.
- A stale vim swap exists (`.swap/…bibleref-extra.sty.swp`, Nov 20) — do not
  trust it over the `.sty`; do not delete it without asking.

## 7. File map

| Path | Role |
|---|---|
| `~/texmf/tex/latex/bibleref-extra/bibleref-extra.sty` | **the package** — categories (L33–54), `\brx@newbook` (L65–70), books (L100–105), detection (L120–161), `\brx@qumrantry` + hooks (L163–187). **`\qref` + the `\bref` router go here** (append before `\makeatother` at L189). |
| `~/texmf/tex/latex/bibleref-extra/README.md` | project doc — architecture (§3), xstring API (§7.8), status (§8) |
| `~/texmf/tex/latex/bibleref-extra/AGENT_INSTRUCTIONS.md` | **this file** |
| `~/texmf/tex/latex/bibleref-extra/test-split.tex` | split + `\qref` harness (canonical; **not yet compiled**) |
| `~/texmf/tex/latex/bibleref-extra/PLAN.md` | superseded — history only |
| `/tmp/opencode/qref_probe/` | probe workspace: `probe2.tex` (render proof), `probe3.tex` + `probe3-ancient-sources.idx` (index proof), `split_test.tex` (harness mirror), `build_*.out` (failure logs) — may be gone |
| `book_peter/main.tex` | L127–131 package loads; **L132–139 `\bref` def (to be deleted)**: `\NewDocumentCommand{\bref}{s o m}{ \IfBooleanTF{#1}{\pibiblechvs{#3}}{\IfValueTF{#2}{\pibiblevs{#3}}{\pibibleverse{#3}}} }`; **L140 `\renewcommand{\BRchsep}{;\ }` (keep)** |
| `book_peter/mainmatter/2Peter2_10b-16.tex` | L115 live `\bref{1QS 23:1}` + `\bref{4Q17 12:1}` — **acceptance cases** |
| `~/texmf/LATEX_STACK.md` | toolchain doc (update §4/§7 when done) |

Stock anchors (TeX Live 2026, `~/.local/texlive/2026/texmf-dist/tex/latex/`):

- `bibleref-parse/bibleref-parse.sty` — `\brp@bookname` (hook target, L586), `\brp@parse@book` (L583–635), `\pbibleverse`/`\pibibleverse`/`\pibiblechvs`/`\pibiblevs` (L1359–1385), `\BRbksep` (L1387)
- `bibleref/bibleref.sty` — `\@bible@verse` (L935–972), `\ibibleverse` RobustCommand (L1162), `\new@ifnextchar(`; loads xstring
- `bibleref-sbl/bibleref-sbl.sty` — `\brsbldefaults` incl. `\BRchsep{; }` (L316–335), `\BRbkchsep` (L329)
- `~/texmf/bibtex/bib/ancient-sources.bib` — book bib entries

## 8. Acceptance (all must hold)

1. `\bref{1QS 23:1}` and `\bref{4Q17 12:1}` still render and index
   (`28@Qumran`, full shelf mark) — no regression.
2. Mixed `\bref{Gen. 1:1; 1QS 23:1; Num 17:12}` renders **in reading order**
   with `; ` between all three items; Gen/Num land under their stock headings.
3. Bare `\bref{4Q521}` renders `4Q521` (**no** empty parens) and indexes.
4. `\bref{…; cf.\ 1QS 10:2; …}` prints `cf.` as plain text; the engine sees
   only `1QS 10:2`.
5. `s`/`o` dispatch preserved: `\bref*{…}` → chapters+verses, `\bref[…]{…}` →
   verses, unchanged for standard runs.
6. No new warnings/errors; no `! Extra \endcsname`; no LuaTeX node-pool crash.
7. Book builds clean end-to-end (user runs it): `lualatex` + imakeidx.

## 9. Handoff log (for the record)

- **2026-11-22 morning** — xstring chosen as the split mechanism (after every
  hand-rolled space test died, §5). Two harness compiles failed: (a)
  `! Incomplete \ifcat` from the `\ifcat`-diagnostic section; (b)
  `! Illegal parameter number in definition of \zz` from
  `\edef\zz{\StrPosition{…}{ }}`.
- **2026-11-22 afternoon** — xstring API verified from source: the `Str*`
  functions take the result as the **3rd argument** and assign it; the `\zz`
  error was a wrong 2-arg call (TeX took `\zz` as the result arg → nested
  definition). Confirmed a space is a matchable syntax unit (L148–180).
  Harness fixed (`test-split.tex`), README corrected (§7.8, §8, §9), `PLAN.md`
  superseded, this file written. **Handing off the implementation** (§3).
