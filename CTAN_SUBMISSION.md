# CTAN Submission Checklist — `bibleref-extra`

**Status:** not started (planning draft, 2026-11-22).
**Goal:** publish `bibleref-extra` on CTAN (ctan.org), from which TeX Live ships it.

---

## 1. Requirements (what CTAN demands)

- [ ] Top-level folder named after the package: `bibleref-extra/`
- [ ] License declared in the `.sty` header **and** a `LICENSE` file (recommend **LPPL v1.3c+**, same as bibleref)
- [ ] LaTeX manual: `doc/bibleref-extra-doc.tex` → PDF in `doc/` (a README does **not** count)
- [ ] `CHANGES` file — dated revision log, newest first (mandatory)
- [ ] `README` — brief (recommended)
- [ ] Working example(s) in `examples/`
- [ ] No junk in the zip: `.bak-*`, `.git/`, `.swap/`, build artifacts
- [ ] `.sty` header: `\ProvidesPackage` with ISO date (already present) + license statement
- [ ] Zip internal top level is the package folder: `bibleref-extra/bibleref-extra.sty`

## 2. Pre-submission tidy-up (package-specific)

- [ ] Pick a license (LPPL) and add the standard header block to `bibleref-extra.sty`
      (Copyright holder + year, LPPL terms, "may be distributed and/or modified
      only under the conditions of the LPPL")
- [ ] Write `doc/bibleref-extra-doc.tex` from the README content: overview,
      `\bref` router + `\qref`, API, index-category table, typeset examples,
      `\changes` section
- [ ] Create `CHANGES` (date + one line per revision; port README §9)
- [ ] Trim `test-split.tex` → `examples/` as a minimal working example
- [ ] **Decide:** `\bref` is redefined unconditionally — consider making the
      router opt-in via a package option (e.g. `\usepackage[router]{bibleref-extra}`)
      before a reviewer raises it
- [ ] Exclude `.bak` files from the zip (may stay on disk)

## 3. Verify

- [ ] `pip install ctk`
- [ ] `ctk test bibleref-extra.zip`
- [ ] Fix until clean (license lines, CHANGES, file dates, naming, junk)
      (free online tier also available: `ctan.org/ctan-check`)

## 4. Submit

- [ ] Create free account at `ctan.org`
- [ ] Zip: from `~/texmf/tex/latex/` → `zip -r bibleref-extra.zip bibleref-extra/`
- [ ] Upload at `ctan.org/upload` (or email zip to `ctan-upload@tug.org`)
- [ ] Choose top-level category **LaTeX** (subcategory assigned by maintainer)
- [ ] Wait days–2 weeks; if nothing after ~2 weeks, email the category
      maintainer (listed on ctan.org)

## 5. Post-publication

- [ ] Future changes: fix → bump `\ProvidesPackage` date → update `CHANGES`
      → re-zip → re-upload to the **same** CTAN entry
- [ ] Note in `~/texmf/LATEX_STACK.md` §4: the "restore warning" is retired —
      the package now survives a TeX Live reinstall (system copy shadows local)
