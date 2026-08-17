# CLAUDE.md — thesis

LaTeX master-thesis document for the MSc in Artificial Intelligence @ UNIBO (Overleaf git remote). Title: *Detecting Face-Swapped Images with Multimodal Large Language Models*. The experiments, results, figures, and numbers it reports come from the sibling [`faceswap_detection/`](../faceswap_detection/CLAUDE.md) project.

## Build

The **actual** build is **pdfLaTeX + Biber** driven by `latexmk`, configured in `.vscode/settings.json` (LaTeX Workshop, builds on save, default recipe "latexmk (pdfLaTeX + biber)"):

```bash
latexmk -pdf -interaction=nonstopmode -synctex=1 -file-line-error main.tex
```

> Note the discrepancy: `README.md` repeats the generic UNIBO template advice ("compile with XeLaTeX + Biber, add `\setmainfont{Times New Roman}`"), but this project does **not** do that. It uses `ai_bo_thesis.sty`, which loads Times-like `newtx` fonts for pdfLaTeX, so no `fontspec`/`\setmainfont` is needed. Build with pdfLaTeX. Don't "fix" it to XeLaTeX.

TeX Live 2025 lives at `/usr/local/texlive/2025/bin/universal-darwin`, which is **not** on the PATH for a Dock-launched VS Code — the `.vscode` config injects it per tool. If running `latexmk`/`biber` from a shell, make sure that bin dir is on PATH.

`latexmk` auto-cleans aux files (`.aux .bbl .bcf .blg .lof .lot .out .toc .run.xml .synctex.gz .fls .fdb_latexmk`) on build but keeps `main.pdf`. Don't commit those aux artifacts.

## Layout

- `main.tex` — root document (`\documentclass[a4paper,oneside,12pt]{report}`; single-side for digital submission — a commented `twoside` line is for printing). Holds all preamble, title-page metadata (`\title`, `\candidate`, `\supervisor`, `\academicyear`, …), and `\input`s each chapter in order.
- `chapters/` — one `.tex` per part, `\input` from `main.tex` in this order: `abstract`, then `introduction`, `background`, `related_work`, `methodology`, `experimental_setup`, `results`, `conclusion`, `appendix`, `acknowledgements`. (`sommario` — Italian abstract — exists as an optional, currently-commented block.)
- `ai_bo_thesis.sty` — the department style (adapted from the Stanford thesis style). Defines `\frontispiece`, `\toc`, `\figstoc`, `\tablestoc`, `\begintext`, `\appendix`, `\acknowledgements`, and the title-page macros. Treat as vendored template — edit chapters, not this.
- `biblio.bib` — Biber/biblatex bibliography. Style: `[backend=biber, style=trad-plain, sorting=nty, giveninits=true]`.

## Conventions

- **Cross-references**: use `cleveref`'s `\cref{...}` (configured `capitalise,noabbrev`) so refs render as "Chapter 3", "Figure 2.1" automatically — don't hand-write "Chapter~\ref{...}".
- **Citations**: biblatex `\cite`/`\parencite`/`\textcite`; bibliography is printed via `\printbibliography[heading=bibintoc]`.
- **`\guide{...}`**: a custom macro (defined in `main.tex`) that renders *italic bracketed writing advice* about what to put in a section. It is scaffolding — replace each `\guide{...}` with real prose and delete the call. To hide all guidance for a near-final draft, redefine it to a no-op: `\renewcommand{\guide}[1]{}`.
- **Tables/figures/math**: `booktabs` for rules, `siunitx` (`\num`, `\SI`) for aligned numbers, `subcaption` for side-by-side figures, `listings` for prompt/code snippets. `\figstoc`/`\tablestoc` in `main.tex` are enabled — keep them only while figures/tables exist.
- `hyperref` is set `hidelinks` for a clean printed look.
