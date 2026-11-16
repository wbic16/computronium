# Computronium — Chromium Duplication Census

**The budget map for rewriting Chromium from first principles in 5M lines (Will Bickford, 2026-11-16).**

- Tree measured: Chromium main repo @ `e4f4ccfcf38bc572208fa1066c7f7927ecc7f801` (depth-1) plus the full public DEPS stratum, fetched and counted on aletheia-core, 2026-11-16.
- Method in one line: every number below is measured on the actual tree (file index + SHA-256, tokei, a normalized-token function/type extractor, line-sketch near-clone detection, constant fingerprinting); anything not directly measured is labelled *estimate*.
- Raw tables: `tables/` beside this file (40 CSV/JSON files). Tool sources: `tools-src/`. The full tree as one phext: `~/chromium-census/chromium.phext` on aletheia-core.

---

## 1. The tree, and the reconciliation against 51M

| Stratum | Files | Bytes | Physical lines | Code-extension lines |
|---|---|---|---|---|
| Main repo, first-party | 257,405 | 2,454,357,554 | 42,293,914 | 34,144,931 |
| Main repo, `third_party/` (vendored) | 260,517 | 2,027,449,580 | 25,257,793 | 14,804,344 |
| DEPS stratum (253 public repos) | 915,367 | 16,655,457,836 | 178,829,592 | 100,311,703 |
| **Whole tree** | **1,433,289** | **21,137,264,970** | **246,381,299** | **149,260,978** |

Tokei (code / comments / blanks), the convention closest to how "lines of code" is usually quoted:

| Scope | Code | Comments | Blanks | All lines |
|---|---|---|---|---|
| Main repo | 42,068,191 | 7,703,753 | 7,036,072 | 56,808,016 |
| — of which first-party | 27,308,567 | 4,584,786 | 5,074,014 | 36,967,367 |
| — of which vendored `third_party/` | 14,759,624 | 3,118,967 | 1,962,058 | 19,840,649 |
| DEPS stratum | 98,547,833 | 31,898,979 | 11,857,289 | 142,304,101 |
| **Whole tree** | **140,616,024** | **39,602,732** | **18,893,361** | **199,112,117** |

**Reconciliation vs Will's 51M.** The 51M figure sits *between* the two main-repo conventions — code only (42.07M) and all lines (56.81M) — and within 2.4% of main-repo code+comments (49.77M). No measured convention contradicts it as a main-repo-centred count. What 51M does **not** include is the DEPS stratum as a separate tree: the 253 sibling repos that `gclient` materialises around the checkout carry another **98.5M tokei code lines**. Counted whole, the tree Computronium must replace is **140.6M code lines — 2.76× the 51M figure**.

Budget ratios for the 5M target, by convention: 1/8.4 of main-repo code · 1/11.4 of main-repo all-lines · 1/28.1 of whole-tree code · 1/29.9 of whole-tree code-extension lines. Which convention the 5M is scored against is Will's scoping call (open since 02:04 CST); this census reports against all of them.

Top languages, main repo (tokei code): C++ 18.47M · C headers 4.27M · JSON 3.09M · Rust 2.56M · Java 2.49M · HTML 2.39M · JavaScript 2.22M · Objective-C++ 1.43M · Python 1.33M · TypeScript 1.33M.

---

## 2. Step 1 — the whole tree in one phext (Will, 02:32 CST)

The entire tree — main repo + DEPS stratum — is encoded as **one phext**: `~/chromium-census/chromium.phext` on aletheia-core, written through libphext `implode` (the reference implementation; no hand-rolled serialiser).

