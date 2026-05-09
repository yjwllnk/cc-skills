---
name: pdf2willbook
description: "Convert PDF lecture notes into Will-format LaTeX book. Two phases: Phase A scaffolds a new subject directory; Phase B fills chapters one at a time (vision-extracts figures). Trigger: 'scaffold <subject>', 'do ch<NN>', 'fill chapter', 'pdf2willbook'. Also auto-applies whenever working in a study-session subject dir."
---

# pdf2willbook — operating contract

Convert PDF lecture notes into a structured LaTeX book following Will's house style. One subject = one dotfile book dir + lecture-source symlinks. Two-phase workflow: scaffold first, fill chapter by chapter.

---

## Directory layout

```
$HOME/.claude-dnjf/WDIR/study-session/
  <subject-name>/
    <short>/                          # symlink → $HOME/.dnlf/__mdil__/notes/<area>/<short>/
    lecture.sources/
      <Class Name>/                   # symlink → iCloud course folder (read-only)

$HOME/.dnlf/__mdil__/notes/<area>/<short>/
    wll<SHORT>.sty
    preamble.tex
    main.tex
    solo.tex
    solo.sh
    refs.bib
    chapters/NN-<topic>.tex
    appendix/A-*.tex  appendix/H-homework-solutions.tex
    figures/
    build/
    tmp/
    lecture-notes/                    # symlink → lecture.sources/<Class Name>
```

**Canonical example:** `quantum-chemistry/qchem/` — read it for template reference before scaffolding.

---

## Permission scope

**Read + write:** active subject's dotfile target and all subpaths.  
**Read only:** `lecture.sources/` targets (iCloud); other completed books (e.g. `qchem`); `~/.config/tex/` sty files.  
**FORBIDDEN — never write:** `~/Library/Mobile Documents/com~apple~CloudDocs/`; any other subject's dotfile dir; `~/.config/tex/wlltex0.sty`, `wllMDIL.sty`, `wll.sty`, `wllplstyle.sty`.  
**Confirm before:** `rm -rf` beyond `build/`; moves out of active subject dir; any git operations.

---

## Phase model

| Phase | Trigger | Goal |
|-------|---------|------|
| **A — scaffold** | `scaffold <subject>` or first setup | Mirror qchem template. Build must pass clean. |
| **B — fill** | `do ch<NN>` / `fill <chapter>` | Per-chapter. Read source PDF, write `chapters/NN-*.tex`. One chapter per trigger — never bulk-fill. |

---

## Phase A — scaffold checklist

Run all 10 steps in order. Declare done only after step 10 passes.

1. `wll<SHORT>.sty` — defines `\Course`, `\Institution`, `\Semester`, `\ProfDepartment`, `\ProfName`, `\ProfEmail`, `\ProfHomepage`, `\Category`, `\SubCategory`.
2. `preamble.tex` — clone of qchem preamble; swap `wllQCHEM` → `wll<SHORT>`.
3. `main.tex` — see **Book template constants** and **ToC + Appendix rules** below.
4. `solo.tex` + executable `solo.sh` — single-chapter dev loop.
5. `refs.bib` — primary refs from syllabus + field staples.
6. `chapters/NN-<topic>.tex` stubs — `\chapter{}`, `\label{}`, section outline, `% Source: <pdf-name>` header.
7. `appendix/A-*.tex`, `appendix/H-homework-solutions.tex` stubs.
8. `lecture-notes/` symlink → iCloud course folder (read-only).
9. Empty dirs: `build/`, `figures/`, `tmp/`.
10. Test build: `latexmk -pdf -output-directory=build -interaction=nonstopmode main.tex` — must finish with `Output written on build/main.pdf`.

---

## Book template constants

Do not change without an explicit reason.

```latex
\documentclass[10pt,oneside]{book}
```

**Geometry (B5 landscape — mandatory):**
```latex
\geometry{b5paper, landscape,
  top=0.05in, bottom=0.05in, left=0.2in, right=0.2in,
  headheight=1.25em, headsep=0.5em, footskip=1.75em,
  includehead, includefoot}
```

**Math sizing:**
```latex
\DeclareMathSizes{10}{7}{7}{5}
\DeclareMathSizes{9}{7}{7}{5}
\DeclareMathSizes{8}{7}{7}{5}
```

**Body shrink:**
```latex
\AtBeginDocument{\small}
```

