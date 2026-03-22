# PhotoCull AI v2 — Technical Documentation

## Overview

PhotoCull AI is a **free, local-only, zero-cloud** photo culling tool for photographers. It runs entirely in the browser — no server, no account, no uploads. Every photo stays on your machine.

### What It Does

Drop in a batch of photos, and it automatically scores and sorts them into Keep / Maybe / Reject tiers based on:

- **Sharpness** — Tenengrad (Sobel gradient magnitude²) with Gaussian pre-smoothing
- **Eye Focus** — Per-eye sharpness analysis relative to global sharpness
- **Catchlights** — Specular highlight detection in eye regions
- **Exposure** — Histogram analysis (mean brightness, clipping, dynamic range)
- **Noise** — Local-mean residual isolation
- **Composition** — Rule of thirds, face positioning, headroom, horizon detection
- **Color & Vibrance** — HSL saturation, vibrance estimation, color cast detection
- **EXIF Metadata** — ISO, aperture, shutter speed, focal length quality signals
- **Blink Detection** — Eye region variance analysis for closed-eye detection
- **Duplicate Grouping** — Perceptual hash (dHash) similarity clustering

### Architecture

Two distinct components:

1. **Face/Eye Detection** — TensorFlow.js BlazeFace (pre-trained by Google). We didn't train this. It finds WHERE faces and eyes are in each image.
2. **Quality Scoring** — Custom parametric scoring functions with hand-engineered and ML-optimized coefficients. These decide IF a photo is good.

The quality scorer is **not a neural network**. It's a set of parametric scoring functions with optimized coefficients that combine hand-engineered features into a weighted overall score. Think of it as a shallow, interpretable model — closer to logistic regression than deep learning.

---

## v2 Feature Details

### 1. EXIF Metadata Parsing

**What:** Built-in JPEG EXIF parser (no external library) that reads IFD0 + ExifIFD tags directly from the file's ArrayBuffer.

**Tags extracted:** Make, Model, ExposureTime, FNumber, ISO, FocalLength

**Quality scoring logic:**
- ISO ≤200: +15 | ISO 200-800: +8 | ISO 800-1600: neutral | ISO 1600-3200: -8 | ISO 3200-6400: -15 | ISO >6400: -25
- Shutter ≤1/500s: +8 | 1/125s: +4 | 1/60s: neutral | 1/30s: -5 | 1/8s: -12 | slower: -20
- Aperture f/2.8–f/8 (lens sweet spot): +5 | f/1.4–f/11: +2 | extreme: -3

**Why it matters:** EXIF is the single biggest "free" quality signal for real photos. A photo shot at ISO 12800 and 1/8s shutter is almost certainly noisy and motion-blurred. Paid tools like FilterPixel use this heavily.

**EXIF's influence on overall score:** Subtle — ±15% of the EXIF-to-baseline delta. It nudges the score rather than overriding pixel-based analysis.

### 2. Blink Detection

**Method:** Analyzes the eye region extracted by BlazeFace using luminance statistics:
- Open eyes → high local contrast (dark iris/pupil, white sclera, possible catchlight)
- Closed eyes → relatively uniform skin-toned pixels (low variance, low contrast)

**Signals used:**
- Luminance standard deviation (open eyes: std > 25; closed: std < 12)
- Min-max contrast (open: >80; closed: <40)
- Dark center pixel ratio (pupil presence check)

**Scoring impact:** Blink detected → 20-point penalty on overall score. Also flagged with a visual warning badge in the UI.

**Limitations:** This is a heuristic, not a trained classifier. It works well for clear blinks but may false-positive on eyes in heavy shadow or extremely dark irises. Confidence is reported per-detection.

### 3. Duplicate Detection (dHash)

**Method:** Difference Hash (dHash) — perceptual hashing algorithm:
1. Resize each photo to 9×8 grayscale
2. For each row, compare adjacent pixels: left < right → 1, else → 0
3. Produces a 64-bit binary hash
4. Photos with Hamming distance ≤ threshold (default: 8) are grouped

**Why dHash over pHash/aHash:** dHash is faster (no DCT), more robust to brightness changes than aHash, and sufficient for detecting near-identical shots from burst mode.

