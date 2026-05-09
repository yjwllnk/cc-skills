# wlltex0.sty cheatsheet

Quick reference for macros and environments provided by `~/.config/tex/wlltex0.sty`, `wllMDIL.sty`. Use these instead of rolling custom equivalents.

Canonical source: `~/.config/tex/wlltex0.sty` — read it for the full picture.

---

## Package load (in `preamble.tex`)

```latex
\usepackage[tikz,lecturenotes,physics]{wlltex0}
\usepackage{wllMDIL}
\usepackage{wll<SHORT>}
```

Options:
- `tikz` — load TikZ (heavy; only when needed)
- `lecturenotes` — B5 landscape geometry + multicol setup + tight spacing
- `physics` — physics package: `\dv`, `\pdv`, `\bra`, `\ket`, `\norm`, `\comm`, `\anticomm`
- `chem` (default true) — mhchem
- `code` (default true) — listings + `bashcode`/`pycode`/`Ccode`/`stdout` envs
- `figures` (default true) — graphicx + caption + subcaption
- `tables` (default true) — booktabs + tabularx + array + multirow

---

## List environments

| Env | Style |
|-----|-------|
| `wlldash` | `--` bullet |
| `wllbul` | `•` bullet |
| `wllnum` | `1.` numbered |
| `wllalph` | `(a)` lettered |
| `wllalphH` | `(A)` uppercase |
| `wllrome` | `i.` roman |
| `wllromeH` | `i)` roman with paren |
| `wllRome` | `I.` Roman uppercase |

Standard `itemize`/`enumerate` are also pre-tightened by wlltex0 (small leftmargin, itemsep 0.15em, no parsep).

---

## Highlighting

```latex
\highlight{text}                  % yellow background, default
\highlight[red!20]{important}     % custom color
\wllhl[textcolor][bgcolor]{text}  % full control (xparse)
```

---

## Math helpers

```latex
\diff          % upright d for integrals: \int f \diff x
\ee            % upright e (Euler's number)
\ii            % upright i (imaginary unit)
\vect{v}       % bold vector (uses \bm)
\mat{A}        % bold matrix
\avg{O}        % <O> angle brackets, auto-sized
\absval{x}     % |x|, auto-sized
\normval{v}    % ||v||, auto-sized
```

From `physics` package option:
```latex
\dv{f}{x}      % derivative
\pdv{f}{x}     % partial derivative
\bra{\psi}     % <psi|
\ket{\psi}     % |psi>
\braket{a}{b}  % <a|b>
\norm{v}       % ||v||
\comm{A}{B}    % [A,B] commutator
\anticomm{A}{B} % {A,B} anticommutator
\Tr            % trace
\Re, \Im       % real / imag (overridden by physics pkg)
```

---

## Code blocks (always available; `code` option default on)

```latex
\begin{bashcode}
echo hello
\end{bashcode}

\begin{pycode}
import numpy as np
\end{pycode}

\begin{Ccode}
int main() { return 0; }
\end{Ccode}

\begin{stdout}
output here
\end{stdout}
```

---

## Tables (always available; `tables` option default on)

`booktabs`, `tabularx`, `array`, `multirow` loaded. Custom column type:

```latex
\begin{tabularx}{\linewidth}{Y l r}
% Y = ragged-right, X-flexible width
\end{tabularx}
```

For wide tables that overflow, wrap in `\adjustbox` (loaded via `adjustbox` in `preamble.tex`):

```latex
\begin{table}[H]
\centering
\footnotesize
\adjustbox{max width=\linewidth}{\begin{tabular}{...}
...
\end{tabular}}
\caption{...}
\end{table}
```

---

## Colors

Predefined: `wllnavy` (#0B1F33), `wllblue` (#0088FF), `wllgray` (#5A6B7A), `seagull`, `redtwo`, `hgreen`, `vgreen`, `navy`, `cblue`.

Hyperref colors (when `links` option on): `linkcolor=wllblue`, `citecolor=wllblue`, `urlcolor=wllblue`.

---

## Date style

```latex
\wllusedatestyle      % activates wlltex0style: "9th, May, 2026. Fri."
\DTMtoday             % current date in active style
```

---

## wllMDIL.sty constants

Personal identity macros (loaded via `\usepackage{wllMDIL}`):

```latex
\Name             % Yujin Kang
\Lab              % MDIL
\Department       % Department of Materials Engineering and Science
\StudentNumber    % 2024-25818
\EmailPrimary     % yjkang@snu.ac.kr
\EmailSecondary   % yjwlln.kang@gmail.com
\AffiliationOne   % MS Student, MSE
\AffiliationTwo   % College of Engineering, SNU
\Address          % full address
\ORCid            % 0009-0005-4104-7635
\Mobile           % +82 10-3859-8353
\GitHubURL, \LinkedinURL, \ScholarURL, \HomepageURL
\GitHubUser       % yjwllnk
\LinkedinUser     % yjwllnkang
```

---

## wll<SHORT>.sty (per-book)

Defined per subject. Standard fields:

```latex
\Course           % e.g. "Quantum Chemistry"
\Institution      % e.g. "SNU"
\Semester         % e.g. "2026 Spring"
\ProfDepartment
\ProfName
\ProfHomepage
\ProfEmail
\Category         % e.g. "Notes"
\SubCategory      % e.g. "QuantumChemistry"
```

---

## Cross-reference (cleveref)

`cleveref` loaded via `[links]` option. Use `\cref{label}` instead of `\ref{label}`:

```latex
\cref{eq:schrodinger}    % "eq. (1.2)"
\cref{fig:lec01-p03}     % "fig. 1.3"
\cref{ch:postulates}     % "ch. 1"
```

`cleveref` capitalizes at sentence start: `\Cref{...}`.

---

## When in doubt

Read `~/.config/tex/wlltex0.sty` directly — it's well-commented and exhaustive. The code blocks in there show exactly what each option enables.
