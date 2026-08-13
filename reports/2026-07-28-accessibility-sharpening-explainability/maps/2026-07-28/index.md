# Visual battery -- 2026-07-28-accessibility-sharpening-explainability (VIS-1 / VIS-2 / VIS-3)

The pre-registered **PRIMARY practical-significance channel** (`../../PREREGISTRATION.md` §Governance clause 8, foundation F4): there is no numeric SESOI, so the human's reading of these maps is the verdict on whether any difference matters. No statistics appear on these pages.

- **Frame**: full 482,706-hexagon embedding-only frame, H3 res9, EPSG:28992 (Dutch RD New). No target join, so no join-scope caveat applies.
- **Seed**: 42 only (as registered for VIS-1/2/3).
- **Rasterisation**: `stage3_analysis.render.aggregate` datashader polygon core at 2000x2400 per panel (full per-panel resolution, never divided by panel count).
- **South Holland zoom bbox** (EPSG:28992 minx,miny,maxx,maxy): `(48120.0, 407743.0, 130564.0, 482534.0)` -- centroid-within `data/study_areas/south_holland/area_gdf/south_holland_boundary.parquet`, the 2026-07-18 battery's derivation.
- **Colour limits** are computed once on the full-Netherlands frame and reused for the zoom, so the two panels on a page are directly comparable.
- **Local-contrast column**: `ring1_cosine_distance`.
- Every PNG carries a sibling `*.provenance.yaml`.

## What you are asked to judge (registered in advance)

1. *Does the accessibility residual land where accessibility should land* -- along corridors, around stations, across the Randstad -- *or is it diffuse?* (VIS-1, VIS-2)
2. *Does the local-contrast difference map show a spatially interpretable pattern, or speckle?* (VIS-3)

## Stop rules (propagated automatically from the residual lane)

No stop rule fired on any rendered row; every row has all three pages. Verified by reading each row's `map_r2.json` `stop_rules_fired` field — see `halts.json`.

## Pages

### Plain U-Net
- `plain_vis1_residnorm.png` -- VIS-1 residual norm ||R|| (magma, p2-p98)
- `plain_vis2_residpcrgb.png` -- VIS-2 residual PC-RGB (rank-normalised PC1/PC2/PC3)
- `plain_vis3_dcontrast.png` -- VIS-3 Δ local contrast (acc - lattice, RdBu_r centred at 0)

### Gaussian-VAE  **COLLAPSE-CAVEAT**
- `gvae_vis1_residnorm.png` -- VIS-1 residual norm ||R|| (magma, p2-p98)
- `gvae_vis2_residpcrgb.png` -- VIS-2 residual PC-RGB (rank-normalised PC1/PC2/PC3)
- `gvae_vis3_dcontrast.png` -- VIS-3 Δ local contrast (acc - lattice, RdBu_r centred at 0)

### Poisson-VAE
- `pvae_vis1_residnorm.png` -- VIS-1 residual norm ||R|| (magma, p2-p98)
- `pvae_vis2_residpcrgb.png` -- VIS-2 residual PC-RGB (rank-normalised PC1/PC2/PC3)
- `pvae_vis3_dcontrast.png` -- VIS-3 Δ local contrast (acc - lattice, RdBu_r centred at 0)

### NAG-S0 (orthogonal core)
- `nag_s0_vis1_residnorm.png` -- VIS-1 residual norm ||R|| (magma, p2-p98)
- `nag_s0_vis2_residpcrgb.png` -- VIS-2 residual PC-RGB (rank-normalised PC1/PC2/PC3)
- `nag_s0_vis3_dcontrast.png` -- VIS-3 Δ local contrast (acc - lattice, RdBu_r centred at 0)

### NAG-S1 (paper-faithful)
- `nag_s1_vis1_residnorm.png` -- VIS-1 residual norm ||R|| (magma, p2-p98)
- `nag_s1_vis2_residpcrgb.png` -- VIS-2 residual PC-RGB (rank-normalised PC1/PC2/PC3)
- `nag_s1_vis3_dcontrast.png` -- VIS-3 Δ local contrast (acc - lattice, RdBu_r centred at 0)

## Provenance

Builder: `scripts/one_off/2026-07-28-acc-residual-map-battery.py`. Regenerated from the shared render primitives -- no rendered artifact was copied from a prior campaign (`.claude/rules/viz-discipline.md` §Figure-creation, D62).