**Two-column (multicol, not twocolumn class option):**
- `preamble.tex` must load `\usepackage[tikz,lecturenotes,physics]{wlltex0}` — this pulls in `multicol`, sets `\columnsep=1em`, `\columnseprule=0.05em`.
- **Every chapter body** wraps its content in `\begin{multicols}{2}...\end{multicols}`.
- Wide figures/tables that need the full page width: break out with `\end{multicols}` before, `\begin{multicols}{2}` after. Never use `figure*` — it does not work with the `multicol` package.

**fancyhdr (copy from qchem exactly):**
```latex
\fancyhead[L]{\small \nouppercase\leftmark}
\fancyhead[R]{\small \nouppercase\rightmark}
\fancyfoot[L]{\footnotesize <Course Title>}
\fancyfoot[R]{\footnotesize Yujin Kang, MDIL}
\fancyfoot[C]{\small \thepage}
\renewcommand{\footrulewidth}{0.03em}
\renewcommand{\headrulewidth}{0.03em}
\renewcommand{\chaptermark}[1]{\markboth{\thechapter.\ #1}{}}
\renewcommand{\sectionmark}[1]{\markright{\thesection.\ \textit{#1}}}
```

**Theorem environments (numbered by chapter):**
```latex
\theoremstyle{plain}
\newtheorem{thm}{Theorem}[chapter]
\newtheorem{prop}[thm]{Proposition}
\newtheorem{lem}[thm]{Lemma}
\newtheorem{cor}[thm]{Corollary}
\theoremstyle{definition}
\newtheorem{defn}[thm]{Definition}
\newtheorem{post}[thm]{Postulate}
\theoremstyle{remark}
\newtheorem*{rem}{Remark}
```

**Chapter format:**
```latex
\titleformat{\chapter}[hang]{\large\bfseries}{\thechapter.}{0.15em}{}
\titlespacing*{\chapter}{0pt}{0.15em}{0.075em}
```

---

## ToC rules — mandatory

### "Table of Contents" must appear in the ToC with fancyhdr sidebars

Place this block in `main.tex` where `\tableofcontents` normally goes:

```latex
\cleardoublepage
\addcontentsline{toc}{chapter}{\contentsname}
\markboth{\contentsname}{\contentsname}
\tableofcontents
\pagestyle{fancy}
```

- `\addcontentsline` inserts the ToC entry itself.
- `\markboth` sets both header marks so fancyhdr shows "Table of Contents" in the left sidebar on the ToC pages.
- `\pagestyle{fancy}` re-engages fancyhdr after `\tableofcontents` resets it.

### Appendix must appear as "Appendix" in the ToC

```latex
\appendix
\addtocontents{toc}{\protect\addvspace{1em}}
\addcontentsline{toc}{part}{Appendix}
\input{appendix/A-...}\clearpage
...
```

- `\addcontentsline{toc}{part}{Appendix}` inserts a part-level (bold, full-width) "Appendix" divider line before appendix chapters in the ToC.
- `\addvspace{1em}` adds vertical breathing room above it.
- Individual appendix chapters then appear as A, B, C... beneath this divider.

---

## Phase B — fill rules

One chapter per trigger. Confirm source PDF before writing.

### Transcription contract

- **Header:** every chapter file starts with `% Source: <pdf-name>`.
- **Completeness:** every word, letter, equation, table heading, bullet point in the source slides must appear in the TeX output. Zero omissions. If a slide has content, it goes in.
- **Rewriting allowed:** format may change freely. Use `\begin{wlldash}`, `\begin{wllbul}`, `\begin{enumerate}`, `\begin{tabular}`, `\begin{equation}`, paragraphs — whatever fits the slide's information geometry better than raw prose.
- **Facts exact:** equations, derivations, definitions, named theorems, parameter values, method names, citations — character-for-character faithful.
- **No insertion:** do not add knowledge, context, or explanation beyond what the PDF contains. This is transcription, not interpretation.

### Figure extraction (vision)

- Use the vision model to read each slide image from the source PDF.
- **Every figure** in the lecture notes must be extracted and included — no exceptions.
- Workflow per figure:
  1. Identify the figure on the slide (use vision to isolate it if possible).
  2. Save the extracted image as `figures/chNN-fig<M>.<ext>` (png preferred).
  3. Include in TeX:
     ```latex
     \begin{figure}[H]
       \centering
       \includegraphics[width=\linewidth]{figures/chNN-figM.png}
       \caption{<caption from slide, or descriptive if none>}
       \label{fig:chNN-figM}
     \end{figure}
     ```
  4. If figure is too wide for one multicol column: break out of multicols, include full-width, resume multicols.