- **Scheme.** One index scroll at `1.1.1/1.1.1/1.1.1`: per file — coordinate, unified path, size, SHA-256, encoding flags. Files follow in sorted unified-path order at `1.1.1/1.1.1/<chapter>.<section>.<scroll>`, 100×100 files per chapter, 144 chapters. A file is stored PLAIN only if it is valid UTF-8 and contains none of the phext break bytes (0x01–0x03, 0x07, 0x17–0x1F set); otherwise it is base64-hoisted and flagged. Empty files carry a single-NUL placeholder scroll (phext scrolls cannot be empty).
- **Result.** 1,433,290 scrolls (1,433,289 files + index). Phext size **24,259,355,426 B** vs tree 21,137,264,970 B = **1.148×** (the growth is base64 inflation on binary payloads). Encode: 205.9 s ≈ 103 MB/s stored.
- **Round-trip gate.** The produced phext was parsed back with libphext's `phokenize` (chapter-sliced) and every scroll's SHA-256 compared against the source manifest: **1,433,290 / 1,433,290 scrolls, 0 mismatches**, 607.3 s. Cross-check with a second, independent parser (the Python dead-reckoning parser) on chapter 100: 10,000 / 10,000 manifest hashes matched.
- **The gate earned its keep.** The first encode silently emptied 1,920 legacy-encoding test files (Big5/EUC-JP/ISO-8859 HTML — invalid UTF-8 but no NUL and no break bytes, so flagged "plain"; the UTF-8 conversion then failed to empty). Verification caught it (464,341 cascade mismatches); the fix made the encoding decision byte-driven rather than flag-driven, and the re-encode verified clean. A serializer that only ever reads its own output would have shipped the corruption.

---

## 3. The pools

### (a) The dependency pool — destination: one chrome.phext

*Method: DEPS parsed from the main repo (253 public git-like entries; `chrome-internal` excluded by design); each repo fetched at its pinned revision and indexed as its own stratum; vendored `third_party/` in the main repo grouped by top-level project directory.*

- DEPS stratum: **100,311,703 code-extension lines** across 253 repos (187 repos matched to ≥1 code file in the per-project accounting; the remainder are small/resource-only checkouts). This pool is treated as **fully recoverable in principle**: Computronium carries one canonical copy of each dependency, in the one chrome.phext.
- Largest deps (code-extension lines): **WebKit 29,729,548** · SwiftShader 6,243,220 · tflite 5,807,806 · perl 4,804,519 · v8 3,794,623 · xnnpack 3,396,528 · dawn 2,845,585 · ffmpeg 2,265,288 · skia 1,906,224 · jetstream (main) 1,754,108.
- Vendored inside the main repo: **306 projects, 14,807,715 code-extension lines** under `src/third_party/` — with two caveats: `blink` (5,467,834 lines) sits under that path but is first-party, and `rust/` (3,375,996) is toolchain vendoring.
- **Multi-copy upstreams** (same project, several copies in the tree): jetstream **×4 copies, 5,262,324 lines total** (three DEPS checkouts — main/v2.2/v3.0 — plus one vendored) · xnnpack ×2, 3,470,813 · speedometer ×6, 909,740 · boringssl ×2, 966,010 · glib ×2, 847,650 · litert ×2, 352,266 · devtools-frontend ×2, 395,262. And **double-homed** deps — present both as a DEPS repo *and* vendored in-tree: libaom (×2 total, 672,676 lines), libvpx (×2, 521,814), breakpad (×2, 409,090), libipp.

### (b) Generated code

*Method: two independent detectors — content markers (`generated by`, `do not edit`, protobuf/upb/grpc/mojo/turbine fingerprints in the first 60 lines) and filename patterns (`*.pb.*`, `*_generated*`, `*.mojom*`, `gen/` paths, …).*

- Marker-based: **52,870 files / 16,731,403 lines**.
- Filename-pattern-based: 43,195 files / 6,463,730 lines.
- In a phext world this code is not carried as source at all — it is regenerated from the schema that lives in the phext. The function-identity pool below shows where much of it lands: the largest C++ duplicate-function clusters are generated upb/grpc accessors.

### (c) Token-level clones

*Method: per file, whitespace-normalised lines and 50-token windows (tokens = identifiers/keywords/literals/operators) counted exactly; duplicates are repeats within the same file. Cross-file token clones are covered by pools (g) and §4.*

