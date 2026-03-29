---
name: stego-attacks
description: Standard steganalysis attacks — RS, SPA, Weighted Stego, Calibration Chi-Square, and Aletheia reference implementation
metadata:
  tags:
    - steganography
    - rs-analysis
    - spa
    - calibration
    - aletheia
    - fridrich
    - lsb
---

# Steganalysis Attack Reference

Standard steganalysis attacks for JPEG and spatial images. Reference implementation: Aletheia framework (`work/roemmele-stego/aletheia/aletheialib/attacks.py`).

## RS Attack (Fridrich et al.)

**Principle:** Examine 2×2 pixel blocks. Compute smoothness under +1 and -1 LSB flips. Clean images show symmetric RS statistics. LSB embedding breaks the symmetry.

**Formula:**
```
RS_stat = (R_pos - S_pos)/N - (R_neg - S_neg)/N
```
- Clean image: RS_stat ≈ 0
- Embedded: RS_stat ≠ 0

**Strengths:** Effective on spatial (uncompressed) images; high sensitivity to LSB embedding.
**Weaknesses:** Can produce false positives on heavily compressed JPEGs; does not detect DCT-domain embedding directly.

**Implementation:** `analyzer_v2.rs_attack(path, channel=0)`

## SPA — Sample Pairs Analysis (Dumitrescu et al.)

**Principle:** Pairs adjacent pixels; categorizes by LSB of second pixel and MSB of first. Embedding distorts the natural co-occurrence patterns. Solve quadratic for embedding rate alpha.

**Formula:**
```
2k * beta^2 + 2(2x - 2k) * beta + (y - x) = 0
```
- `alpha = 2 * beta` (embedding rate)
- `beta` (relative payload)

**Strengths:** Precise embedding rate estimation for spatial images.
**Weaknesses:** Less effective on heavily compressed JPEGs; requires large image areas.

**Implementation:** `analyzer_v2.spa_attack(path, channel=0)`

## Weighted Stego Attack (Fridrich & Goljan)

**Principle:** Compute weights as inverse of local variance. Clean images have weighted LSB sum ≈ 0.25 (uniform random). Embedding biases the sum proportionally to embedding rate.

**Formula:**
```
weighted_lsb_sum = sum(weight[i,j] * (pixel[i,j] & 1)) / sum(weights)
```
- Clean: ≈ 0.25
- Embedded: deviates proportionally

**Strengths:** Works on any spatial image; no assumptions about image content.
**Weaknesses:** Computationally expensive (per-pixel variance computation); less sensitive than RS.

**Implementation:** `analyzer_v2.weighted_stego_attack(path, channel=0)`

## Calibration Chi-Square (F5 Detection)

**Principle:** Crop image by 4px, recompress as JPEG (simulating original encoding path). Compare DCT coefficient distributions. F5 encoding deliberately minimizes histogram changes → calibrated reference matches too well → low chi-square.

**Method:**
1. Crop by 4 pixels in both dimensions (removes edge 8×8 blocks)
2. Recompress as JPEG (quality ≈ original)
3. Extract DCT coefficients from both original and calibrated
4. Compute chi-square per coefficient group
5. Average: low score = F5-like encoding

**Interpretation:**
- avg χ² < 5: **F5-like** (possible F5/matrix encoding)
- avg χ² 5-20: inconclusive
- avg χ² > 20: natural or non-F5

**Requirements:** ImageMagick (`convert`) + jpegio

**Implementation:** `analyzer_v2.calibration_chi_square(path)`

## Aletheia Reference Implementation

Aletheia provides ML-based ensemble classifiers combining SRM features with neural networks:

| Model | Features | Best For |
|-------|---------|---------|
| **Ensemble** | SRM + machine learning | General JPEG steganalysis |
| **DCTR** | DCT residual features | JPEG尤其是 F5 |
| **GFR** | Gabor-filtered DCT | High-quality JPEGs |
| **SRNet** | Deep neural network | Binary classification |

To use Aletheia:
```bash
cd work/roemmele-stego/aletheia
# Run Aletheia CLI for a specific image
python3 -m aletheia estimate image.jpg
```

## Combined Interpretation

Never rely on a single test. The strongest evidence comes from **multiple tests agreeing**:

| Confidence | Pattern |
|-----------|---------|
| **CONFIRMED** | DCT LSB bias + RS + Weighted Stego all suspicious |
| **LIKELY** | DCT LSB bias suspicious + one spatial test suspicious |
| **POSSIBLE** | Only one test suspicious, others clean |
| **CLEAN** | All tests within expected ranges |
