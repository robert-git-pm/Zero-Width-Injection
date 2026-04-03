# Zero-Width Combine & Inspect Tool

A lightweight, browser-only tool to:
1. combine **visible text** with **hidden text** encoded in zero-width Unicode characters,
2. copy the generated combined text,
3. paste it back after third-party roundtrips (e.g. social platforms),
4. extract and verify the hidden payload.

> This project is intentionally dependency-free (single `index.html`) to keep inspection, portability, and experimentation simple.

---

## Credits / Sources

This implementation is based on the ideas and research shared by **umpox**:

- GitHub repo: https://github.com/umpox/zero-width-detection
- Live demo/site: https://www.umpox.com/zero-width-detection/
- Background article: https://medium.com/@umpox/be-careful-what-you-copy-invisibly-inserting-usernames-into-text-with-zero-width-characters-18b4e6f17b66

Huge thanks to umpox for documenting practical zero-width insertion/detection behavior and real-world caveats around copy/paste pipelines.

---

## Quick Start

1. Open `index.html` in your browser.
2. Fill **Field 1**: printable text (visible).
3. Fill **Field 2**: hidden text (payload).
4. Copy **Field 3** using the button.
5. Paste into a target system (or anywhere else), save if needed, then copy it again.
6. Paste that copied text into **Field 4**.
7. Inspect **Field 5** (decoded hidden text).

No build step, no server required.

---

## End-User Guide

## UI Fields

1. **Printable Text**  
   The visible message users will read.

2. **Hidden Text**  
   The secret payload to embed into the visible message.

3. **Generated combined text (copy only)**  
   Auto-generated output (visible + zero-width payload). Read-only.  
   Use the **Copy combined text** button.

4. **Verification input (paste text here)**  
   Paste text copied from external systems (social media, docs, chats, etc.).

5. **Extracted Hidden Text**  
   Decoded payload extracted from Field 4.

## Typical Verification Flow

- Generate combined text locally.
- Paste into platform X and save.
- Re-copy text from platform X.
- Paste back into Field 4.
- Compare extracted output in Field 5 with original hidden input.

If extraction is empty or truncated, the platform likely normalized or removed some invisible code points.

---

## Technical Deep Dive

## Design Goals

- **Zero dependencies** for transparent auditing.
- **Cross-platform resilience** for imperfect copy/paste pipelines.
- **Compatibility-minded** with patterns used in the referenced umpox approach.
- **Readable code path** for future AI/human contributors.

## Unicode Characters Used

Inside `index.html`, the encoder/decoder currently uses:

- `U+200B` (ZERO WIDTH SPACE) => bit `1`
- `U+200C` (ZERO WIDTH NON-JOINER) => bit `0`
- `U+200D` (ZERO WIDTH JOINER) => byte delimiter (space between 8-bit chunks)
- `U+FEFF` (ZERO WIDTH NO-BREAK SPACE) => token separator

This forms an invisible “alphabet” for payload transport.

## Encoding Pipeline

`hiddenText` → UTF-8 bytes → binary string with byte spaces → zero-width token mapping.

High-level steps:

1. Convert hidden text to UTF-8 bytes (`TextEncoder`).
2. Represent each byte as 8-bit binary.
3. Join bytes with visible binary spaces.
4. Map binary chars to zero-width chars:
   - `'1'` → `U+200B`
   - `'0'` → `U+200C`
   - `' '` → `U+200D`
5. Join mapped tokens using `U+FEFF` separator.
6. Append the encoded payload to visible text.

## Decoding Pipeline

Decoding is intentionally **two-stage**:

### 1) Separator-aware path

- Filter stream to `[U+200B, U+200C, U+200D, U+FEFF]`.
- Split by `U+FEFF`.
- Map tokens back to binary (`1/0/space`).
- Parse bytes and decode UTF-8 with `TextDecoder`.

### 2) Fallback (separatorless)

Some platforms keep only parts of the invisible set (common issue in rich-text processors).  
Fallback logic:

- Ignore separators.
- Recover bitstream from `[U+200B, U+200C]` only.
- Group into full bytes (8-bit chunks).
- Decode best-effort UTF-8.

This was added to reduce failures after platform transformations (e.g., systems that strip markers/separators but keep some zero-width characters).

## Why this matters

A strict marker-based decoder can fail if that exact marker is removed/normalized by external systems.  
A stream-based decoder + fallback improves practical extraction rates.

---

## Known Limitations

- Not all platforms preserve zero-width characters equally.
- Some editors normalize Unicode and can destroy payload integrity.
- No cryptographic integrity checks (tamper detection) yet.
- No authentication/signing model.
- No formal automated test suite yet (manual and ad-hoc command-line checks used so far).

---

## Security / Ethics Notes

Zero-width techniques can be used for benign metadata embedding, QA, and research, but can also be abused (tracking, obfuscation).  
Use responsibly and in compliance with legal/policy requirements of your environment and platform terms.

---

## Developer Notes (for future AI/human contributors)

## File Layout

- `index.html`: full UI, styles, encode/decode logic.

## Core Functions (in `index.html`)

- `textToBinaryWithSpaces(text)`
- `binaryToText(binaryText)`
- `encodeHiddenText(hiddenText)`
- `decodeWithSeparator(encoded)`
- `decodeWithoutSeparatorFallback(encoded)`
- `decodeHiddenText(text)`
- UI wiring: `buildCombined`, `copyCombined`, `extractFromPasted`

## Extension Ideas

1. **Test Harness**
   - Add browser tests (Playwright/Cypress) and unit tests for codec logic.
   - Add roundtrip fixtures for multilingual payloads + emoji.

2. **Codec Profiles**
   - Introduce selectable encoding profiles:
     - umpox-compatible profile
     - marker-based profile
     - redundancy/error-correction profile

3. **Error Correction**
   - Add parity/checksum/CRC or Reed-Solomon style redundancy to survive character loss.

4. **Payload Framing**
   - Add explicit framing headers (version/profile/length/checksum).

5. **Interoperability Dashboard**
   - Add per-platform compatibility matrix (LinkedIn, X, Slack, Google Docs, Notion, etc.).

6. **Safer UX**
   - Add warnings for suspicious contexts.
   - Add “character preservation preview” utility.

## Suggested AI Handoff Prompt

If another AI continues this project, provide:

- goal (e.g., “increase LinkedIn survivability”),
- target platforms,
- acceptable payload loss tolerance,
- backward-compat requirements with current encoding.

Then ask it to:

1. keep existing behavior backward-compatible,
2. add automated tests first,
3. implement a versioned codec profile layer,
4. document migration paths in this README.

---

## Running Locally

Open directly:

- double-click `index.html`, or
- serve via a local static server if preferred.

Example (optional):

```bash
python3 -m http.server 8080
```

Then open `http://localhost:8080`.

---

## License

No explicit license file is currently included in this repository.
If you plan to distribute publicly, add a license (e.g., MIT) in `LICENSE`.