- **98,874,075 of 246,467,230 physical lines (40.1%) are whitespace-normalised repeats of a line already seen in the same file.**
- **83,406,592 duplicate 50-token windows** within-file (of 898,737,587 total windows).
- The DEPS third-party stratum is the extreme: 69,751,727 of its 174,279,004 lines (40.0%) repeat within their own file — generated tables, vendored test corpora, and bundled assets dominate.

### (d) Platform-variant multiplication

*Method: filename families differing only by platform suffix (`_win`, `_mac`, `_linux`, `_android`, `_ios`, `_posix`, `_x11`, …) or platform directory, code files only.*

- **8,050 variant files / 1,775,507 lines in 833 families; 200,656 lines exist only as the 2nd..nth variant** beyond the largest file of each family.
- Largest: `web_app_integration_browsertest` (4 variants, 14,131 lines), sqlite `os.c` family (14,039), `render_widget_host_view` (prod + unittest families, ~20,000 lines combined).

### (e) The test pool — the verification contract

*Method: directory/filename conventions (`test`, `tests`, `unittest`, `browsertest`, `Test.java`, WPT layout, …) over code-extension files; `testdata/` counted separately. Physical lines.*

| Stratum | Test lines | Test files | Production lines |
|---|---|---|---|
| Main repo, first-party | 14,631,849 | 41,561 | 19,505,325 |
| Main repo, vendored | 2,482,199 | 10,209 | 12,321,893 |
| DEPS stratum | 23,042,833 | 157,656 | 77,232,721 |
| **Total** | **40,156,881** | **209,426** | **109,059,939** |

- Tests are **26.9%** of all code-extension lines. First-party code is the most test-dense: 42.9% of first-party code-extension lines are test lines.
- Test *data* is its own mass: across all file types, 6,672 files / 3,753,911 lines / **883,546,153 bytes** live under `testdata/` paths (only 44,158 of those lines are in code-extension files — the rest is fixtures, media, and golden files).
- Per Will's ruling this pool is **not** a recovery target: the test corpus is the verification contract that makes dedup safe. It is stated in full so the budget math stays honest.

### (f) The syntax-delta pool — recoverable by design

*Will, 02:39 CST: "ensure that we dedup any syntax deltas by design." Frame: one canonical syntax; the deltas between per-subsystem dialects are not content.*

Measured components:

- **Boilerplate mass**: license/header comment blocks (detected header blocks at file top) = **4,092,895 lines across 539,057 code files**.
- **Within-file repetition** (pool c): 98.87M whitespace-normalised duplicate lines tree-wide — repetition that survives whitespace changes, i.e. content-level; the formatting-only share inside it is what a canonical formatter/syntax erases without any factoring.
- **Compression attribution** (main repo, zstd −19 −−long=27: 4,481,807,134 B → 1,140,726,847 B, 3.93×, 3,341,080,287 B saved): the main repo's within-file repeated lines are 27,307,547 lines ≈ 2.15 GB at the tree's 78.9 B/line average — **≈65% of everything compression removed is lines repeating inside their own file**. Boilerplate headers add ≈2.9M lines (≈0.23 GB, mostly cross-file, so largely disjoint). Cross-file clones (pool g: 22.74M lines sit in duplicate-function groups tree-wide) account for much of the rest. The residual — what compression removes that no measured clone pool explains — is language regularity itself: keyword frequency, idioms, per-subsystem style dialects. **That residual is the syntax-delta pool proper: it cannot be deduplicated as content, only collapsed by design — one canonical syntax, which is what phext-everywhere provides.**
- Net bound: the main repo's effective unique content is **≈10.7M lines** (compression bound, optimistic — compression also eats syntax regularity, so true semantic uniqueness is somewhat higher). The distance between the dedup pools' reach and this bound is precisely the syntax/design space.

### (g) Function identity — same behaviour, same function

