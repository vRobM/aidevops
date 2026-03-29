---
name: stego-setup
description: Setup guide for steganography analysis environment — Python venv, jpegio, ImageMagick, and Aletheia framework
metadata:
  tags:
    - setup
    - python
    - venv
    - jpegio
    - imagemagick
    - dependencies
---

# Setup — Steganography Analysis Environment

Environment lives in `~/.aidevops/.agent-workspace/work/roemmele-stego/`.

## Python Environment

A venv is pre-configured with all dependencies:

```bash
cd ~/.aidevops/.agent-workspace/work/roemmele-stego/
source stego-venv/bin/activate
python --version
```

If venv is missing or broken, recreate:
```bash
cd ~/.aidevops/.agent-workspace/work/roemmele-stego/

# Create venv with uv (preferred)
uv venv stego-venv --python 3.11

# Activate and install dependencies
source stego-venv/bin/activate
uv pip install jpegio numpy Pillow scipy

# Verify
python -c "import jpegio; print('jpegio', jpegio.__version__)"
```

## Dependencies

| Package | Purpose | Install |
|---------|---------|---------|
| `jpegio` | JPEG DCT coefficient reader | `uv pip install jpegio` |
| `numpy` | Array operations | `uv pip install numpy` |
| `Pillow` | Image loading | `uv pip install Pillow` |
| `scipy` | Fallback DCT | `uv pip install scipy` |

## ImageMagick

Required for calibration attack (crop + recompress):

```bash
# macOS
brew install imagemagick

# Verify
convert --version
identify --version
```

If ImageMagick is unavailable, the calibration test will return `{"verdict": "ERROR"}`.

## Aletheia (Reference Implementation)

Aletheia is a reference steganalysis framework. Clone if not present:
```bash
cd ~/.aidevops/.agent-workspace/work/roemmele-stego/
git clone https://github.com/danieller/aletheia.git aletheia
```

Or install via pip:
```bash
uv pip install aletheia
```

Key files in Aletheia:
```
aletheia/aletheialib/
├── attacks.py      # SPA, RS, calibration implementations
├── feaext.py       # SRM/DCTR/GFR feature extractors
├── models.py       # ML ensemble, neural network models
└── jpeg.py         # JPEG reader (Octave-based, for comparison)
```

## Quick Health Check

```bash
cd ~/.aidevops/.agent-workspace/work/roemmele-stego/
source stego-venv/bin/activate

python -c "
import sys
print('Python:', sys.version)
try:
    import jpegio; print('jpegio: OK')
except: print('jpegio: MISSING')
try:
    import numpy; print('numpy:', numpy.__version__)
except: print('numpy: MISSING')
try:
    from PIL import Image; print('Pillow: OK')
except: print('Pillow: MISSING')
"

# ImageMagick check
convert -version 2>&1 | head -1
```
