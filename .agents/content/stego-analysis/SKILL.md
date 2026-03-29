---
name: stego-analysis
description: JPEG/PNG steganography analysis workflow — DCT LSB bias, RS, SPA, WSA, calibration attacks, and Aletheia reference
mode: general
tags:
  - steganography
  - jpeg
  - dct
  - jpegio
  - security
  - forensics
---

# Steganography Analysis Skill

JPEG/PNG steganography analysis using DCT-domain and spatial-domain attacks. Primary tool: `analyzer_v2.py` in the `roemmele-stego` workspace. Reference implementation: Aletheia steganalysis framework.

## When to Use

Use when analyzing images suspected of containing steganographic payloads. Triggered by:
- "analyze image for steganography"
- "check if image has hidden data"
- "stego analysis"
- Any worktree under `work/roemmele-stego/`

## Prerequisites

```
# Python venv with dependencies
source ~/.aidevops/.agent-workspace/work/roemmele-stego/stego-venv/bin/activate

# ImageMagick (for calibration attack)
convert --version    # must be installed

# jpegio (for actual DCT coefficient extraction, NOT scipy re-encoding)
python -c "import jpegio; print('jpegio OK')"
```

If jpegio is missing: `uv pip install jpegio` inside the venv.

## Workflow

### 1. Collect Images

Download from Twitter/X via XCancel (Nitter frontend) to get original-resolution images:
```
https://xcancel.com/{username}/status/{id}/photo/1
```
Always use `?name=orig` for Twitter CDN URLs to get unprocessed originals.

### 2. Run Full Analysis

```
cd ~/.aidevops/.agent-workspace/work/roemmele-stego/
source stego-venv/bin/activate
python3 analyzer_v2.py
```

### 3. Run Individual Tests (targeted analysis)

```
# DCT LSB bias — most decisive test for JPEG steganography
python3 -c "
from analyzer_v2 import analyze_dct_lsb; analyze_dct_lsb('images/IMAGE.jpg')
"

# RS Attack — spatial domain LSB analysis
python3 -c "
from analyzer_v2 import rs_attack; import json, sys
r = rs_attack('images/IMAGE.jpg')
print(json.dumps(r, indent=2))
"

# SPA — Sample Pairs Analysis
python3 -c "
from analyzer_v2 import spa_attack; import json
r = spa_attack('images/IMAGE.jpg')
print(json.dumps(r, indent=2))
"

# Weighted Stego Attack
python3 -c "
from analyzer_v2 import weighted_stego_attack; import json
r = weighted_stego_attack('images/IMAGE.jpg')
print(json.dumps(r, indent=2))
"

# Calibration Chi-Square (F5 detection)
python3 -c "
from analyzer_v2 import calibration_chi_square; import json
r = calibration_chi_square('images/IMAGE.jpg')
print(json.dumps(r, indent=2))
"
```

### 4. Interpret Results

| Test | Key Metric | Suspicious Threshold | What It Detects |
|------|-----------|---------------------|-----------------|
| **DCT LSB Bias** | LSB=1 ratio | >0.55 or <0.45 | DCT-domain steganography (primary) |
| **DCT LSB Bias** | Odd/even small coef | >0.60 | Payload concentration in \|1\| coeffs |
| **RS Attack** | RS stat | abs > 0.05 | Spatial LSB embedding |
| **SPA** | alpha | > 0.05 | LSB embedding rate |
| **Weighted Stego** | deviation from 0.25 | > 0.02 | Weighted LSB bias |
| **Calibration** | avg chi-square | < 5.0 | F5-like encoding (low variance) |

## Key Finding: DCT LSB Bias in Quantized AC Coefficients

This is the **most decisive test** for JPEG steganography. It analyzes the LSB of each quantized DCT coefficient value read directly from the JPEG file via `jpegio`.

Natural JPEGs: LSB=1 ratio ≈ 0.50 (uniform)
Steganographic: LSB=1 ratio > 0.55 (biased toward odd coefficients)

The test is particularly effective when:
- Applied to small-magnitude coefficients (|coef| ≤ 10)
- Used on the channel with highest payload concentration (often Ch2/blue)
- Combined with magnitude concentration analysis (stego payloads concentrate in |1| coeffs)

## Tool Reference

| Tool | Purpose | Source |
|------|---------|--------|
| `analyzer_v2.py` | Unified analyzer (all tests) | `work/roemmele-stego/` |
| `jpegio` | JPEG DCT coefficient reader | `uv pip install jpegio` |
| `Aletheia` | Reference steganalysis framework | `work/roemmele-stego/aletheia/` |
| `ImageMagick` | Calibration attack (crop/recompress) | System install |
| `scipy` | Fallback DCT (less accurate) | Standard scientific Python |

## Workspace Structure

```
work/roemmele-stego/
├── analyzer_v2.py       # Enhanced analyzer (main tool)
├── dct_fast.py          # v1 DCT analyzer
├── compare.py           # v1 cross-image correlation
├── stego-venv/         # Python venv
├── images/             # Sample images
├── aletheia/           # Reference implementation
│   ├── aletheialib/attacks.py   # SPA, RS, calibration
│   ├── aletheialib/feaext.py    # SRM/DCTR/GFR feature extractors
│   └── aletheialib/jpeg.py      # JPEG reader (Octave-based)
└── report/
    └── README.md       # Analysis report (auto-update from findings)
```
