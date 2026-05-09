# Rangeable RFC

[![Status](https://img.shields.io/badge/RFC-v1.0%20APPROVED-success)](RFC.md) [![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

A language-neutral specification for a generic, integer-coordinate, closed-interval set container.

`Rangeable<Element>` pairs hashable elements with the integer ranges over which they are active and answers three queries efficiently:

- **`getRange(of element)`** — merged disjoint ranges of a given element.
- **`subscript[i].objs`** — every element active at position `i`, in first-insert order.
- **`transitions(over: lo..hi)`** — open / close boundary events within a range.

Same-element overlapping or integer-adjacent ranges merge automatically; different elements coexist independently.

## What you'll find here

- **[`RFC.md`](RFC.md)** — the normative specification (~1800 lines), reviewed by an independent academic reviewer over two rounds (REJECTED → APPROVED). Covers API, semantics, data structures, pseudocode, amortised-complexity proofs (Φ-potential), edge cases, a 23-case normative test contract, and 25 academic references.

## Reference implementations

| Language | Repository | Notes |
|---|---|---|
| Ruby 3.2+ | [github.com/ZhgChgLi/RubyRangeable](https://github.com/ZhgChgLi/RubyRangeable) — `gem install rangeable` | 29 tests, ~5.5× speedup over brute-force |
| Swift 5.7+ | [github.com/ZhgChgLi/SwiftRangeable](https://github.com/ZhgChgLi/SwiftRangeable) — SPM, iOS 12+ / macOS 10.14+ | 38 tests, COW value semantics |
| Python 3.10+ | [github.com/ZhgChgLi/PythonRangeable](https://github.com/ZhgChgLi/PythonRangeable) — `pip install rangeable` | 117 tests, frozen-dataclass value types, PEP 561 typed |
| TypeScript / JS (Node 18+) | [github.com/ZhgChgLi/JSRangeable](https://github.com/ZhgChgLi/JSRangeable) — `npm i rangeable-js` | 117 tests, dual ESM + CJS, `keyFn`-based equality |
| Kotlin / JVM 11+ | [github.com/ZhgChgLi/KotlinRangeable](https://github.com/ZhgChgLi/KotlinRangeable) — `implementation("com.github.ZhgChgLi:KotlinRangeable:v1.0.0")` (JitPack) | 32 tests, Kotlin 2.2, data-class equality |

All five implementations cross-verify against a shared 160-op / 86-probe fixture; their outputs are byte-identical.

## Real-world consumers

- **[`ZMediumToMarkdown`](https://github.com/ZhgChgLi/ZMediumToMarkdown)** uses `RubyRangeable` to render Medium GraphQL paragraphs into Markdown. Replacing the previous `O(L · m)` per-paragraph walk with `Rangeable.transitions` yielded a **2.23× end-to-end speedup** (55% reduction in render time) while keeping all 306 existing tests byte-identical.
- **AVPlayer non-contiguous byte-range cache** (case study, [RFC §1.3.1](RFC.md)) — a 2021 article on `AVURLAsset` + `AVAssetResourceLoaderDelegate` ([source post](https://medium.com/zrealm-ios-dev/avplayer-%E5%AF%A6%E8%B8%90%E6%9C%AC%E5%9C%B0-cache-%E5%8A%9F%E8%83%BD%E5%A4%A7%E5%85%A8-6ce488898003), [`ZPlayerCacher`](https://github.com/ZhgChgLi/ZPlayerCacher)) explicitly defers non-contiguous `Data` merge as "too complex". `Rangeable<CacheToken>` over a sparse `FileHandle` collapses both the read-side stab ("which prefix of `[lo, hi]` is on disk?") and the write-side merge ("union the new bytes with all prior runs, including integer-adjacent ones") into `transitions(over:)` + `insert(...)`, with no manual merge code.

## Status

| Round | Verdict | Notes |
|---|---|---|
| 1 | REJECTED | 6 MUST-FIX items (Optional<Int> sentinel, Int.min underflow, Φ-potential proof gaps, cleaner-variant pseudocode, missing lemmas, lazy-build concurrency) |
| 2 | **APPROVED** | All MUST-FIX resolved. See [`RFC.md` § 16](RFC.md) for the full review history. |

## License

MIT © [ZhgChgLi](https://github.com/ZhgChgLi)
