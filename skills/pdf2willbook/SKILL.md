---
name: pdf2willbook
description: "Convert PDF lecture notes into Will-format LaTeX book (B5 landscape, multicol). Two phases: Phase A scaffolds a new subject directory; Phase B fills chapters one at a time with vision figure extraction. Triggers: 'scaffold <subject>', 'do ch<NN>', 'fill chapter', '/pdf2willbook'."
---

# pdf2willbook — operating contract

Convert PDF lecture notes into a structured LaTeX book following Will's house style. One subject = one dotfile book dir + lecture-source symlinks. Two-phase workflow: scaffold first, fill chapter by chapter.

**Companion file:** [`wlltex0-cheatsheet.md`](./wlltex0-cheatsheet.md) — macros, environments, math helpers from `wlltex0.sty`/`wllMDIL.sty`. Read it before transcribing chapters.

**Account context:** the user's personal Claude Code account is `~/.claude-dnjf` — that's where `WDIR/study-session/` lives and where this skill is normally used. The work-account `~/.claude-mdil` exists but study-session is not active there. Both accounts have this skill installed; only `.claude-dnjf` actively uses it.

---

## Directory layout

```
~/.claude-dnjf/WDIR/study-session/
  <subject-name>/
    <short>/                          # symlink → ~/.dnlf/__mdil__/notes/<area>/<short>/
    lecture.sources/
      <Class Name>/                   # symlink → iCloud course folder (read-only)

~/.dnlf/__mdil__/notes/<area>/<short>/
    wll<SHORT>.sty
    preamble.tex
    main.tex
    solo.tex
    solo.sh
    refs.bib
    chapters/NN-<topic>.tex
    appendix/A-*.tex  appendix/H-homework-solutions.tex
    figures/lec<NN>/p<M>.png          # one subdir per chapter, page-numbered images
    build/
    tmp/
    lecture-notes/                    # symlink → lecture.sources/<Class Name>
```

**Canonical examples:** `quantum-chemistry/qchem/` (older) and `computational-chemistry/comp.chem/` (more recent). Read both for template reference; comp.chem reflects current practice.

---

## Permission scope

**Read + write:** active subject's dotfile target and all subpaths.  
**Read only:** `lecture.sources/` targets (iCloud); other completed books; `~/.config/tex/` sty files.  
**FORBIDDEN — never write:** `~/Library/Mobile Documents/com~apple~CloudDocs/`; any other subject's dotfile dir; `~/.config/tex/wlltex0.sty`, `wllMDIL.sty`, `wll.sty`, `wllplstyle.sty`.  
**Confirm before:** `rm -rf` beyond `build/`; moves out of active subject dir; any git operations.

---

## Phase model

| Phase | Trigger | Goal |
|-------|---------|------|
| **A — scaffold** | `scaffold <subject>` or first setup | Create directory, file templates, symlinks. Build must pass clean. |
| **B — fill** | `do ch<NN>` / `fill <chapter>` | Per-chapter. Read source PDF, write `chapters/NN-*.tex`. One chapter per trigger — never bulk-fill. |

Detect which phase to enter: if the dotfile target dir doesn't exist or `main.tex` is missing, run Phase A. Otherwise Phase B.

---

## Phase A — scaffold

Run all 10 steps in order. Declare done only after step 10 passes.

1. **Create dotfile target dir:** `~/.dnlf/__mdil__/notes/<area>/<short>/`.
2. **Create WDIR subject dir + symlinks:**  
   `mkdir -p ~/.claude-dnjf/WDIR/study-session/<subject-name>/lecture.sources/`  
   `ln -s ~/.dnlf/__mdil__/notes/<area>/<short> ~/.claude-dnjf/WDIR/study-session/<subject-name>/<short>`  
   `ln -s "<iCloud course folder>" ~/.claude-dnjf/WDIR/study-session/<subject-name>/lecture.sources/<Class Name>`
