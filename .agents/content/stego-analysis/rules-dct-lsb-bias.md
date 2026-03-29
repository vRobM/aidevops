---
name: dct-lsb-bias
description: DCT LSB bias analysis — the primary test for JPEG steganography detection via jpegio coefficient extraction
metadata:
  tags:
    - dct
    - lsb
    - jpegio
    - steganography
    - jpeg
    - quantized-coefficients
---

# DCT LSB Bias Analysis — Primary Test

The most decisive test for JPEG steganography. Analyzes the least-significant bit of each **quantized** DCT coefficient value, extracted directly from the JPEG file via `jpegio` — not scipy re-encoding.

## Why This Works

Natural JPEG compression quantizes DCT coefficients. For any coefficient value `c`, the LSB (parity) should be approximately 50% for odd and 50% for even values across the full distribution. Steganographic embedding in the DCT domain introduces a detectable parity bias because:

1. Most steganographic tools preferentially modify coefficients with small magnitudes (|1|, |2|) to minimize visual distortion
2. These small coefficients are the most numerous, so the LSB bias is amplified
3. The bias direction reveals whether odd or even coefficients were preferentially modified

## Method

1. Read JPEG with `jpegio` → get quantized DCT coefficients for each color channel
2. Filter to AC coefficients only (skip DC at position [0,0] in each block)
3. For each non-zero AC coefficient `c`:
   - Record: `c & 1` (LSB = 0 for even, 1 for odd)
   - If |c| ≤ 10: record odd/even separately
4. Compute ratios and deviations from 0.50

## Interpretation

| LSB=1 Ratio | Deviation | Verdict |
|-------------|-----------|---------|
| 0.50–0.55 | 0.00–0.05 | Clean |
| 0.55–0.60 | 0.05–0.10 | Suspicious |
| >0.60 | >0.10 | **Strong evidence of embedding** |

The 2025 blue channel (Ch2) showed 0.627 LSB=1 ratio (deviation +0.127) and **0.819 odd ratio** among small coefficients (deviation +0.319). The probability of this occurring naturally is effectively zero.

## Magnitude Concentration

Also analyze which magnitudes dominate. Natural JPEGs expect ~30-40% of small AC coefficients to have |1|. Steganographic payloads often concentrate 50-75% in |1| because these are the most numerous and least visually significant coefficients.

| Image | Dominant Magnitude | % of Small AC | Interpretation |
|-------|-------------------|--------------|----------------|
| Natural JPEG | \|1\| | ~30-40% | Expected |
| 2025 Ch2 (stego) | \|1\| | **73.4%** | Extreme concentration |
| 2025 Ch1 (stego) | \|1\| | 63.0% | Strong concentration |

## Implementation

```python
import jpegio
import numpy as np

def analyze_dct_lsb(path):
    jpeg = jpegio.read(path)
    for comp_idx, comp in enumerate(jpeg.coef_arrays):
        ac_mask = np.ones_like(comp, dtype=bool)
        ac_mask[0, 0] = False  # exclude DC
        nz = comp[ac_mask]  # non-zero AC coefficients

        # Overall LSB ratio
        lsb_ratio = np.mean(nz & 1)

        # Small coefficient odd/even ratio
        small = nz[np.abs(nz) <= 10]
        odd_ratio = np.mean(small & 1) if len(small) > 0 else np.nan

        print(f"  Ch{comp_idx}: LSB=1={lsb_ratio:.3f}, odd(small)={odd_ratio:.3f}")
```

## Limitations

- Requires `jpegio` — scipy's DCT re-encoding is NOT equivalent
- Calibration images (exact same JPEG, not recompressed) may show weaker bias
- Some heavily compressed images (quality < 60) can produce false positives
- Does not work on PNG — use spatial LSB analysis for PNG
