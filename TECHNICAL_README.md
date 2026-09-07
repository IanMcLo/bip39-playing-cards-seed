# Technical & Cryptographic Specification

This document describes the information-theoretic foundations, entropy bounds, sampling pipeline, and security model of the `bip-39-cards` physical seed generator. It is intended as a precise, implementable reference for auditors and advanced users.

---

## 1. Information-Theoretic Foundations

### 1.1 Physical Entropy Source and Uniform Card Mapping

The physical entropy source is repeated independent draws from a standard 52-card pack **with replacement and reshuffling**. Each draw yields exactly one of 52 distinct outcomes (13 ranks × 4 suits).

Shannon entropy per draw:

$$H(X) = -\sum_{x=0}^{51} P(X=x)\log_2 P(X=x) = -52 \times \frac{1}{52}\log_2\left(\frac{1}{52}\right) = \log_2 52 \approx 5.7004 \text{ bits/draw}$$

Min-entropy per draw (worst-case single-trial predictability):

$$H_\infty(X) = -\log_2\left(\max_x P(X=x)\right) = \log_2 52 \approx 5.7004 \text{ bits/draw}$$

Because Shannon entropy and min-entropy coincide under the uniform distribution, each draw contributes ≈ 5.7004 bits in both the average and worst-case senses prior to software processing. Returning the card and reshuffling between draws is what makes successive draws i.i.d.; skipping the reshuffle (dealing without replacement) changes the model entirely and is **not supported** (§5d).

### 1.2 Canonical Card Encoding

Each card maps to exactly one base-52 digit via a fixed bijection, in display order (clubs, diamonds, hearts, spades; ace low):

$$\text{digit}(s, r) = 13\,s + r,\quad s \in \{\clubsuit{=}0,\ \diamondsuit{=}1,\ \heartsuit{=}2,\ \spadesuit{=}3\},\quad r \in \{A{=}0,\ 2{=}1,\ \dots,\ K{=}12\}$$

yielding `A♣=0, 2♣=1, …, K♣=12, A♦=13, …, K♠=51`. Unlike orientation-ambiguous physical objects, a card has no canonical-ordering hazard: the UI enforces the mapping structurally — input is a 52-button tap-grid, so **no invalid or malformed draw can be entered**. The on-load known-answer test pins this mapping in code.

---

## 2. Framing, Checksum and Word Slicing

### 2.1 Raw Entropy Lengths

BIP-39 requires raw entropy lengths $E$ that are multiples of 32 bits:

| Mnemonic Length | Raw Entropy $E$ | Checksum bits $CS = E/32$ | Total bits ($E + CS$) |
|---|---|---|---|
| 12 words | 128 | 4 | 132 |
| 15 words | 160 | 5 | 165 |
| 18 words | 192 | 6 | 198 |
| 21 words | 224 | 7 | 231 |
| 24 words | 256 | 8 | 264 |

### 2.2 Checksum Derivation (SHA-256)

$$\text{Digest} = \text{SHA-256}(\mathrm{RawEntropy}_{\mathrm{bytes}}),\qquad \text{Checksum} = \text{Digest}[0 : CS-1]$$

The raw entropy bitstring concatenated with the checksum bits forms the bitstream $S$ of length $L = E + CS$. **This is the only use of SHA-256 in the entropy path**: it is mandated by BIP-39 itself. No hash-based extractor is applied to the draw stream at any point (§5d).

### 2.3 11-bit Word Indices

Partition $S$ into contiguous 11-bit chunks, MSB-first:

$$W_k = \sum_{i=0}^{10} S[11k+i] \times 2^{10-i},\qquad W_k \in [0, 2047]$$

indexing the BIP-39 English wordlist.

---

## 3. Empirical vs. Theoretical Min-Entropy

### 3.1 Single-Bit Variance Example (256 bits)

A uniformly random 256-bit string has expected number of ones $\mu = 128$ and standard deviation $\sigma = \sqrt{256/4} = 8$. Observing 132 ones corresponds to $z = 0.5$, well within normal fluctuation. Apparent "entropy loss" in short samples is a measurement artifact, not a cryptographic weakness.