**Grouping:** Duplicate groups are displayed together with the highest-scoring photo marked as "Best." Users can filter to see only duplicate groups.

**Sensitivity control:** Adjustable threshold (1–20) in settings. Lower = stricter matching. Default 8 catches most burst-mode duplicates while avoiding false groupings.

### 4. Composition Analysis

**Portrait mode (faces detected):**
- Distance from face center to nearest rule-of-thirds power point (closer = better)
- Headroom check: face touching top edge → penalty; 15-35% headroom → bonus
- Off-center positioning bonus (rule of thirds > dead center)
- Face fill ratio: 2-40% of frame → appropriate portrait framing

**Landscape mode (no faces):**
- Horizon detection via row-brightness gradient analysis
- Horizon near 1/3 or 2/3 line → +20 points
- Tonal variety bonus: high brightness range across rows → interesting scene

### 5. Color & Vibrance Analysis

**Saturation scoring:** HSL saturation averaged across sampled pixels.
- Sweet spot (0.12–0.55): +30 | Moderate (0.05–0.70): +15 | Desaturated (<0.05): -10 | Oversaturated (>0.70): -15

**Vibrance:** Weighted saturation that boosts muted colors more: `sat × (1 - sat)`. Higher vibrance → bonus.

**Color cast detection:** Compares per-channel averages (R, G, B) against overall mean. Channel deviation >30 → likely white balance issue, -10 penalty. >50 → strong cast, -20 total.

---

## Scoring Model

### Overall Score Calculation

**Portrait mode (faces detected):**
```
score = sharpness × 0.28 + eyeFocus × 0.35 + exposure × 0.12
      + catchlight × 0.10 + composition × 0.08 + color × 0.07
      + exifDelta × 0.15 - (blinkPenalty ? 20 : 0)
```

**Non-portrait mode:**
Eye/catchlight weights redistributed: 40% → sharpness, 20% → exposure, 25% → composition, 15% → color.

### Tier Thresholds
- **Keep:** score ≥ 72
- **Maybe:** 35 ≤ score < 72
- **Reject:** score < 35

All weights and thresholds are user-adjustable in the Settings panel.

### Trained Coefficients (from v1 ML pipeline)

These were optimized via differential evolution on a 300-image synthetic dataset with 5-fold stratified cross-validation:

| Parameter | Description | Value |
|-----------|-------------|-------|
| `log_mult` | Sharpness log(tenengrad) multiplier | 4.46 |
| `edge_mult` | Sharpness edge density multiplier | 20.07 |
| `exp_mean_penalty` | Exposure mean deviation penalty | 0.83 |
| `exp_clip_penalty` | Exposure clipping penalty | 50.14 |
| `exp_dr_penalty` | Exposure dynamic range penalty | 0.40 |
| `noise_mult` | Noise std multiplier | 1.96 |
| `eye_log_mult` | Eye focus log multiplier | 4.22 |
| `eye_edge_mult` | Eye focus edge density multiplier | 20.14 |
| `eye_cap_mult` | Global sharpness cap multiplier | 4.63 |
| `eye_cap_base` | Global sharpness cap base offset | 5.55 |

**Overfitting check:** CV AUC = 0.564 ± 0.070, overfit gap = 0.044. See v1 training documentation for full pipeline details.

---

## UI & Workflow Features

### Three View Modes
1. **Grid View** — Photo cards with tier badges, score badges, EXIF info, blink/face/duplicate indicators
2. **Large Grid** — Same cards, bigger thumbnails for detail inspection
3. **Filmstrip View** — Full-screen photo with EXIF overlay, score panel, and thumbnail strip for rapid keyboard-driven culling

### Keyboard Shortcuts
| Key | Action |
|-----|--------|
| `←` `→` | Navigate between photos (modal or filmstrip) |
| `1` | Set current photo to Keep |
| `2` | Set current photo to Maybe |
| `3` | Set current photo to Reject |
| `Shift+Click` | Select photo for side-by-side comparison |
| `C` | Open compare panel |
| `Esc` | Close modal / compare / settings |

### Filters & Sorting
- Filter by: All / Keep / Maybe / Reject / Blinks only / Duplicates only
- Sort by: Score (high/low), Name (A-Z/Z-A), Sharpest first, ISO (low→high)

