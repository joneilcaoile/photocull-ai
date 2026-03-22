# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.0.x   | Yes       |

## Reporting a Vulnerability

If you discover a security vulnerability in PhotoCull AI, please report it responsibly.

**Email:** supernoob230@gmail.com

**What to include:**
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if you have one)

**Response time:** I'll acknowledge your report within 48 hours and aim to release a fix within 7 days for confirmed vulnerabilities.

## Security Architecture

PhotoCull AI is designed with security as a core principle:

- **100% local processing** — No data leaves your browser. No server, no cloud, no analytics.
- **XSS prevention** — All user-controlled strings (filenames, EXIF data) are sanitized via HTML entity escaping before DOM insertion.
- **Content Security Policy** — CSP meta tag restricts script sources, image sources, and network connections.
- **Resource limits** — 80MB file size cap prevents memory exhaustion attacks.
- **No eval()** — No dynamic code execution anywhere in the codebase.
- **No cookies or storage** — Nothing is persisted between sessions.

## Threat Model

Since PhotoCull AI runs entirely client-side with no server component:

- **No network attack surface** — The app makes no outbound requests except loading TensorFlow.js from CDN.
- **No data storage** — Photos are processed in memory and discarded when you close the tab.
- **Primary risk vector** — Maliciously crafted image files with XSS payloads in EXIF metadata (mitigated by the `esc()` sanitizer).
