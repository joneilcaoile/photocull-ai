# Changelog

All notable changes to PhotoCull AI will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-03-22

### Added
- Tenengrad sharpness analysis with Gaussian pre-smoothing (threshold=10)
- Face detection via TensorFlow.js BlazeFace
- Per-eye focus scoring relative to global sharpness
- Catchlight detection (specular highlights in eyes)
- Blink detection via eye region luminance variance analysis
- Built-in EXIF parser reading ISO, aperture, shutter speed, focal length, make/model
- Duplicate photo grouping via perceptual hash (dHash, Hamming distance ≤ 8)
- Composition scoring (rule of thirds, face positioning, headroom, horizon detection)
- Color and vibrance analysis (HSL saturation, vibrance weighting, color cast detection)
- Exposure scoring (histogram analysis, clipping detection, dynamic range)
- Noise estimation (local-mean residual isolation)
- ML scoring model with differential evolution-optimized coefficients (5-fold stratified CV)
- Three view modes: Grid, Large Grid, Filmstrip
- Keyboard rapid-culling (1/2/3 = Keep/Maybe/Reject, arrow keys to navigate)
- Side-by-side photo comparison (Shift+click)
- Sort by score, name, sharpness, or ISO
- Filter by tier, blinks, or duplicates
- CSV export with all metrics and EXIF data
- Adjustable weights, thresholds, and feature toggles
- XSS prevention via HTML entity escaping on all user-controlled strings
- Content Security Policy meta tag
- 80MB file size limit to prevent resource exhaustion

[1.0.0]: https://github.com/joneilcaoile/photocull-ai/releases/tag/v1.0.0
