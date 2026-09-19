# Project 2 — Image Compression with the SVD

Truncated-SVD compression of three greyscale images, with an honest storage count (`k(m+n+1)`, not `k`), three data-driven ways of choosing `k`, a noise experiment, and stretch work on colour, 8×8 blocks and randomised SVD. The written analysis is in [`REPORT.md`](REPORT.md).

## Contents

```text
week02/
├── project2_svd.ipynb    all code, plots and results (run top to bottom)
├── images/               source images (~1.6 MB total)
│   ├── kolkata.jpeg
│   ├── Inspiration.jpeg
│   └── Meri_photo.jpeg
├── REPORT.md             written analysis (~1,150 words)
└── README.md             this file
```

## How to run

**Requirements:** Python 3.9+, `numpy`, `pandas`, `matplotlib`, `pillow`, `scikit-learn` (only for `randomized_svd`), `jupyter`. If the notebook still uses `skimage.color.rgb2ycbcr` for the YCbCr section, `scikit-image` is needed too.

```bash
pip install numpy pandas matplotlib pillow scikit-learn jupyter
cd week02
jupyter lab project2_svd.ipynb      # then: Kernel → Restart Kernel and Run All Cells
```

- Run from inside `week02/`. All paths are relative (`images/...`); nothing depends on Colab's `/content/`.
- **Runtime:** about a minute in total. The largest cost is the full SVD of the 3000×2250 image, roughly 6–17 s depending on the machine (it also gets timed in the randomised-SVD comparison).
- **Reproducibility:** `np.random.seed(42)` at the top. The noise experiment uses fixed `default_rng` seeds and `randomized_svd` uses `random_state=42`, so re-running gives the same numbers.

## Image sources and licences

| File | Character | Source | Licence / permission |
|---|---|---|---|
| `kolkata.jpeg` | smooth: sky, lake, foliage, tiles | **TODO — confirm:** own photograph? Add place and date | **TODO** — e.g. "own work, all rights reserved; used here for coursework" |
| `Meri_photo.jpeg` | textured: portrait, plants, granite, patterned carpet | **TODO — confirm:** own photograph? Add place and date | **TODO** — same as above |
| `Inspiration.jpeg` | detailed: collage of about 13 photos, quotes and diagrams | A personal mood-board collage assembled from images found online. The individual images are not my own work and I have not traced their original sources | **Not established.** Used privately for coursework only; it is not covered by any licence I can state |

Notes for the marker:

- Two of the three images are my own photographs. The collage is the exception, and I am stating that plainly rather than claiming a licence. If it must be replaced, any image of similar detail (own photo, or a CC0 image with its source URL added here) will run through the notebook unchanged. Only the numbers in `REPORT.md` would change.
- The photographs include a person's face. If this repository is public, consider whether you are comfortable with that.
- All images are used as greyscale float arrays (`Image.convert("L")`, luma weights 0.299 / 0.587 / 0.114). The colour stretch section uses `Inspiration.jpeg` in RGB and YCbCr.

## Headline results (5% relative-error target)

| Image | Shape | k | Numbers stored | Compression ratio | Break-even k |
|---|---|---|---|---|---|
| Kolkata | 987×1080 | 144 | 297,792 | 3.58 | 515.5 |
| Meri Photo | 3000×2250 | 450 | 2,362,950 | 2.86 | 1285.5 |
| Inspiration | 915×1360 | 196 | 446,096 | 2.79 | 546.7 |

Compression ratio is `mn / (k(m+n+1))`. See `REPORT.md` for the byte-level caveat (SVD factors are floats, so in bytes this is worse than the ratio suggests) and the JPEG comparison.
