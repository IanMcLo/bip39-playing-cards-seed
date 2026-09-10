## CHANGELOG

## v1.0.1

- Fixed placeholder card icon (🂡 → 🃏) for consistency with the rest of the UI.
- Fixed Hard Reset / Clear Draws to unconditionally overwrite the clipboard on explicit user action, rather than silently skipping the clear when clipboard-read permission is unavailable
- Added a CSP meta tag (default-src 'self' 'unsafe-inline'; connect-src 'none';) restricting the page to same-origin resources and blocking all outbound network requests.

Checksum file hash: 2a25b8376f615bae7d8e0eb04ef0112a6212bade26bf819f6b4f2c3c0006a893


## V1.0.0
**Initial release**
- Playing-card BIP-39 seed generator: draw-with-replacement from a standard 52-card pack, base-52 exact rejection sampling (mirrors the dice/dominoes bounded-rejection approach — zero modulo bias, verified draw counts per tier: 25/31/36/42/48 for 12/15/18/21/24 words).
- Full public-domain SVG card artwork (Adrian Kennard / RevK, cards.revk.uk), inlined as a single collision-free sprite sheet — zero network calls, fully self-contained offline.
- Mnemonic verification: paste a recovery phrase to check its BIP-39 checksum, with the same OPSEC-safe result (no raw entropy shown) as the sibling tools.
- Full non-persistence hardening: burn-after-reading auto-clear (3 min), clipboard scrub on paste, panic double-Escape hotkey, Hard Reset, `beforeunload` cleanup.
- Audit terminal diagnostics: rank (df=12) and suit (df=3) chi-square tests, decomposed from an infeasible flat 52-category test — following the dominoes build's own precedent for handling too-many-categories. Lag-1 autocorrelation test included.
- Full self-test suite on load: wordlist integrity, SHA-256 KATs (with `try/catch` hardening from day one), draws→entropy KAT, verifier KAT, official BIP-39 test vectors.

Checksum file hash: c9843a5fbd17747ed585498c4393f4c139911b7abf2acaa074b42ae106b0c069
