# Upstream integration assessment — PdfCraft Android

Assessed 26 October 2026. Priority: performance, then useful editing/export features, while preserving Kotlin/Jetpack Compose screens, navigation and Android storage integration.

## Baseline and provenance

- Android source: [artcraft-android/pdfcraft-android](https://github.com/artcraft-android/pdfcraft-android), main at `723fb924e19810d3674e963985728a4ec1ef534d`.
- Desktop engine upstream: [storytold/pdfcraft v0.14.0](https://github.com/storytold/pdfcraft/releases/tag/v0.14.0), `5a78b223871add8e3883856205cc653d741e0e94`, released 25 October.
- Android's Cargo workspace still declares 0.6.0. Its source is a mixed snapshot: 131/211 shared crate Rust files match v0.7.1 exactly (62.1%); only 32/191 match v0.14.0 (16.8%). These are file identity counts, not functional completeness or a precise engine-version claim.
- The Android bridge uses UniFFI 0.32.2, typed document/render/text helpers and a generic automation-tool interface. It renders the viewer through raw RGBA pixels; PNG export is a separate path.
- The six Stage 1 changes below are absent: the old hot-loop checks, linear glyph lookup, repeated space measurements, RGBA-only PNG encoder, unbalanced OCR graphics state and unsafe paragraph-width clamp remain in the Android source.
- No application implementation changes, APK, Android benchmark or device validation are part of this assessment.

Risk means likelihood and breadth of regressions when backporting to this Android snapshot. It does not mean severity of the bug being fixed. Low risk still needs build and regression verification.

## Stage 1 — low risk: recommended first

Backport only the necessary production changes and matching regression coverage; do not copy whole newer files, desktop screens, roadmap changes or unrelated dependency updates.

| Priority | Upstream change | In ordinary use | Evidence and limits |
|---|---|---|---|
| 1 | [#3955: cache the text-debug flag](https://github.com/storytold/pdfcraft/pull/3955), merge `498a7d60b93d62061a724f4ae36b197e03cb7c09` | Stop checking the process environment for every character. Helps text extraction and paths used by search/read-aloud. | Upstream text extraction: 623 → 551 ms on 391 pages (about 12% less time); other files improved less or more. One small render change; debug flag becomes fixed after first use. |
| 2 | [#3958: cache CFF glyph names](https://github.com/storytold/pdfcraft/pull/3958), merge `5f256d53fd7bec23b289e6b94d61cff3b4208f7e` | Stop searching an entire font's character list repeatedly. Speeds text processing for PDFs using the affected fonts. | One 65-page PDF: extraction 83 → 57 ms (~31% less time), structured open 207 → 187 ms. Other tested PDFs unchanged. Small lazy font cache, shared across clones; preserve first-match behaviour. |
| 3 | [#3297: measure font spaces once per paragraph rewrite](https://github.com/storytold/pdfcraft/pull/3297), merge `16f36810cc6869b1957b505e4950b926f36574f9` | Reduces pauses while changing text in paragraphs whose subset fonts have no space character. | Upstream examples: 460 → 71 ms and 255 → 16 ms per keystroke. Output unchanged in its tests. Cache is local to one rewrite, avoiding complex invalidation. Android's editing flow must be checked separately. |
| 4 | [#2425: faster PNG encoding](https://github.com/storytold/pdfcraft/pull/2425), merge `8f3456f29c9fbcb94a46e91fd9cdf58588a323a0` | Faster page-to-PNG export without discarding image detail. Opaque pages use RGB; transparency remains supported. | Upstream encoder examples: 805 → 167 ms, 70 → 40 ms, 179 → 64 ms (~1.8–4.8× faster). Sizes were close, not identical. Does not accelerate the raw-RGBA viewer itself. Check Android PNG consumers accept RGB and preserve the viewer's RGBA contract. |
| 5 | [#3033: restore graphics state after OCR](https://github.com/storytold/pdfcraft/pull/3033), merge `15af2860f1c2a3596530824a1cbcd297d9a46495` | Prevents later added text from becoming invisible after recognising a scan. | Two production byte-string changes plus a regression test. Does not repair every PDF already saved with the defective layer and does not add OCR languages. |
| 6 | [#4110: cap oversized paragraph width safely](https://github.com/storytold/pdfcraft/pull/4110), merge `75207dc8ed02ec91bc6bed35574f113b18452aba` | Avoids an editing crash when a PDF contains exceptionally large or transformed text. | A narrow clamp fix and synthetic regression coverage. Normal-size text keeps existing behaviour. |

Stage 1 acceptance: targeted Rust tests, Android bridge tests, regenerated bindings if required, ARM64 APK assembly, and open/edit/save/reopen checks. Specifically compare extracted text, PNG pixels/transparency, and OCR followed by added text. Check the existing release workflow and signing requirements before attempting publication. Keep the APK release separate from this assessment.

## Stage 2 — medium risk: strongest subsequent value

| Order | Candidate | Benefit | Why integration needs more care |
|---|---|---|---|
| 1 | [#2427: parse each object-stream index once](https://github.com/storytold/pdfcraft/pull/2427) | Removes severe stalls on PDFs packing many objects together. Upstream affected-file render: 37 s → 0.5 s; text: 40 s → 1.9 s. | Changes core parser state, locking and cached malformed-stream handling. Test malformed/encrypted files, document repair, memory retention and both supported ARM sizes. The dramatic gains concern unusual files. |
| 2 | [#2436: sampled functions, image caching across zooms and meshes](https://github.com/storytold/pdfcraft/pull/2436) | Faster gradients and scanned-page zooming; avoids keeping another full image decode for each zoom level. Example gradient page: 1,340 → 485 ms upstream. | Several vendored rendering patches. Check masks, JPEG 2000 size-dependent decoding, colours and cache memory on Android. Pixel comparison is needed. |
| 3 | [#1434: defer edit-marker/link scans](https://github.com/storytold/pdfcraft/pull/1434) | Opens ordinary documents without scanning editing structures until needed. | Changes engine access patterns and cache invalidation after edits/undo/comments. Android consumers must be audited; desktop view changes cannot simply be copied. Savings depend on document and which Android tools are requested immediately. |
| 4 | [#1642: correct CMYK/YCCK image extraction](https://github.com/storytold/pdfcraft/pull/1642) | Exported pictures retain intended colours instead of appearing as negatives. | Adds a direct JPEG decoder dependency and changes colour/Decode handling. Test CMYK, RGB, masks and output compatibility. |
| 5 | [#4214: column-aware text order](https://github.com/storytold/pdfcraft/pull/4214) | Copy/search/read-aloud can follow columns more naturally in papers and reports. | Changes geometric reading-order rules. Test tables, mixed-size form rows, headings and footnotes; baseline-aligned columns remain a known limitation. |
| 6 | [#3488: form fonts as separate PDF objects](https://github.com/storytold/pdfcraft/pull/3488) | Text entered into affected forms remains visible when opened in Acrobat. | Alters saved PDF object structure and includes special treatment of signed forms. Verify round trips in other viewers and existing signatures. |
| 7 | [#4230: copy/paste added text boxes and images](https://github.com/storytold/pdfcraft/pull/4230), with [#4258](https://github.com/storytold/pdfcraft/pull/4258) | Reuse added text and images on another page/document with their style and placement. | New commands and Android controls are needed. Cross-document copies may lose colour profiles; some lossless CMYK images become RGB. Image matching can be slow with hundreds of same-sized scans. |

The first three offer the most attractive further performance gains. Form/export correctness and column order should follow according to use. Copy/paste adds a visible feature but requires a deliberate Android UI addition.

## Stage 3 — high integration risk

| Candidate | Benefit | Recommendation |
|---|---|---|
| Shared work/decode limits: [#4114](https://github.com/storytold/pdfcraft/pull/4114), [#4121](https://github.com/storytold/pdfcraft/pull/4121), and related v0.14 fixes | Prevent small or malformed documents/forms from multiplying work or consuming excessive memory; useful for phone reliability. | High priority despite higher risk. Port as a coherent, dependency-reviewed group rather than scattered limit constants. #4114 alone touches 32 files across multiple crates and changes shared decoding APIs. Verify legitimate complex documents still work and failures leave documents intact. |
| Redaction/signature/encryption corrections: [#4244](https://github.com/storytold/pdfcraft/pull/4244) | Removes more hidden material correctly and classifies signed-document changes more accurately. | Integrate with focused adversarial and cross-viewer fixtures. Do not infer complete redaction safety: this upstream PR explicitly leaves two design findings unresolved. |
| [#1609: recognise a scanned page when Edit is enabled](https://github.com/storytold/pdfcraft/pull/1609) | Reduces the manual steps needed to edit a scan. | Valuable later. This is largely desktop UI orchestration, so implement Android progress, cancellation, one-run-per-page tracking and undo deliberately. It still uses the existing limited-language recogniser; it is not multilingual OCR or full scan reconstruction. |
| Replace the entire shared engine with v0.14.0 | Broadest feature and compatibility catch-up. | Avoid as the first integration. Many APIs, dependencies and assumptions have changed. Adapt the Rust bridge around upstream changes while retaining Compose screens and Android-specific behaviour. |

## Recommended sequence

1. Ship the six low-risk backports as one reviewable integration branch.
2. Tackle parser/image performance improvements in separate reviewable changes.
3. In parallel in the project schedule (not by blind merging), plan coherent resource-limit and document-integrity work; high risk is not a reason to leave it indefinitely.
4. Add form/export/text-order improvements, then Android controls for new features.
5. Consider a complete engine refresh only after the adapter and regression fixtures are established.

Upstream measurements above were made on named desktop machines/documents by upstream contributors. They are evidence for selecting work, not predicted Android speed-ups, additive percentages or a promised overall performance increase.
