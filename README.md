# 🃏 BIP39 Playing Card Seed Generator

Convert physical playing card draws into mathematically rigorous BIP-39 seed phrases. 

This is a single, offline, network-free HTML file. It is a dumb calculator: it provides zero entropy of its own. The randomness comes entirely from the physical universe—your shuffling of a standard 52-card deck.

## Screenshots

<p float="left">
  <img src="Photos/p1.jpg" alt="52-card tap grid, five cards per row on mobile" width="300">
  <img src="Photos/p2.jpg" alt="Draw counter, Generate Seed and Clear controls" width="300">
  <img src="Photos/p3.jpg" alt="Generated mnemonic and raw entropy panels" width="300">
  <img src="Photos/p4.jpg" alt="Modulo bias audit terminal" width="300">
</p>

## Core Philosophy: Zero Extraction, Exact Sampling

Most "cards-to-seed" tools on the internet contain a fatal mathematical flaw: they allow you to deal through a single deck (drawing *without replacement*). A single 52-card deck contains exactly **225.58 bits** of entropy ($\log_2(52!)$). This is mathematically insufficient to generate a 24-word (256-bit) BIP-39 seed. To bypass this, other tools secretly run your deck string through a SHA-256 extractor to "stretch" it to 256 bits. 

**This tool refuses to do that.** 

We demand you draw **with replacement** (record the card, return it to the deck, and reshuffle). This guarantees an infinite stream of independent, identically distributed (i.i.d.) uniform random variables. Each draw yields exactly $\log_2(52) \approx 5.7004$ bits of entropy. 

The accumulated bits are converted to binary via **exact bounded rejection sampling**. There is no modulo bias, no hash-based stretching, and no extractor assumption. SHA-256 is used in this pipeline *exactly once*, at the precise step the BIP-39 specification mandates: deriving the checksum bits for the final word.

## The Math & Tiers

Because we use a base-52 stream with exact rejection sampling, we target the BIP-39 bit requirements plus a small entropy buffer to ensure the sampling shortfall is negligible (any extra draws are absorbed, never wasted).

| Seed Length | Entropy Required | Theoretical Min Draws | **Tool Requirement (with buffer)** |
| :--- | :--- | :--- | :--- |
| **12 Words** | 128 bits | 23 | **25 draws** |
| **15 Words** | 160 bits | 29 | **31 draws** |
| **18 Words** | 192 bits | 34 | **36 draws** |
| **21 Words** | 224 bits | 40 | **42 draws** |
| **24 Words** | 256 bits | 45 | **48 draws** |

## Security & Air-Gap Hygiene

*   **Zero Network Calls:** The file makes no external requests. No analytics, no CDNs, no fonts. 
*   **Fail-Closed Integrity:** On load, the tool runs Known Answer Tests (KATs) against the BIP-39 wordlist, its internal SHA-256 implementation, and the `cardsToEntropy` derivation function. If any self-test fails, the UI refuses to generate a seed.
*   **Volatile Memory:** The generated phrase and raw entropy auto-clear from the screen after 3 minutes.
*   **Clipboard Scrubbing:** Copy buttons are disabled by default. If enabled and used, the tool attempts to overwrite and clear the clipboard shortly after copying. Use the **🧹 Clear / Reset (Wipe Memory)** button to instantly nullify all inputs and variables.
*   **Audit Terminal:** A built-in modulo bias and sequence audit terminal allows you to verify the math on your specific physical draws. Advanced sequence-derived values are hidden by default to prevent shoulder-surfing or accidental screenshots.

## UI & Art Attribution

The UI features a dynamic flexbox layout that scales from 4 cards per row on small phones up to 13 cards per row (full suits) on desktop. 