### 3.2 Byte-Level Sampling Artifacts

Viewing $N = 32$ bytes (256 bits) as 8-bit symbols, the maximum empirical frequency of any byte value is $1/32 = 0.031$. Naive per-byte frequency calculations under-estimate entropy for short samples; use bitwise statistics or aggregate many samples before inferring degradation.

---

## 4. Downstream Key-Derivation Architecture

Standard wallet software expands the mnemonic per BIP-39:

$$\text{Seed} = \text{PBKDF2-HMAC-SHA512}(\text{Mnemonic},\ \texttt{"mnemonic"} \parallel \text{Passphrase},\ 2048,\ 512)$$

The derived seed's security cannot exceed the entropy of the mnemonic; the dominant security parameter remains the raw entropy length $E$.

---

## 5. Bounded Rejection Sampling: Eliminating Modulo Bias Exactly

### a. Defining the Spaces

For $k$ draws, the total outcome space is:

$$N = 52^k$$

For target bits $b$, the target range is $R = 2^b$. Let $q = \lfloor N/R \rfloor$ and $r = N \bmod R$. Naive modulo reduction would give $r$ buckets $q+1$ preimages and $R-r$ buckets $q$ — the source of modulo bias.

### b. The Rejection Boundary

Define the uniformity boundary:

$$T = N - (N \bmod R)$$

Let $X$ be the integer formed by treating the draws as base-52 digits (first draw = most significant digit, digit per §1.2).

- **If $X \geq T$: reject** — the UI surfaces the rejection and requests one additional draw; the accumulator extends to $k+1$ digits. Rejection costs physical draws only; it never biases output.
- **If $X < T$: accept** — output $X \bmod R$.

Every accepted $X$ lies in a range of size $T$, an exact multiple of $R$; each output bucket receives exactly $T/R$ accepted inputs. Uniformity is exact, by construction, with **no extractor assumption**.

### c. Rejection Probability Per Tier

$P(\text{reject}) = r/N < 2^{-(H-b)}$, exactly computable per tier:

| Words | Draws ($k$) | Target ($b$) | Raw bits $H = k\log_2 52$ | Buffer ($H-b$) | $P(\text{reject})$ |
|---|---|---|---|---|---|
| 12 | 25 | 128 | 142.511 | 14.511 | ≈ 1 in 23,348 |
| 15 | 31 | 160 | 176.714 | 16.714 | ≈ 1 in 107,482 |
| 18 | 36 | 192 | 205.216 | 13.216 | ≈ 1 in 9,514 |
| 21 | 42 | 224 | 239.418 | 15.418 | ≈ 1 in 43,995 |
| 24 | 48 | 256 | 273.621 | 17.621 | ≈ 1 in 201,596 |

Draw counts above the theoretical minima ($\lceil b / \log_2 52 \rceil$ = 23/29/34/40/45) provide these buffers. Extra draws beyond the tier count are absorbed, never wasted.

### d. Why Dealing (Without Replacement) Is Refused

A fully dealt 52-card deck contains exactly:

$$\log_2(52!) \approx 225.582 \text{ bits}$$

which is **less than the 256 bits BIP-39 requires for 24 words**. Tools that accept a dealt deck must therefore stretch 225.582 bits into 256 via a hash-based extractor (typically SHA-256 over the deal string), substituting a computational assumption for information that does not exist. This tool refuses that model: per-draw radix under dealing is non-constant ($52, 51, 50, \dots$) and draws are not independent, invalidating §1.1 and §5. With replacement and reshuffling restores i.i.d. uniformity and makes the base-52 stream unbounded.

---

## 6. Physical Drawing Protocol

1. **Shuffle** the full 52-card pack thoroughly.
2. **Draw** one card at random, face unseen until selected.
3. **Record** it by tapping the matching card in the grid (each tap is a full card).
4. **Return** the card to the pack.
5. **Reshuffle** thoroughly before the next draw.
6. **Repeat** until the tier's draw count is reached.

