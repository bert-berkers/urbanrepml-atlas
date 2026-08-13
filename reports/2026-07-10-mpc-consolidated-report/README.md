# Cone-Glimpse MPC — consolidated report (2026-07-10)

`mpc_lane_consolidated_report.pdf` (35 pages, 31 figures) is the single publication-grade
reference for the Cone-Glimpse MPC representation-learning direction. It opens with a terminology
table (§1) fixing one exact name per representation — a deliberate fix for the "trained vs.
untrained" confusion (the MPC architecture is EM-trained, the VAEs are gradient-trained, and the
best representation — the champion — is random-weights). Contents: the architecture and its H3
mapping; an input-modality gallery (AlphaEarth / hex2vec / highway2vec individually and combined);
the EM-training erosion result and the random-weights champion; the two experiments (entropy-guided
fixation sampling; amortized Gaussian/Poisson VAEs, with a Poisson-VAE collapse-mechanism
subsection); the probe-validation and dimensionality controls (random-projection ladder, fair
per-representation regularisation); a consolidated ranking; a synthesis; open scientific questions;
and a qualitative map atlas (Appendix B) for visual investigation. All evidence is the South
Holland res9 pilot (34,302 anchors), ridge probes under spatial-block 5-fold CV.

## Rebuild

From this directory, run XeLaTeX twice (fontspec + Aptos cloud-fonts require XeLaTeX):

```
xelatex -interaction=nonstopmode mpc_lane_consolidated_report.tex
xelatex -interaction=nonstopmode mpc_lane_consolidated_report.tex
```

## Figure regeneration

- Results figures (ranking, EM erosion, ladders, VAE diagnostics, probe-estimator fingerprint,
  dimensionality controls): `scripts/stage3/mpc_lane_report_figures.py`
- Methodology + Poisson-VAE mechanism figures (entropy field, cone-sampling patch, pipeline
  schematic, `f_pvae_mechanism` / `f_pvae_latent_activity` / `f_pvae_training`):
  `scripts/stage2/entropy_sampling_methodology_figures.py`
- Qualitative atlas + input-modality gallery (grids + zooms + 68 standalone maps under
  `figures/atlas/`): `scripts/stage3/mpc_atlas_figures.py`
- Dimensionality-control data + probe-validation canaries:
  `data/study_areas/south_holland/stage3_analysis/matrix_2026-07-10_dimcontrols/` and
  `scripts/one_off/2026-07-10-probe-canaries.py`
- Reused figures (windmill composites, signal-concentration, PC-RGB panel) are copied in from the
  2026-07-04 methodology and 2026-07-05/07 report figure suites.

Companion documents: the methodology note (`reports/2026-07-04-mpc-fovea-options/`) carries the
full mathematics; the South Holland benchmark (`reports/2026-07-05-mpc-cone-glimpse-southholland/`)
carries the training control ladder.
