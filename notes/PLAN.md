# PLAN — COMPLETED (2026-11-22)

The implementation plan for Qumran shelf marks inside `\bref` is fully completed and verified.

## Summary of Accomplishments

1. **Split Proven (`\brx@split`):**
   - Verified xstring API: bracket argument for assignment (`\StrBefore{str}{ }[\res]` and `\StrBehind{str}{ }[\res]`).
   - Guarded with `\IfSubStr{#1}{ }` for bare shelf marks (e.g. `4Q521`).
   - Verified with harness `test-split.tex`.

2. **`\qref` Implemented:**
   - Evaluates `\brx@split{#1}`.
   - For bare shelf mark: dispatches `\ibibleverse{\brx@shelf}`.
   - With ref: dispatches `\ibibleverse{\brx@shelf}(\brx@ref)` with adjacent parenthesis.
   - Expands `\brx@shelf` so hooks and bibleref see plain character tokens.
   - Auto-registers on-the-fly under `28@Qumran` heading in imakeidx.

3. **`\bref` Router Implemented (`\NewDocumentCommand{\bref}{s o m}`):**
   - Splits input on `;` using `\DeclareListParser*` from `etoolbox`.
   - Walks items in reading order.
   - Preserves and prints `cf.` prefix directly in text without passing it to parser or index.
   - Detects Qumran items (first token starts with digit AND contains capital `Q`).
   - Groups contiguous standard runs and dispatches to `\pibibleverse` (plain), `\pibiblechvs` (`*`), or `\pibiblevs` (`o`).
   - Dispatches Qumran items to `\qref`.
   - Separates units with `\BRchsep` (defaults to `;\space`, book overrides to `;\ `).

4. **Book Surgery Completed:**
   - Removed duplicate local `\NewDocumentCommand{\bref}{s o m}` block from `book_peter/main.tex` (lines 132–139).
   - Preserved `\usepackage{bibleref-extra}` and `\renewcommand{\BRchsep}{;\ }`.

5. **Documentation Updated:**
   - `bibleref-extra/README.md` (§7.8, §8, §9)
   - `~/texmf/LATEX_STACK.md` (§4, §7)
   - `AGENT_INSTRUCTIONS.md` (§3, §8)