### Two-column chapter structure

```latex
\chapter{Chapter Title}
\label{ch:slug}
% Source: lecture-slide-filename.pdf

\begin{multicols}{2}

% ... all section/subsection content here ...

\end{multicols}
```

Wide elements (full-page tables, wide figures): temporarily exit multicols as described above.

### Build verification after each chapter

After filling a chapter, run:
```bash
./solo.sh <stem>    # e.g. ./solo.sh 04-dft
```
Must compile without errors before declaring chapter done.

---

## wlltex0.sty — available macros and environments

Key items from `~/.config/tex/wlltex0.sty`. Use these instead of rolling custom equivalents.

**List environments:**
| Env | Style |
|-----|-------|
| `wlldash` | `--` bullet |
| `wllbul` | `•` bullet |
| `wllnum` | `1.` numbered |
| `wllalph` | `(a)` lettered |
| `wllalphH` | `(A)` uppercase |
| `wllrome` | `i.` roman |
| `wllRome` | `I.` Roman uppercase |

**Highlighting:**
```latex
\highlight[yellow]{text}          % colored box, default yellow
\highlight[red!20]{important}
```

**Math helpers:**
```latex
\diff          % upright d for integrals: \int f \diff x
\ee            % upright e (Euler's number)
\ii            % upright i (imaginary unit)
\vect{v}       % bold vector
\mat{A}        % bold matrix
\avg{O}        % <O> angle brackets
\absval{x}     % |x|
\normval{v}    % ||v||
```

**Code blocks (from wlltex0 `code` option, always on):**
```latex
\begin{bashcode}...\end{bashcode}
\begin{pycode}...\end{pycode}
\begin{Ccode}...\end{Ccode}
\begin{stdout}...\end{stdout}
```

**Physics package** (loaded via `[physics]` option): `\dv`, `\pdv`, `\bra`, `\ket`, `\braket`, `\norm`, `\comm`, `\anticomm`, etc.

**Colors defined:** `wllnavy` (#0B1F33), `wllblue` (#0088FF), `wllgray` (#5A6B7A).

**wllMDIL.sty defines:** `\Name` (Yujin Kang), `\Lab` (MDIL), `\Department`, `\StudentNumber`, `\EmailPrimary`, etc.

---

## Build commands

```bash
# Full book
TEXINPUTS=$HOME/.config/tex//: latexmk -pdf -output-directory=build -interaction=nonstopmode main.tex

# Single chapter dev loop
./solo.sh <stem>                  # e.g. ./solo.sh 04-dft

# Nuke stale aux/toc/idx (use when formatting looks corrupted)
latexmk -C -output-directory=build && latexmk -pdf -output-directory=build -interaction=nonstopmode main.tex

# Inspect specific page
pdftoppm -f <n> -l <n> -r 100 build/main.pdf /tmp/out -png
```

**`TEXINPUTS` must be set** for `wll*.sty` resolution. Verify with `kpsewhich wlltex0.sty` before first build.

---

## Known gotchas

- **iCloud paths:** contain spaces and `~`; always quote in shell commands.
- **Symlink + latexmk:** `-output-directory` on a symlinked path confuses latexmk. Always keep `build/` inside the dotfile target (not the WDIR symlink side) — current layout is safe.
- **TEXINPUTS:** if `kpsewhich wlltex0.sty` returns empty, `wll*.sty` won't resolve. Export `TEXINPUTS=$HOME/.config/tex//:` in shell or prepend to every latexmk call.
- **Stale aux/toc/idx:** poisoned state causes layout corruption on subsequent passes. Symptom: bizarre chapter/section spacing or missing headers. Fix: nuke with `latexmk -C` then rebuild.
- **`figure*` + multicol:** `figure*` is undefined inside the `multicol` environment. Use `\end{multicols}`, full-width figure, `\begin{multicols}{2}` instead.
- **`lecturenotes` option geometry:** wlltex0's `lecturenotes` mode sets its own geometry. The explicit `\geometry{...}` call in `main.tex` must come after `\usepackage{wlltex0}` to override it. Verify geometry is b5+landscape in the compiled PDF.
- **`lecutre-sources` vs `lecture.sources`:** typo exists in some existing WDIR layouts. Propagate whichever spelling is already present; do not silently rename.