Steps 4–5 are load-bearing: they are what makes each draw an independent uniform sample over 52 (§1.1). Dealing through the pack instead is an incompatible model (§5d) and the tool does not offer it.

---

## 7. Live Statistical Diagnostics (Bias Audit Terminal)

Sections 5–6 establish exact uniformity *provided* the physical draws are independent and fair. The terminal's checks are diagnostics **about the operator's pack and technique**, advisory only, with no influence on the §5b accept/reject decision. All live behind the opt-in toggle once minimum sample sizes are met.

### a. Suit Uniformity (Chi-Squared, df = 3)

For $n$ draws, let $O_s$ be the observed count of suit $s$, $E = n/4$. 

$$\chi^2 = \sum_{s=0}^{3} \frac{(O_s - E)^2}{E}$$

Compared against $\chi^2_{\text{crit}} = 7.815$ (df = 3, $\alpha = 0.05$). Requires $E \geq 5 \Rightarrow n \geq 20$.

### b. Rank Uniformity (Chi-Squared, df = 12)

Same statistic over 13 rank bins, $E = n/13$, $\chi^2_{\text{crit}} = 21.026$. Requires $n \geq 65$; reported as descriptive below that.

### c. Face-Level Chi-Squared (df = 51)

Over all 52 faces, $\chi^2_{\text{crit}} = 68.669$, but $E = n/52 \geq 5$ demands $n \geq 260$ — beyond every tier. Reported as a descriptive statistic only at tier sizes; never pass/fail below 260.

### d. Repeat-Rate Test

Under i.i.d. draws, $P(\text{draw}_i = \text{draw}_{i-1}) = 1/52$. For $M$ repeats in $n-1$ transitions:

$$z = \frac{M - (n-1)/52}{\sqrt{(n-1)\cdot \frac{1}{52}\cdot \frac{51}{52}}}$$

$|z| > 1.96$ shown as ⚠️ (insufficient shuffling tends to *deflate* repeats; over-mixing cannot inflate them systematically).

### e. Lag-1 Autocorrelation

On the digit sequence $v_1,\dots,v_n$ with mean $\bar{v}$:

$$r = \frac{\sum_{i=1}^{n-1}(v_i-\bar{v})(v_{i+1}-\bar{v})}{\sum_{i=1}^{n}(v_i-\bar{v})^2},\qquad z = \tanh^{-1}(r)\cdot\sqrt{(n-1)-3}$$

$|z| > 1.96$ shown as ⚠️. Detects adjacent-draw correlation only.

### f. Scope and Limitations

- Advisory only; opt-in; computed from the operator's actual draws.
- Cannot distinguish a biased pack from non-independent drawing technique.
- Lag-1 misses higher-lag patterns.

---

## 8. Implementation & Interoperability Notes

- **Byte/bit ordering:** first draw = most significant base-52 digit (MSB-first); the accepted integer is emitted as big-endian hex, zero-padded to the target byte length.
- **Rejection sampling, not truncation:** the accept/reject test precedes any modulo reduction; on rejection the UI requests one further draw (accumulator extends). Rejection is surfaced visibly, never silent.
- **Input methods:** a 52-button tap-grid is the sole draw-input path; malformed draws are structurally impossible.
- **Verification:** cross-check via the displayed raw entropy hex. Re-entering draw sequences into tools with different digit orderings, extraction hashes, or draw counts will not reproduce the same entropy.
- **Cryptographic primitives:** SHA-256 via `window.crypto.subtle` with a pure-JS fallback for offline `file://` use; PBKDF2-HMAC-SHA512 per BIP-39 for downstream derivation only.
- **Embedded artwork:** card faces are inert SVG path data (public domain, cards.revk.uk), sanitised of scripts, event handlers and external references; it participates in no code path.

---

## 9. Threat Model & Operational Security