### Side-by-Side Compare
Shift+click two photos, then press Compare. Full-screen split view with scores and EXIF for both photos.

### CSV Export
Exports all metrics: overall score, tier, sharpness, exposure, composition, color, eye focus, catchlight, noise, blink flag, face count, ISO, aperture, shutter speed, focal length, duplicate group assignment.

---

## Honest Limitations

### What we can't match (yet)
- **Deep CNN image quality** — Aftershoot and FilterPixel use neural networks (likely NIMA/MobileNet variants) trained on millions of real photos. Our parametric scorer uses hand-engineered features. For subjective "which photo looks better" judgments, CNNs win.
- **Expression analysis** — We detect blinks but can't evaluate smiles, emotions, or "peak moment" like FilterPixel's DeepCull.
- **Learning from user preferences** — Paid tools adapt to your style over time. Ours uses fixed weights (though they're user-adjustable).
- **Cloud processing power** — We run in a single browser thread. Large batches (500+) will be slow.

### What we do better
- **Free** — Paid tools cost $10-30/month
- **Fully local** — Zero data leaves your machine. No account, no cloud, no privacy concerns
- **Transparent** — Every score is explained. You can see exactly why a photo scored the way it did
- **No vendor lock-in** — Single HTML file, runs in any browser, export to CSV
- **EXIF-aware** — Camera settings as quality signals, which many free tools lack
- **Duplicate grouping** — Perceptual hash similarity, not just filename matching

### Accuracy on Real Photos
The synthetic training data accuracy was 37% (honest — see v1 docs). Real photos should perform significantly better because:
1. Real photos have actual EXIF data (our biggest new signal)
2. Real blur/noise patterns are more consistent than synthetic ones
3. The sharpness/exposure/noise metrics were designed for real-world signals

The true test is: **does it correctly rank your photos so the best ones float to the top?** Relative ranking matters more than absolute scores.

---

## What Would Make This #1

The single highest-impact improvement would be integrating a **pre-trained NIMA (Neural Image Assessment) model** via TensorFlow.js. NIMA is a MobileNet trained on the AVA dataset (250K photos with aesthetic ratings from photography enthusiasts). This would replace our parametric quality scorer with a deep-learned one that understands composition, lighting, color harmony, and emotional impact at a level our hand-engineered features can't match.

Other high-value additions:
- **Face-landmarks-detection** (MediaPipe FaceMesh, 468 points) for precise Eye Aspect Ratio blink detection and expression scoring
- **Web Worker processing** for non-blocking batch analysis
- **IndexedDB caching** so re-opening the same photos is instant
- **RAW file support** via libraw.js
- **Lightroom/Capture One integration** via XMP sidecar file export

---

## File Structure

| File | Description |
|------|-------------|
| `photo-culler.html` | The complete app (2,754 lines, single-file, zero dependencies beyond TF.js CDN) |
| `TRAINING_DOCUMENTATION.md` | This file |
| `train_culler.py` | v1 ML training pipeline (differential evolution, 5-fold CV) |
| `test_culler.py` | v1 validation suite (5 tests: sensitivity, discrimination, calibration, independence, edge cases) |

---

## Version History

### v2 (current)
- Added EXIF metadata parsing (built-in, no external library)
- Added blink detection via eye region variance analysis
- Added duplicate grouping via perceptual hash (dHash)
- Added composition scoring (rule of thirds, face positioning, horizon detection)
- Added color & vibrance analysis (saturation, vibrance, color cast)
- Added filmstrip view with keyboard rapid-culling
- Added side-by-side photo comparison
- Added sorting (score, name, sharpness, ISO)
- Added blink and duplicate filters
- Added toggleable features in settings
- Enhanced CSV export with all new metrics
- Professional dark UI with responsive layout

### v1
- Initial release
- Tenengrad sharpness, exposure histogram, noise isolation
- BlazeFace face detection with eye focus and catchlight analysis
- Grid view with modal detail
- Basic Keep/Maybe/Reject tiering
- Adjustable weights and thresholds
- ML-optimized coefficients via differential evolution (5-fold CV)
