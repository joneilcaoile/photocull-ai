# Contributing to PhotoCull AI

Thanks for your interest in contributing! PhotoCull AI is a single-file web application by design — this keeps it easy to fork, share, and modify without any build tools.

## How to Contribute

1. **Fork** the repository
2. **Create a branch** for your feature or fix (`git checkout -b feature/your-feature`)
3. **Make your changes** to `photo-culler.html`
4. **Test** by opening the file in a browser and running it against real photos
5. **Submit a pull request** with a clear description of what you changed and why

## What Would Help Most

These are the highest-impact contributions right now (in priority order):

1. **NIMA integration** — Neural Image Assessment via TensorFlow.js for deep-learned aesthetic scoring. This would be the single biggest quality improvement.
2. **Web Worker processing** — Move analysis to a background thread so the UI stays responsive during large batches.
3. **More EXIF tags** — White balance, metering mode, flash status.
4. **Better blink detection** — MediaPipe FaceMesh for precise eye landmark tracking.
5. **RAW file support** — .CR2, .ARW, .NEF parsing.

## Code Style

- The project is a single HTML file. Keep it that way.
- Use `'use strict'` mode.
- All user-controlled strings displayed in HTML must be escaped with the `esc()` function.
- No external JS dependencies beyond TensorFlow.js (loaded from CDN).
- Use `const` and `let`, never `var`.
- Comment non-obvious logic, especially math (scoring functions, signal processing).

## Security

- All innerHTML injections must use `esc()` for user-controlled data (filenames, EXIF strings).
- Do not add `eval()`, `Function()`, or dynamic script injection.
- Do not add external network calls — the app must remain fully local.
- If you find a security issue, please report it privately (see [SECURITY.md](SECURITY.md)).

## Testing

There's no automated test suite yet. Before submitting a PR, please manually verify:

- Drop 10+ photos and confirm all analysis features run without errors
- Check the browser console for warnings or errors
- Test with both portrait photos (faces) and non-portrait photos (landscapes, objects)
- Test with photos that have no EXIF data
- Test keyboard shortcuts in filmstrip view

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
