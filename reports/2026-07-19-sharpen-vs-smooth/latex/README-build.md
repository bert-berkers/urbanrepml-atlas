<!--
  BUILD-NOTE for reports/2026-07-19-sharpen-vs-smooth/latex (pre-registration-science-lane, report layer §(g)).
  PART 2 of the smoothing story (Part 1: ../../2026-07-18-smooth-vs-learn-res10/).
  This README convention IS the build system — there is deliberately NO build script (anti-bloat).
  Derived from: specs/templates/README-build.template.md.

  STATUS (2026-07-19 late evening): report.tex is a RESULTS-INDEPENDENT draft. Front-half
  (title/abstract/related-works/methods) complete; Results/Discussion/Provenance are PENDING stubs
  filled from the frozen on-disk cells after the campaign chain completes. Do NOT build the final
  PDF until results land; a compile-in-principle check is fine (needs Aptos or a swapped mainfont).
-->

# Build

Compile from this dir (run **twice** for refs; **XeLaTeX** required — the document uses `fontspec`
with the Aptos / Aptos Display cloud fonts):

```
xelatex -interaction=nonstopmode report.tex
xelatex -interaction=nonstopmode report.tex
```

Output: `report.tex` -> `report.pdf` (<NN> pp.). Copy the final PDF up one level to the canonical,
rebuildable `../report.pdf`. Figures resolve from `../figures/` via `\graphicspath`; every figure is
deliberately title-less, so the captions carry the full explanation.

**Fonts.** Aptos (main) + Aptos Display (headings) load by TTF filename from the local Microsoft 365
cloud-font cache (`C:/Users/.../FontCache/4/CloudFonts/`). On another machine, install Aptos
system-wide and replace the `Path=` options in the preamble, or swap `\setmainfont` for any humanist
sans — the compiled PDF embeds the glyphs. Mono is Consolas.

**Design tokens & the variable/index sidebar** (`\varsidebar` / `\vsym`) are house style; the
asymmetric geometry (wide right margin) holds the sidebars. Color carries meaning, not decoration.

**Regenerate the figures** (from the repository root — pre-build BEFORE compiling):

```
python -u scripts/stage2/sharpening_methodology_figures.py
```

The five composite methodology figures (`fig_kernel_geometry`, `fig_kernel_profiles`,
`fig_subtract_lowpass_story`, `fig_dose_strip`, `fig_target_gallery`) plus their sub-panels already
exist under `../figures/` with `*.provenance.yaml` sidecars (builder commit `c7df2d0`). Results-cell
figures (S-H1/S-H2/S-H3 evidence maps + dose-response curves) are added at assembly time.

Each figure writes a `*.provenance.yaml` sidecar recording its builder, source, and the derivation
of every annotated number. **No number in any figure is invented** (`specs/figure_pipeline.md`,
`.claude/rules/viz-discipline.md`).

**Visual QAQC.** `qaqc_render/` holds a `pdftoppm` page-render of the compiled PDF; rebuildable:

```
pdftoppm -png -r 100 report.pdf qaqc_render/p
```

**Integrity note.** The body reads as science; harness/process jargon is quarantined to the appendix,
which also carries the `scripts/science/order_gate.py` PASS/VIOLATION receipt and the frozen
pre-registration excerpt (`specs/preregistration_science_lane.md` §(g)).
