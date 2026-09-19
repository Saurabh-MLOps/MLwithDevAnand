# Project 2 — Image Compression with the SVD: Report

Three greyscale images (luma = 0.299 R + 0.587 G + 0.114 B, via Pillow `convert("L")`):

| Image | Character | Shape (m×n) | Numbers (mn) |
|---|---|---|---|
| Kolkata | smooth: large sky and lake bands, foliage between | 987 × 1080 | 1,065,960 |
| Inspiration | detailed: collage of ~13 photos with text, faces, hard edges | 915 × 1360 | 1,244,400 |
| Meri Photo | my choice: portrait, foliage, granite, patterned carpet | 3000 × 2250 | 6,750,000 |

## 1. What the SVD produces, and what singular values mean

The SVD writes an image matrix as **A = U Σ Vᵀ**, which is a sum of rank-1 layers σ₁u₁v₁ᵀ + σ₂u₂v₂ᵀ + …. Each layer is a column pattern times a row pattern, and σᵢ says how strongly it contributes. The layers are ordered by σ and are orthogonal, so they don't double-count, which gives ‖A − Aₖ‖²_F = Σᵢ>ₖ σᵢ². By the Eckart–Young theorem, keeping the first k layers is the best possible rank-k approximation. So σᵢ² is exactly the squared error you incur by discarding layer i.

## 2. Measurement method, and why the storage count is what it is

A rank-k approximation stores k columns of U (m numbers each), k rows of Vᵀ (n each) and k singular values: **k(m+n+1)** numbers, not k. The ratio is mn / [k(m+n+1)]. Past **k = mn/(m+n+1)** the factors outweigh the image; this break-even is 546.7 (Inspiration), 515.5 (Kolkata) and 1285.5 (Meri Photo).

Relative error is ‖A − Aₖ‖_F/‖A‖_F and energy is Σᵢ≤ₖ σᵢ²/Σσᵢ². The identity above means **error² = 1 − energy**; I verified it against a direct A − Aₖ computation at k = 20. I also tested the metric function on a 4×5 rank-1 matrix with hand-worked answers (10 stored, ratio 2.0, error 0), and confirmed the full-rank round trip to within 10⁻¹⁰.

A lesson: my first pass tested only seven hand-picked k values, so "smallest k with error ≤ 5%" was just the first large k I had tried. The final version evaluates every k.

## 3. Comparison across image types

Smallest k with relative error ≤ 5%:

| Image | k | k / min(m,n) | Numbers stored | Ratio |
|---|---|---|---|---|
| Kolkata | 144 | 0.146 | 297,792 | **3.58** |
| Meri Photo | 450 | 0.200 | 2,362,950 | 2.86 |
| Inspiration | 196 | 0.214 | 446,096 | 2.79 |

**Kolkata compresses best**, and the picture explains it. 55% of its pixels are nearly flat (gradient < 2), against 37% for Inspiration and 24% for Meri Photo; the sky's mean gradient is 0.34 against 13.3 for the bottom third. Sky and lake are horizontal bands whose rows are near-copies of each other, which is what a low-rank matrix is. Its σ₁ alone holds 93.7% of the energy, and 99% needs only k = 47 (66 for Inspiration, 188 for Meri Photo).

The collage is a patchwork: each tile has its own content, and text and faces add many small independent layers. Meri Photo has the most edge content (mean gradient 13.7) yet lands level with the collage. Its strongest edges are diagonal (window frames, planter sides, palm fronds), and a diagonal line has full rank, like the identity matrix. I didn't isolate this by experiment, so it is a likely explanation, not a proven one. Kolkata's spectrum also cliffs to about 10⁻¹¹ at the end: it has numerical rank 971 of 987 and 93 exactly duplicated columns, a trace of resizing, not of the scene.

**Noise.** I added Gaussian noise to Kolkata. Noise is unstructured, so it adds roughly equal energy to every singular value and lifts the whole tail (over 10× at i = 400 for σ = 50), flattening the spectrum onto a floor near σ(√m+√n), which is 1286 and 3214. Truncation does denoise, with the best k at or below where the noisy spectrum meets that floor:

| noise σ | PSNR noisy → truncated | best k | error noisy → truncated |
|---|---|---|---|
| 20 | 22.1 → 25.6 dB | 107 | 14.6% → 9.7% |
| 50 | 14.2 → 21.8 dB | 21 | 36.5% → 15.2% |

But it is a blunt denoiser: it can't restore detail below the noise level, and it treats sky and edges alike.

## 4. How I would choose k, and its limits

I'd use the **quality target** (smallest k with error ≤ a chosen value), checked visually. Energy thresholds and the error target are the same criterion in different units (5% error = 99.75% energy). The difference is the reference point: on the raw image σ₁² is 77–94% of the energy, because a large mean brightness is a rank-1 component (a constant matrix has rank 1). So "90% energy" needs only k = 1–4 and says nothing about picture content. Centring removes the offset, and the same thresholds become k = 21 / 25 / 83 (Inspiration / Kolkata / Meri Photo).

The **elbow** is unreliable here. The farthest-from-chord point on log σ gives 46, 19 and 63 at a 99.9% window, but 16–79, 9–29 and 25–108 across windows from 99% to 99.99%. There is no real knee, just a smooth slope.

Limits of the quality target: 5% is arbitrary; Frobenius error is global, so smooth areas dominate it and it can hide ringing at edges; and it is measured against the uncentred norm. Chroma shows this: Cb and Cr sit near 128 with a standard deviation of 12.7, so a raw 5% target needs k = 4–6, but k = 81–83 once centred.

## 5. When SVD compression is a bad idea

**When storage is measured in bytes, or against JPEG.** My ratios count numbers, but a pixel is 1 byte and an SVD factor is a float. Kolkata at k = 144 is 1.19 MB as float32, *more* than the 1.07 MB raw image. JPEG at 4.6% error (quality 20) is 56.6 kB, about 20× smaller. (My sources were already JPEGs, which flatters JPEG somewhat.) JPEG needs no per-image basis, since its DCT is fixed; it works on 8×8 blocks, then quantises and entropy-codes.

The same cost explains why 8×8 block SVD lost to whole-image SVD at equal storage: 3.79% against 1.95% error on Kolkata, 3.98% against 3.43% on Inspiration. Each block pays 17 numbers for its own rank-1 basis, capping the ratio at 3.76. Other bad cases are diagonal edges, fine texture and noise, which all need large k.

**What it is good for.** The SVD is adaptive: it finds the provably best rank-k basis for this specific matrix, which is the idea behind PCA. Colour shows it. Separate R, G, B needed k = 192, 197, 202 (1.35M numbers); YCbCr with k = (220, 20, 20) hit the same 5.0% RGB-space error with 592k, about 2.3× fewer, because chroma is smooth. Randomised SVD matched the full SVD's error (0.1347 against 0.1346, k = 100) about 7× faster on Colab.
