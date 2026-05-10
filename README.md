# Rangeable RFC

[![Status](https://img.shields.io/badge/RFC-v2.0%20APPROVED-success)](RFC.md) [![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

A language-neutral specification for a generic, integer-coordinate, closed-interval set container.

`Rangeable<Element>` pairs hashable elements with the integer ranges over which they are active and answers three queries efficiently:

- **`getRange(of element)`** — merged disjoint ranges of a given element.
- **`subscript[i].objs`** — every element active at position `i`, in first-insert order.
- **`transitions(over: lo..hi)`** — open / close boundary events within a range.

Same-element overlapping or integer-adjacent ranges merge automatically; different elements coexist independently.

**v2 adds** Removal (`remove(e, start, end)`, `remove(e)`, `clear`, `removeRanges(start, end)`) and Set Operations (`union`, `intersect`, `difference`, `symmetric_difference`, in mutating + non-mutating pairs) — see RFC § 6.6 – § 6.13. v1 API is preserved; v2 is a strict superset.

## What you'll find here

- **[`RFC.md`](RFC.md)** — the normative specification (~2870 lines, v2.0), reviewed by an independent academic reviewer over four rounds (Round 1 REJECTED → Round 2 v1.0 APPROVED → Round 3 v2.0 spec APPROVED → Round 4 v2.0 implementation APPROVED). Covers API, semantics, data structures, pseudocode, amortised-complexity proofs (Φ-potential), edge cases, an 80-case normative test contract (#1 – #80), and 25+ academic references.

## Reference implementations

| Language | Repository | Notes |
|---|---|---|
| Ruby 3.2+ | [github.com/ZhgChgLi/RubyRangeable](https://github.com/ZhgChgLi/RubyRangeable) — `gem install rangeable` (v2.0.0) | 116 tests, ~5.5× speedup over brute-force |
| Swift 5.7+ | [github.com/ZhgChgLi/SwiftRangeable](https://github.com/ZhgChgLi/SwiftRangeable) — SPM tag `2.0.0`, iOS 12+ / macOS 10.14+ | 115 tests, COW value semantics |
| Python 3.10+ | [github.com/ZhgChgLi/PythonRangeable](https://github.com/ZhgChgLi/PythonRangeable) — `pip install rangeable` (v2.0.0) | 262 tests, frozen-dataclass value types, PEP 561 typed |
| TypeScript / JS (Node 18+) | [github.com/ZhgChgLi/JSRangeable](https://github.com/ZhgChgLi/JSRangeable) — `npm i rangeable-js@2.0.0` | 275 tests, dual ESM + CJS, `keyFn`-based equality |
| Kotlin / JVM 11+ | [github.com/ZhgChgLi/KotlinRangeable](https://github.com/ZhgChgLi/KotlinRangeable) — `implementation("com.github.ZhgChgLi:KotlinRangeable:v2.0.0")` (JitPack) | 111 tests, Kotlin 2.2, data-class equality |
| Go 1.22+ | [github.com/ZhgChgLi/GoRangeable](https://github.com/ZhgChgLi/GoRangeable) — `go get github.com/ZhgChgLi/GoRangeable@v2.0.0` | 101 tests, generics over `comparable`, stdlib only |

All six implementations cross-verify against a shared 200-op / 126-probe / 20-set-op fixture; their outputs are byte-identical.

## Real-world consumers

- **[`ZMediumToMarkdown`](https://github.com/ZhgChgLi/ZMediumToMarkdown)** (v3.7.0+) uses `RubyRangeable` to render Medium GraphQL paragraphs into Markdown. Replacing the previous `O(L · m)` per-paragraph walk with `Rangeable.transitions` yielded a **2.23× end-to-end speedup** (55% reduction in render time) while keeping all 329 existing tests byte-identical.
- **AVPlayer non-contiguous byte-range cache** (case study, [RFC §1.3.1](RFC.md)) — a 2021 article on `AVURLAsset` + `AVAssetResourceLoaderDelegate` ([source post](https://en.zhgchg.li/posts/6ce488898003/), [`ZPlayerCacher`](https://github.com/ZhgChgLi/ZPlayerCacher)) explicitly defers non-contiguous `Data` merge as "too complex". `Rangeable<CacheToken>` over a sparse `FileHandle` collapses both the read-side stab ("which prefix of `[lo, hi]` is on disk?") and the write-side merge ("union the new bytes with all prior runs, including integer-adjacent ones") into `transitions(over:)` + `insert(...)`, with no manual merge code.

## Status

| Round | Verdict | Notes |
|---|---|---|
| 1 | REJECTED | 6 MUST-FIX items (Optional<Int> sentinel, Int.min underflow, Φ-potential proof gaps, cleaner-variant pseudocode, missing lemmas, lazy-build concurrency) |
| 2 | **APPROVED v1.0** | All Round 1 MUST-FIX resolved; v1.0 normative spec frozen. |
| 3 | **APPROVED v2.0 spec** | Removal + Set Operations promoted from v1 deferred (§14.1, §14.9) to v2 normative (§6.6 – §6.13). Adds eager-pruning (§4.10), sweep-line algorithms, and 60 new test cases (#21 – #80). |
| 4 | **APPROVED v2.0 final** | All 6 reference implementations conform byte-identically to the v2 fixture (200 ops + 126 probes + 20 set-ops). 4 in-line RFC patches; 0 MUST-FIX. See [`RFC.md` § 16](RFC.md) for the full review history. |

## License

MIT © [ZhgChgLi](https://github.com/ZhgChgLi)

# Buy me a beer ❤️❤️❤️

[![Buy Me A Beer](https://github.com/user-attachments/assets/63f01edf-2aa5-4d91-8f8a-861e5b6b4feb)](https://www.paypal.com/ncp/payment/CMALMPT8UUTY2)

[**If this project has helped you, feel free to sponsor me a cup of coffee, thank you.**](https://www.paypal.com/ncp/payment/CMALMPT8UUTY2)