3. **Write `wll<SHORT>.sty`** (see [template](#wllshortsty-template) below).
4. **Write `preamble.tex`** (see [template](#preambletex-template) below).
5. **Write `main.tex`** (see [template](#maintex-template) below).
6. **Write `solo.tex` + `solo.sh`** (see [templates](#solotex-template) below). `chmod +x solo.sh`.
7. **Write `refs.bib`** — primary refs from syllabus + field staples (a few `@article` entries enough).
8. **Write chapter stubs:** `chapters/NN-<topic>.tex` per syllabus, each with `\chapter{...}`, `\label{ch:...}`, `% Source: <pdf-name>` header.
9. **Write appendix stubs + create empty dirs:** `appendix/A-*.tex`, `appendix/H-homework-solutions.tex`, `build/`, `figures/`, `tmp/`. Symlink `lecture-notes/ → ../lecture.sources/<Class Name>`.
10. **Test build:**  
    `TEXINPUTS=$HOME/.config/tex//: latexmk -pdf -output-directory=build -interaction=nonstopmode main.tex`  
    Must finish with `Output written on build/main.pdf`.

If step 10 fails, fix the build before declaring scaffold done.

---

## File templates

### `wll<SHORT>.sty` template

```latex
\NeedsTeXFormat{LaTeX2e}
\ProvidesPackage{wll<SHORT>}[variables]

\RequirePackage{fancyhdr}
\RequirePackage{fontawesome5}

\newcommand{\Category}{Notes}
\newcommand{\SubCategory}{<SubjectCamelCase>}

\newcommand{\Course}{<Course Title>}
\newcommand{\Institution}{SNU}
\newcommand{\Semester}{<YYYY Season>}

\newcommand{\ProfDepartment}{<Dept>}
\newcommand{\ProfName}{<Prof Full Name>}
\newcommand{\ProfHomepage}{<URL or empty>}
\newcommand{\ProfEmail}{<email>}
```

### `preamble.tex` template

```latex
% --- Encoding & language ---
\usepackage[utf8]{inputenc}
\usepackage[T1]{fontenc}
\usepackage[american]{babel}

% --- natbib BEFORE hyperref (hyperref loaded inside wlltex0) ---
\usepackage[round,authoryear]{natbib}

% --- House style ---
\usepackage[tikz,lecturenotes,physics]{wlltex0}
\usepackage{wllMDIL}
\usepackage{wll<SHORT>}

% --- Things wlltex0 doesn't cover ---
\usepackage{amsthm}
\usepackage{microtype}
\usepackage{float}
\usepackage[section]{placeins}
\setcounter{totalnumber}{20}

\usepackage[colorinlistoftodos,prependcaption,textsize=small]{todonotes}
\usepackage{makeidx}
\makeindex

\usepackage{adjustbox}
\usepackage{wrapfig}    % only if wrapfig figures used; see Phase B figure rules

% Korean (only if any slide contains Hangul)
% \usepackage{kotex}

% Underline (\uline)
\usepackage[normalem]{ulem}

% --- Body font shrink ---
\AtBeginDocument{\small}

% --- Theorem environments (numbered by chapter) ---
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

\makeatletter
\renewcommand{\tagform@}[1]{\maketag@@@{\footnotesize(\ignorespaces#1\unskip\@@italiccorr)}}
\makeatother

% --- Chapter format ---
\titleformat{\chapter}[hang]{\large\bfseries}{\thechapter.}{0.15em}{}
\titlespacing*{\chapter}{0pt}{0.15em}{0.075em}
```

### `main.tex` template

The ToC-in-ToC pattern below is **verified to compile and render headers correctly** under B5 landscape + titlesec + multicol (tested 2026-05-09 against full preamble).

```latex
\documentclass[10pt,oneside]{book}
\input{preamble}

\title{<Course Title> Notes}
\author{Yujin Kang}

\usepackage{fancyhdr}
\pagestyle{fancy}
\fancyhf{}
\fancyhead[L]{\small \nouppercase\leftmark}
\fancyhead[R]{\small \nouppercase\rightmark}
\fancyfoot[L]{\footnotesize <Course Title>}
\fancyfoot[R]{\footnotesize Yujin Kang, MDIL}
\fancyfoot[C]{\small \thepage}
\renewcommand{\footrulewidth}{0.03em}
\renewcommand{\headrulewidth}{0.03em}
\renewcommand{\chaptermark}[1]{\markboth{\thechapter.\ #1}{}}
\renewcommand{\sectionmark}[1]{\markright{\thesection.\ \textit{#1}}}

\fancypagestyle{plain}{
    \fancyhf{}
    \fancyfoot[L]{\footnotesize <Course Title>}
    \fancyfoot[R]{\footnotesize Yujin Kang, MDIL}
    \fancyfoot[C]{\small \thepage}
}

% ToC-specific page style — header reads "Contents" on both sides
\fancypagestyle{tocstyle}{
    \fancyhf{}
    \fancyhead[L]{\small \contentsname}
    \fancyhead[R]{\small \contentsname}
    \fancyfoot[L]{\footnotesize <Course Title>}
    \fancyfoot[R]{\footnotesize Yujin Kang, MDIL}
    \fancyfoot[C]{\small \thepage}
    \renewcommand{\headrulewidth}{0.03em}
    \renewcommand{\footrulewidth}{0.03em}
}

\geometry{b5paper, landscape, top=0.05in, bottom=0.05in, left=0.2in, right=0.2in,
          headheight=1.25em, headsep=0.5em, footskip=1.75em, includehead, includefoot}

\wllusedatestyle

\DeclareMathSizes{10}{7}{7}{5}
\DeclareMathSizes{9}{7}{7}{5}
\DeclareMathSizes{8}{7}{7}{5}

\begin{document}
\maketitle

\newpage
\null\vfill
\begin{flushleft}
    {\large \textbf{Primary source:}} \\ \vspace{0.5em}
    {\large \textbf{Lecture notes from \hspace{0.3em} \Course}} \\
    \normalsize
    {\Semester \hspace{0.2em}} \\
    Lecturer: \ProfName (\ProfEmail), \ProfDepartment, \Institution \\
    {\small \textbf{\Name}} \\
    {\small Created: <Day, Nth, Month YYYY>} \\ \vspace{0.1em}
    {\small Last updated: \textbf{\DTMtoday}} \\ \vspace{0.1em}
\end{flushleft}
\vfill
\clearpage

% --- Table of Contents (with header sidebars + ToC-in-ToC entry) ---
\cleardoublepage
\addcontentsline{toc}{chapter}{\contentsname}
\pagestyle{tocstyle}
\begingroup
  \let\origthispagestyle\thispagestyle
  \renewcommand{\thispagestyle}[1]{\origthispagestyle{tocstyle}}
  \tableofcontents
\endgroup
\pagestyle{fancy}
\small

% --- Chapters ---
\input{chapters/01-<topic>}\clearpage
\input{chapters/02-<topic>}\clearpage
% ... add per syllabus

% --- Appendix ---
\appendix
\addtocontents{toc}{\protect\addvspace{1em}}
\addcontentsline{toc}{part}{Appendix}
\input{appendix/A-<topic>}\clearpage
\input{appendix/H-homework-solutions}\clearpage

\bibliographystyle{plainnat}
\bibliography{refs}

\printindex
\end{document}
```

**Why the `\let\origthispagestyle\thispagestyle` hijack:** `\tableofcontents` internally calls `\chapter*{Contents}` which fires `\thispagestyle{plain}`. Our default `plain` has no header. Hijacking `\thispagestyle` for the duration of `\tableofcontents` forces it to use `tocstyle` (which does have header) instead. The `\begingroup`/`\endgroup` auto-restores after.

### `solo.tex` template

```latex
% solo.tex --- single-chapter build driver.
% Usage: ./solo.sh <stem>     (jobname = stem; output = build/<stem>.pdf)

\documentclass[10pt,oneside]{book}
\input{preamble}

\title{Solo build: \jobname}
\author{Yujin Kang}

\usepackage{fancyhdr}
\pagestyle{fancy}
\fancyhf{}
\fancyhead[L]{\small \nouppercase\leftmark}
\fancyhead[R]{\small \nouppercase\rightmark}
\fancyfoot[L]{\footnotesize <Course Title> --- solo: \texttt{\jobname}}
\fancyfoot[R]{\footnotesize Yujin Kang, MDIL}
\fancyfoot[C]{\small \thepage}
\renewcommand{\footrulewidth}{0.03em}
\renewcommand{\headrulewidth}{0.03em}
\renewcommand{\chaptermark}[1]{\markboth{\thechapter.\ #1}{}}
\renewcommand{\sectionmark}[1]{\markright{\thesection.\ \textit{#1}}}

\geometry{b5paper, landscape, top=0.05in, bottom=0.05in, left=0.2in, right=0.2in,
          headheight=1.25em, headsep=0.5em, footskip=1.75em, includehead, includefoot}

\wllusedatestyle
\DeclareMathSizes{10}{7}{7}{5}
\DeclareMathSizes{9}{7}{7}{5}
\DeclareMathSizes{8}{7}{7}{5}

\begin{document}
\IfFileExists{chapters/\jobname.tex}
  {\input{chapters/\jobname}}
  {\IfFileExists{appendix/\jobname.tex}
    {\appendix\input{appendix/\jobname}}
    {\errmessage{No chapter or appendix matches jobname=\jobname}}}
\end{document}
```

### `solo.sh` template

```bash
#!/usr/bin/env bash
# solo.sh --- quick single-chapter build helper.
# Usage: ./solo.sh <stem>
#   <stem> is the basename (no extension) of a file under chapters/ or appendix/.
#   Output: build/<stem>.pdf

set -euo pipefail

if [ $# -lt 1 ]; then
    echo "usage: $0 <chapter-or-appendix-stem>" >&2
    exit 1
fi

STEM="$1"
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" &> /dev/null && pwd)"
cd "$SCRIPT_DIR"

if [ ! -f "chapters/${STEM}.tex" ] && [ ! -f "appendix/${STEM}.tex" ]; then
    echo "error: no chapters/${STEM}.tex or appendix/${STEM}.tex" >&2
    exit 2
fi

mkdir -p build
rm -rf build/$STEM.*

# -f: keep going past missing cross-refs (expected in solo build).
TEXINPUTS=$HOME/.config/tex//: latexmk -pdf -f -output-directory=build \
    -interaction=nonstopmode -jobname="$STEM" solo.tex || true

PDF="build/${STEM}.pdf"
if [ -f "$PDF" ]; then
    open -a Skim "$PDF" 2>/dev/null || open "$PDF"
else
    echo "error: build failed, no $PDF produced" >&2
    exit 3
fi

# Live preview: -pvc rebuilds on file change.
TEXINPUTS=$HOME/.config/tex//: latexmk -pdf -pvc -f -output-directory=build \
    -interaction=nonstopmode -jobname="$STEM" solo.tex || true
```

---

## Phase B — fill rules

One chapter per trigger. Read the source PDF before writing.

### Transcription contract

- **Header:** every chapter file starts with `% Source: <pdf-name>`.
- **Completeness:** every word, letter, equation, table heading, bullet point in the source slides must appear in the TeX output. Zero omissions.
- **Rewriting allowed:** format may change freely. Use `\begin{wlldash}`, `\begin{wllbul}`, `\begin{enumerate}`, `\begin{tabular}`, `\begin{equation}`, paragraphs — whichever fits the slide best.
- **Facts exact:** equations, derivations, definitions, named theorems, parameter values, method names, citations — character-for-character faithful.
- **No insertion:** transcription, not interpretation. Don't add knowledge beyond what the PDF contains.
- **Available macros:** see [`wlltex0-cheatsheet.md`](./wlltex0-cheatsheet.md).

### Two-column chapter structure

```latex
\chapter{Chapter Title}
\label{ch:slug}
% Source: lecture-slide-filename.pdf

\begin{multicols}{2}
\raggedcolumns

% all section/subsection/figure content here

\end{multicols}
```

- `\raggedcolumns` keeps columns from auto-stretching to equalize heights (recommended; comp.chem uses it; qchem ch01 doesn't — both compile fine).
- Figures stay inside multicols. Wide tables break out (see below).

### Figure extraction (vision)

- Use the vision model to read each slide image from the source PDF.
- **Every figure** in the lecture notes must be extracted and included.
- Workflow per figure:
  1. Identify the figure on the slide (use vision to isolate it).
  2. Save extracted image as `figures/lec<NN>/p<M>.png` (matches comp.chem convention; descriptive names like `figures/lec02/eigenfunctions-and-collapse.png` also acceptable, used in qchem).
  3. Include in TeX using width and layout rules below.

### Figure width decision

Figures stay **inside** `multicols`. `\linewidth` inside multicol = one column width.

| Content type | Width | When |
|---|---|---|
| Small icon / simple schematic | `0.3–0.5\linewidth` | Tiny diagram, arrow diagram, symbol |
| Medium diagram | `0.7\linewidth` | Moderate detail |
| Standard figure | `0.85–0.9\linewidth` | Most slide figures — default |
| Dense / detailed figure | `0.95\linewidth` | Lots of labels, crowded content |

Default to `0.9\linewidth` when unsure. Never use `1.0\linewidth` — leaves no breathing room.

### Single figure (standard)

```latex
\begin{figure}[H]
  \centering
  \includegraphics[width=0.9\linewidth]{figures/lec01/p03.png}
  \caption{<caption from slide, or descriptive if none>}
  \label{fig:lec01-p03}
\end{figure}
```

### Multi-panel slide — subfigure side-by-side

When a slide has 2–3 related panels: extract each panel separately, compose with `subcaption` (already loaded via wlltex0).

```latex
\begin{figure}[H]
  \centering
  \begin{subfigure}[t]{0.48\linewidth}
    \centering
    \includegraphics[width=\linewidth]{figures/lec01/p05a.png}
    \caption{Left panel}
    \label{fig:lec01-p05a}
  \end{subfigure}
  \hfill
  \begin{subfigure}[t]{0.48\linewidth}
    \centering
    \includegraphics[width=\linewidth]{figures/lec01/p05b.png}
    \caption{Right panel}
    \label{fig:lec01-p05b}
  \end{subfigure}
  \caption{Overall slide caption.}
  \label{fig:lec01-p05}
\end{figure}
```

For 3 panels use `0.31\linewidth` each with `\hfill` between.

### Small inline diagram — wrapfig (untested inside multicol; test before relying)

`wrapfig` inside `multicol` is a known-fragile combination — `wrapfigure` can overlap with column boundaries, get bumped to next page, or break layout in non-obvious ways. **Pattern below has not been tested under this skill's exact preamble.** If you use it, render the affected page immediately to check; fall back to standard `\begin{figure}[H]` if layout breaks.

```latex
\begin{wrapfigure}{r}{0.4\linewidth}
  \centering
  \includegraphics[width=\linewidth]{figures/lec01/p07.png}
  \caption{Short caption.}
  \label{fig:lec01-p07}
\end{wrapfigure}
```

`{r}` = right-placed, `{l}` = left. Width `0.35–0.45\linewidth` if you try it. Requires `\usepackage{wrapfig}` in `preamble.tex` (already in the template above).

**Do not use wrapfig for:** diagrams with many labels; figures the reader needs to study carefully; any figure wider than `0.5\linewidth`; first or last paragraph of a section; near a column break.

If wrapfig misbehaves: replace with standard figure[H] at appropriate width.

### Wide tables — the only case for breaking out of multicols

Figures **never** break out. Only genuinely full-page-wide **tables** do:

```latex
\end{multicols}

\begin{table}[H]
\centering
\footnotesize
\adjustbox{max width=\linewidth}{\begin{tabular}{...}
...
\end{tabular}}
\caption{...}
\label{tab:...}
\end{table}

\begin{multicols}{2}
\raggedcolumns
```

- `\footnotesize` inside every table — always, regardless of content.
- `\adjustbox{max width=\linewidth}` prevents overflow.
- Resume `\begin{multicols}{2}\raggedcolumns` immediately after `\end{table}`.

### Build verification after each chapter

```bash
./solo.sh <stem>    # e.g. ./solo.sh 04-dft
```

Must compile without errors before declaring chapter done. If build fails: report the error to user verbatim, do not silently retry or hide it. Common failures: missing figure file, unmatched `\end{multicols}`, undefined wlltex0 macro (typo), unbalanced `$`/braces.

---

## Build commands

```bash
# Full book
TEXINPUTS=$HOME/.config/tex//: latexmk -pdf -output-directory=build -interaction=nonstopmode main.tex

# Single chapter
./solo.sh <stem>

# Nuke stale aux/toc/idx (formatting looks corrupted)
latexmk -C -output-directory=build && \
  TEXINPUTS=$HOME/.config/tex//: latexmk -pdf -output-directory=build -interaction=nonstopmode main.tex

# Inspect specific page
pdftoppm -f <n> -l <n> -r 100 build/main.pdf /tmp/out -png
```

`TEXINPUTS=$HOME/.config/tex//:` must be set for `wll*.sty` resolution. Verify with `kpsewhich wlltex0.sty` before first build.

---

## Known gotchas

- **iCloud paths:** contain spaces and `~`; always quote in shell commands.
- **Symlink + latexmk:** `-output-directory` on a symlinked path confuses latexmk. Keep `build/` inside the dotfile target — current layout is safe.
- **TEXINPUTS:** if `kpsewhich wlltex0.sty` returns empty, `wll*.sty` won't resolve. Export or prepend to every latexmk call.
- **Stale aux/toc/idx:** poisoned state causes layout corruption on subsequent passes. Symptom: bizarre chapter/section spacing or missing headers. Fix: nuke with `latexmk -C` then rebuild.
- **`figure*` + multicol:** undefined inside `multicol`. Do not use it. Figures stay inside multicols at reduced width; only wide tables break out.
- **`lecturenotes` option geometry:** wlltex0's `lecturenotes` mode sets its own geometry. Explicit `\geometry{...}` in `main.tex` must come **after** `\usepackage{wlltex0}` to override. Verify geometry is b5+landscape in compiled PDF.
- **wrapfig + multicol:** see Phase B figure rules above. Untested in this preamble; test before relying.
- **`lecutre-sources` typo:** exists in some existing WDIR layouts. Propagate whichever spelling is already present; do not silently rename.
- **qchem predates the ToC-in-ToC rule:** qchem and comp.chem do not have the `\fancypagestyle{tocstyle}` + `\thispagestyle` hijack pattern. New books built from this skill will; old books can be retrofitted but it's optional.