The crisp SVG card art is sourced from **[cards.revk.uk](https://cards.revk.uk/)** and is in the public domain. The art has been sanitized to inert `<path>` data and embedded directly into the HTML to maintain the single-file, zero-dependency architecture.

### Known Browser Quirks

1.  **The "Not Secure" Banner:** When opening a local file, Chromium-based browsers (Brave, Opera, Chrome) may display a "Not Secure" or "Identity not verified" warning because there is no TLS certificate. **This is expected and irrelevant.** The tool performs no network activity; your secrets never traverse a connection. Integrity is established by the SHA-256 sidecar and signed releases, not HTTPS.
2.  **Night Mode / Eye Protection Pips:** Some mobile browsers (e.g., Xiaomi/Mi Browser) apply forced-dark heuristics that remap pure-black fills on light backgrounds. If your Clubs and Spades number pips look "washed out" or grey, disable your browser's night mode or eye-protection feature for this page. The tool's native dark theme does not require browser intervention.

## Provenance & Signed Releases

Release tags (September 2026 onward) are signed with the account SSH signing key — the same key covers all of IanMcLo's repositories.

Fingerprint (SHA256): `6D+lcVxQsXH+3QK+x6luF5L7tdjajExZSKGJQuPcqZo`

Verify: 
```bash
git verify-tag <tag> --show-signature
```

Full scope, safe harbor, and reporting policy: see [SECURITY.md](SECURITY.md).

## License

MIT License. Verify the fingerprint, everything else is yours to use.# 🎲 BIP39 Dice Roll Seed Generator

A single-file, offline, cryptographically auditable BIP-39 seed phrase generator driven by physical dice rolls.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/IanMcLo/bip-39-dice/blob/main/LICENSE) [![HTML 100%](https://img.shields.io/badge/HTML-100%25-orange)](https://github.com/IanMcLo/bip-39-dice/blob/main)

Built for security purists: this tool uses **exact rejection sampling** to reduce modulo bias to mathematically **zero**, features on-load Known Answer Tests (KATs) that fail-closed, and includes a live Modulo Bias Audit Terminal so you can verify the math yourself.

## 🛡️ Security & Features

- **Zero Modulo Bias (Rejection Sampling):** The base-6 roll integer is mapped to the target $2^b$ space. If the integer falls into the remainder zone ($X \ge T$), the tool refuses to generate and asks you to re-roll.
- **Live Modulo Bias Audit Terminal:** A floating action button (🔬) opens a live audit sheet showing target size, roll count, and the live verdict (✅ ACCEPT or ⛔ REJECT) on every keystroke. By default the sequence-derived math (N, r, T, and the roll-derived integer X) stays hidden — those values are computed from your actual dice rolls and shouldn't sit on screen or in a screenshot during ordinary use. An explicit "Show advanced values" toggle in the audit sheet reveals them for anyone who wants to verify the math directly.
- **On-Load Self-Tests (Fail-Closed):** Every page load verifies the SHA-256 implementation, the official BIP-39 wordlist hash, the roll-to-entropy packing, and 4 official BIP-39 test vectors. If any test fails, the Generate button is disabled.
- **Strict Input Validation:** The dice-roll field only accepts digits 1–6 and optional whitespace. Anything else is flagged immediately with a visible error and disables Generate — invalid characters are never silently discarded, so a mistyped digit can't quietly change your entropy without you noticing.
- **Enhanced Entropy Buffers:** Minimum roll requirements are increased across all tiers to maximize the cryptographic safety margin:
  * **12 words:** 55 rolls (~142 bits raw)
  * **15 words:** 66 rolls (~170 bits raw)
  * **18 words:** 79 rolls (~204 bits raw)
  * **21 words:** 92 rolls (~238 bits raw)
  * **24 words:** 105 rolls (~271 bits raw)
- **Clipboard-Aware Reset:** Copying the seed phrase or entropy hex auto-clears the clipboard 45 seconds later — but only if it still holds exactly what was copied, so nothing else you copy in the meantime gets clobbered. "Clear Rolls" and "Hard Reset" both attempt the same clipboard clear immediately, alongside wiping the DOM and nullifying closure-scoped JavaScript variables. Note this is a best-effort clear, not guaranteed secure erasure — no web app can force the OS or other apps to forget a value once it's been on the clipboard.
- **Mobile-First & Desktop-Ready:** Responsive bottom sheets, collapsible status pills, and native numeric keypads for phones; centered audit cards for desktops.

## 📦 Verification

To ensure the file you downloaded hasn't been tampered with, verify the SHA-256 checksum.

**Option 1: Sidecar file (Linux/macOS)** Download `index.html.sha256` into the same folder and run:

```
sha256sum -c index.html.sha256
# Expected output: index.html: OK
```

**Option 2: Manual Hash** Hash your local `index.html` file using any trusted SHA-256 tool and compare it against the published hash in `index.html.sha256`.

## 💻 Usage

1. **Air-gap your device:** Disconnect from the internet.
2. Open `index.html` in any modern browser.
3. Wait for the green "✅ Wordlist + self-tests verified" pill to appear.
4. Select your desired word count (12–24 words).
5. Roll a physical 6-sided die and enter the numbers into the input field, one digit per roll. Whitespace is fine if you want to group rolls visually — it's ignored either way — but any other character will be flagged as invalid rather than silently dropped.
6. Tap the 🔬 button to watch the live rejection sampling verdict; toggle "Show advanced values" if you want to see the underlying math.
7. Once you hit the target roll count, tap **Generate Seed**.
8. Write down your phrase, tap **Clear / Reset**, and power off the device.

## 📄 License

MIT – use at your own risk. This is security‑critical software.
Review the code, verify the outputs against known test vectors, and **only use on air‑gapped devices**.