*Method: function bodies extracted (C-like brace matching for C/C++/Java/JS/TS/Rust/Go/C#; indentation for Python), normalised — identifiers alpha-renamed in order of appearance (seeded by the signature's own identifiers, so renaming a function does not change its hash), string/char/number literals folded to S/C/N — then hashed (FNV-1a over the normalised token stream). Two functions are identical iff their normalised body hashes match. 3,948,534 functions extracted; 70,397,319 body lines; 2,708,347 functions have ≥15 body tokens.*

At ≥15 body tokens:

- **316,712 duplicate groups; 1,059,660 duplicate functions; 16,591,012 recoverable lines** (group lines minus the largest member). 22,741,518 lines sit in duplicate groups in total.
- **Will's exact case — same body, different names: 83,833 groups / 562,374 functions / 4,909,277 lines.**
- Same body under the same name but different parameter lists: 1,103 groups / 14,738 functions.
- At ≥30 tokens (substantial functions only): 235,437 groups / 14,859,579 recoverable lines; multi-name 45,923 groups / 3,587,130 lines.

Largest clusters are dominated by (i) generated upb/grpc binding accessors — thousands of identical `GetStructDef`-style bodies, one per message type — and (ii) vendored JS benchmark bundles (the Octane `mandreel.js` bundle alone appears as a 12-copy, 1.84M-total-line "function"). Both are the same phenomenon at different scales: a body copied because there was no cheaper way to reuse it.

---

## 4. Reuse-failure prospecting (Will, 02:41 CST)

Seeded classes first, then the classes the tree revealed on its own.

**Exact duplicate files** (SHA-256 over all 1,433,289 files): **129,184 groups; 224,186 duplicate files; 27,068,586 duplicate lines; 3,643,008,089 duplicate bytes.** The top of the table is jetstream's three copies (a 67 MB ONNX model ×3, BabylonJS source maps ×3).

**Copy-pasted files that drifted** (48-line-sketch MinHash-style candidates, verified by true line-set Jaccard ≥0.85): 594,260 candidate pairs; verification was capped at 4,000 pairs — **≥4,000 verified near-clone pairs, forming 776 clusters over 2,190 files with 131,557 recoverable lines. This is a floor, not a total** (the cap, not the tree, is the limit).

**Versioned duplicate directories** (same project under version-named dirs): 567 families, 7,839,053 lines, of which **4,600,124 lines are the older copies** — jetstream and speedometer version stacks dominate.

**Parallel implementations of the same thing**, two lenses:
- *Constant fingerprints* (files embedding the algorithm's constants, src+deps): SHA-256 initial state in **103 files**, SHA-256 K-table in 51, SHA-1/MD5 initial state in 68, SHA-1 H0 in 42, CRC32 polynomial in **55**, SHA-512/Blowfish π digits in 17 — and the **base64 alphabet literal in 322 files**.
- *Filename lens* (conservative): URL parsing 464 files / 88 distinct roots · JSON 212 / 47 · JPEG 173 / 18 · XML 114 / 29 · ZIP 82 / 22 · PNG 76 / 11 · UTF conversion 63 / 10 · base64 19 / 10 · MD5 19 / 4 · SHA-1 11 / 4 · CRC32 4 / 2.

**Dead code** (lower-bound *estimate*): BUILD.gn analysis of the main repo — 8,176 GN files referencing 180,082 source paths; **9,006 files (1,732,217 lines) are referenced by no BUILD.gn target** under basename+path matching. Some are legitimately reached by other means (scripts, resources), so this is an orphan-candidate pool, not a verdict. Separately: 1,297 files (302,967 lines) are compiled into multiple targets.

**Forwarding wrappers** (*estimate*, name heuristic, ≤40-line files named `*_wrapper`/`*_bridge`/`*_adapter`/`*_proxy`/`*_shim`/`*_forwarder`): **2,911 files / 78,076 lines**.

**Duplicated data tables** (exact-duplicate files ≥10 KB in data extensions): 1,117 groups / **278,948,260 duplicate bytes** — Unicode/ICU/zoneinfo-class tables copied per consumer.

**Hand-rolled utility reimplementations** (name heuristic: common utility names — string split/join/trim, case conversion, clamp, gcd, base64/hex codecs — defined outside their canonical homes in `base/`, WTF, `std::`): **1,165 functions / 17,655 body lines** (lower bound; the fingerprint lens above is the stronger evidence for codecs specifically).

**Similar types** (Will, 02:43 CST — field+method multiset Jaccard ≥0.75 over 796,780 parsed class/struct declarations, rare-element bucketing): **49,631 clusters covering 163,222 types; 8,337,981 span lines; 5,345,065 recoverable lines** if each cluster kept one canonical declaration. 76,487 type names are declared in more than one file. Exemplars: the Vulkan `Device` handle set declared in skia's bundle, SwiftShader's bundle, *and* vulkan-headers (29,643 span lines ×3); `basic_json` in icu's toolutil copy and both nlohmann distributions; `cpuinfo_mock_file` re-declared 30 times in cpuinfo's own tests.

**Parameter-delta methods** (Will, 02:43 CST — same-name groups; token-bag containment ≥0.90 with the rarest-token candidate keying; bodies or parameter lists must differ): **2,910 families / 15,784 member functions / 788,015 shorter-body lines recoverable** by delegation to the longest variant. Reported separately, because it is a different animal: **489 macro/test-harness families / 11,730 members / 2,860,267 lines** — `TEST_P`-style parameterised-test bodies and annotation-harness extractions, i.e. one harness pattern wearing thousands of name tags.

### Classes the tree revealed (not on the seed list)

1. **A second browser engine.** WebKit — 29.7M code lines, the single largest "dependency" — is a complete alternative engine vendored inside the first one's tree (for iOS). The largest duplication in Chromium is not a function or a file; it is an entire second implementation of the product.
2. **Benchmarks as a duplication engine.** jetstream ×4 copies, speedometer ×6, Octane bundles ×12 — performance suites are vendored whole, per version, per consumer, and their payloads (ONNX models, source maps) top the exact-duplicate byte table.
3. **A library duplicated by its own distribution format.** nlohmann_json exists as multi-header *and* single-header copies inside the same dep; the single-header amalgamation is a second full copy by construction.
4. **Headers triplicated by bundling.** Vulkan headers live in vulkan-headers, inside skia's third_party, and inside SwiftShader's — each bundler re-vendors what it builds against.
5. **Style-dialect files as clones.** `.clang-format` / `.clang-tidy` configs are near-clones across hundreds of projects (Jaccard 1.00 pairs) — per-project syntax dialects, carried as files. Pool (f) made visible as artifacts.
6. **Test-harness patterns masquerading as functions.** The 2.86M-line macro bucket above: parameterised-test and mock patterns repeated per suite because the harness offers no cheaper reuse.

---

## 5. The budget map — ranked against 5M

Pools overlap (a jetstream copy is simultaneously an exact-duplicate file, a versioned directory, a multi-copy dep, and generated/bundled content). **The figures below must not be summed.** They are ranked by size to show where the mass is; the honest totals are the bounds at the end.

| # | Pool | Measured size | Nature |
|---|---|---|---|
| 1 | DEPS stratum → one chrome.phext | 100.31M code-ext lines carried as 253 repos | consolidation |
| 2 | Test corpus | 40.16M lines | **kept** — verification contract |
| 3 | Second engine (WebKit, inside #1) | 29.73M lines | design elimination |
| 4 | Exact duplicate files | 27.07M dup lines / 3.64 GB | dedup |
| 5 | Within-file repeated lines (syntax-delta mass) | 98.87M lines tree-wide (40.1%) | design (canonical syntax) |
| 6 | Generated code | 16.73M lines (marker) | regenerate from schema |
| 7 | Function identity (≥15 tok) | 16.59M recoverable lines | dedup by behaviour |
| 8 | Similar types | 5.35M recoverable lines | dedup by structure |
| 9 | Versioned older copies | 4.60M lines | dedup |
| 10 | Boilerplate headers | 4.09M lines | design (canonical syntax) |
| 11 | Test-harness macro families | 2.86M lines | harness design |
| 12 | BUILD.gn orphan candidates | 1.73M lines (*estimate*, lower bound) | dead-code removal |
| 13 | Parameter-delta methods | 0.79M recoverable lines | dedup by delegation |
| 14 | Platform extra variants | 0.20M lines | factoring |
| 15 | Near-clone drifted files | ≥0.13M lines (floor; verification capped) | dedup |
| 16 | Forwarding wrappers | 0.08M lines (*estimate*) | factoring |
| 17 | Utility reimplementations | 0.018M lines (lower bound) | dedup |

**The bounds.** Deduplication alone does not reach 5M, and the census can say exactly why. The compression bound puts the main repo's unique content at ≈10.7M lines of its 42.07M code lines (25%). Applying the same uniqueness ratio to the whole tree's 140.6M tokei code lines gives **≈35M unique lines (estimate)** — the floor that pure dedup approaches. The remaining ~30M between that floor and the 5M budget is not duplication in any pool above; it is:

- the test corpus, if tests count toward the budget (40.16M lines — the single largest scoping question);
- wholesale design duplication no clone detector sees as clones — a second engine, per-subsystem protocol/format/state implementations that are *similar in purpose* but not in tokens;
- generated code carried as source instead of schema;
- and the syntax-delta residual — per-subsystem dialects and bespoke formats whose cost is paid in every file, visible only as compression's unexplained remainder.

That remainder is the Computronium thesis stated as a measurement: **the pools are, in large part, the cost of every subsystem carrying bespoke state and formats — the cost phext-everywhere collapses.** Dedup recovers the copies. Design elimination — one engine, one canonical copy of each dep in one chrome.phext, generated code as schema output, one canonical syntax, function identity by behaviour — is what closes the rest of the distance to 5M. Whether it closes *all* of it depends on Will's open scoping calls: what counts toward the 5M (tests? deps' unique content?), the subsystem scope, and the compatibility target.

---

## Appendix — methods, caveats, reproducibility

- **Counting.** Physical lines = `\n` count. Code-extension lines = physical lines in a fixed code/data extension set (597,978 files). Tokei 12.1.2 for code/comment/blank splits. The censusprobe extractor (Rust, sources in `tools-src/`) produced the file indexes, function/type extractions, and line/window statistics; pool scripts are in `tools-src/` (pools1/2/2a/2b/2c/3, tokei_parse).
- **Extractor limits.** The function extractor is a brace-matching parser, not a compiler front-end: it undercounts macro-heavy C++ and treats bundled JS files' top-level wrappers as single giant functions (visible at the top of `function-clusters-top.csv`). Python extraction replicates the C-like normaliser exactly (asserted by self-test). A thread panic in the first deps-third-party analysis (an indexing bug at a file-final signature) was found, fixed, and the stratum re-analysed in full; all pool figures here are from the completed runs.
- **Near-clone verification cap.** 4,000 verified pairs maximum per run; the pool is reported as a floor.
- **BUILD.gn orphans** are candidates under basename+path matching — a lower-bound estimate, not proof of deadness.
- **Compression** was measured on the main repo only (first census leg): zstd −19 −−long=27 over the payload tar; gzip −9 gives 2.95×. The DEPS stratum was not compression-measured; the whole-tree uniqueness figure in §5 is an extrapolation and is labelled as such.
- **The phext** (`~/chromium-census/chromium.phext`, hub) is the census's Step-1 artifact: 1,433,290 scrolls, verified byte-exact against the source manifest by the reference parser and cross-checked by an independent parser. Manifest: `~/chromium-census/manifest.tsv` (hub).