- **Assumptions:** honest operator, physically private environment, recording medium under operator control.
- **Threats considered:** shoulder-surfing, biased or marked cards, insufficient shuffling, networked-device leakage, operator misreads, residual secrets in clipboard or Audit Terminal.
- **Operational recommendations:**
  - Draw and record in private; cameras out, microphones off, networked devices away.
  - Use a standard, undamaged 52-card pack; treat a flagged §7 diagnostic as a prompt to inspect the pack or improve shuffling.
  - **Pip legibility:** some mobile browsers' forced-dark / eye-protection modes remap pure-black fills, washing out ♣/♠ pips and inviting misreads. If pips look grey, disable browser night mode for this page — the tool's own theme is already dark.
  - Prefer an air-gapped device; verify artifact checksums and release signatures before use.
  - Treat raw entropy hex and mnemonic as maximally sensitive; clipboard auto-scrub after copy and on **🧹 Clear / Reset (Wipe Memory)** is best-effort, not guaranteed erasure; OS clipboard history may persist copies.
  - Sequence-derived values (H, N, r, T, X) stay hidden behind the opt-in toggle: they are computed from your actual draws and leak partial information in screenshots.
- **Out of scope:** supply-chain compromise of cryptographic libraries, OS-level compromise, coercion.

---

## 10. Common Pitfalls and How to Avoid Them

- **Dealing instead of drawing with replacement:** the single most common error in card-based entropy. Return and reshuffle every draw (§6); a dealt deck cannot even reach 256 bits (§5d).
- **Misreading pips:** verify rank and suit before tapping, particularly under poor lighting or browser recolouring (§9).
- **Re-typing draws into other tools:** verify with the displayed raw entropy hex instead.
- **Ordering mismatch:** this tool's digit mapping is §1.2's; other tools differ. Use the hex for cross-checks.
- **Insufficient draws:** generation is fail-closed below the tier count (25/31/36/42/48).

---

## 11. Trusted Code Base & Audit Checklist

- Identify the code paths for draw → base-52 BigInt → rejection gate → byte array → checksum → word slicing; confirm they are the only generation path.
- Confirm the rejection boundary $T = N - (N \bmod R)$ is enforced **before** any reduction, and that rejection is visible.
- Confirm §7 diagnostics are read-only: no feedback into the accept/reject decision.
- Confirm the on-load self-test suite fails closed: wordlist SHA-256 digest, SHA-256 known-answer tests, the `cardsToEntropy` known-answer test, and 4/4 official BIP-39 vectors must pass before Generate enables.
- Confirm embedded SVG art contains no scripts, handlers, or external references.
- Verify there are no network calls, telemetry, or remote loads.
- Verify release artifact checksums (sidecar) and tag signatures (SSH signing key fingerprint `SHA256:6D+lcVxQsXH+3QK+x6luF5L7tdjajExZSKGJQuPcqZo`).
- Reproduce §12 in an air-gapped environment.

---

## 12. Worked Example — Mechanics at Hand-Checkable Scale

The production known-answer test (a fixed 48-draw stream → pinned hex → mnemonic) lives in source as `CARDS_KAT` and is re-verified on every page load; it is deliberately not duplicated here so code remains the single source of truth. The reduced example below ($b = 8$) exercises every stage of §5 at a scale a human can verify with a calculator.

**Draws:** two, mapping to digits $d_0 = 39$, $d_1 = 32$ (§1.2).

1. $N = 52^2 = 2704$, $R = 2^8 = 256$.
2. $q = \lfloor 2704/256 \rfloor = 10$; $r = 2704 - 2560 = 144$; $T = 2560$.
3. $X = 39 \times 52 + 32 = 2060$.
4. $X < T \implies$ **ACCEPT**.
5. Output $= 2060 \bmod 256 = 12 = $ `0x0c`.

Had $X$ fallen in $[2560, 2704)$, the tool would request a third draw and re-evaluate at $k = 3$ — bias removed, cost one physical draw.

---

## Acknowledgements & References

- BIP-39: <https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki>
- Ian Coleman BIP39 tool: <https://iancoleman.io/bip39/>
- Playing-card SVG artwork: RevK, <https://cards.revk.uk/> (public domain)
- SHA-256 / PBKDF2 specifications (NIST, RFC 8018)
