# Rangeable RFC

> **Status:** Draft **RFC v2.0** (supersedes v1.0; v1.0 normative content is preserved verbatim except where explicitly amended for v2)
> **Editor:** Algorithms Research (graduate-level draft)
> **Audience:** library implementers in Ruby, Swift, Python, Ruby, Kotlin, Go, JavaScript/TypeScript, prospective ports to other strongly-typed languages, theoretical reviewers
> **Discussion forum:** in-repo issue tracker; this document is the normative reference
> **Conformance language:** RFC 2119 / BCP 14 (MUST / MUST NOT / SHOULD / SHOULD NOT / MAY)
>
> **v2.0 delta (informative summary):** v2 promotes the Removal API (`remove(e, start:, end:)`, `remove(e:)`, `removeAll()/clear()`, `removeRanges(start:, end:)`) and the Set Operations API (`union`, `intersect`, `difference`, `symmetric_difference`) from v1's deferred-future status to **normative**. Eight new method specifications are added in §6.6–§6.13; their normative test contracts are specified in §10.B–§10.G (#21–#80). §4.10 promotes the eager-pruning empty-entry rule to a normative invariant; §5.1 (I1.4) is upgraded from a v1-caveat to an unconditional MUST. §13 and §14.1 / §14.9 are reduced to historical pointers. The v1 API surface (insert, subscript, getRange, transitions, count, isEmpty, copy, each) is **byte-identical** before/after v2 — no v1 caller code or fixture data is broken by the upgrade.

---

## §0  Abstract

`Rangeable<Element>` is a **language-neutral**, generic, integer-coordinate **closed-interval set container**. It pairs an arbitrary Hashable element (`Hashable` in Swift, an object implementing `eql?`/`hash` in Ruby) with the set of intervals on a one-dimensional discrete axis over which the element is active, and provides three query forms:

1. **Element-keyed**: given an element `e`, return all merged, non-overlapping, non-adjacent intervals of `e` on the axis (`getRange`).
2. **Position-keyed**: given an integer position `i`, return all elements active at that position (`subscript[i].objs`), strictly ordered by first-insert order.
3. **Range-keyed**: given `[lo, hi]`, return the ordered sequence of all boundary events within that range (`transitions`).

Semantically, `insert(e, start:end:)` performs an **idempotent, order-preserving, integer-adjacency-merging union** of the existing intervals of `e` with the new interval. `Rangeable` is designed to serve the **build-once-then-query-densely** workload — a canonical example being a markdown / Medium paragraph renderer, which parses all markups as a one-shot batch insert and then densely queries, by character position, "which styles are active at this character?".

This RFC formalises (i) interval semantics (endpoint inclusion, merge on adjacency, idempotence, singleton intervals, out-of-order handling), (ii) element equivalence semantics (based on value equality), (iii) the internal data structure (per-element sorted disjoint non-adjacent list + lazy boundary-event index + version counter), (iv) algorithmic pseudocode and invariants, (v) amortised complexity with a potential-method proof, (vi) comparison against alternatives such as interval tree / segment tree / skip list / Roaring bitmap / Boost.ICL, and (vii) the 20 test contracts that both reference languages (Ruby + Swift) MUST pass byte-identically.

---

## §1  Motivation

### 1.1 The recurring "what's active at index `i`?" problem

Many domains present the same shape of problem:

- **Rich-text rendering**: "Which inline styles (bold, italic, link) are active at this character position? In what outer-first order?"
- **Calendar / resource scheduling**: "What events are occurring at this second? Which resources are being held?"
- **Genome annotation**: "Which genes / exons / regulatory regions cover this base pair?"
- **Game engine status effects**: "Which buffs / debuffs are on the player at this frame?"
- **Tax / contract / regulatory effective dates**: "Which clauses apply on `2026-05-09`?"
- **Streaming-media local cache (HTTP Range)**: "Which byte ranges of this asset are already on disk? For a player request `[a, b]`, which sub-spans can I serve from the local file and which sub-spans must I fetch over the network?" (See §1.3.1.)

The common shape is:

```
Σ = { (eᵢ, lᵢ, hᵢ) | eᵢ is an element, [lᵢ, hᵢ] is its active interval } 
queries: given i, return { e | ∃[l,h] ∈ Σ_e, l ≤ i ≤ h }
         given e, return the merged interval set of e
```

In practice these queries are often **extremely dense** (one query per position) while inserts are **one-shot batch**; the query path must therefore be very fast, and the insert path need only be amortised-acceptable.

### 1.2 Why language-built-in ranges fall short

The "Range" types built into mainstream languages are inadequate for this problem:

- Swift's `Range<Int>` / `ClosedRange<Int>` and Ruby's `Range` describe a **single** interval. They are not interval-set containers; they have no union, no stab query, and no element-dimensional index.
- Swift's `RangeSet` (in the standard library since Swift 5.x), while supporting set operations over multiple disjoint intervals, can still only store `Int` positions themselves; it cannot attach an `Element`, and provides no endpoint index for stab queries.
- Boost.ICL's `interval_map<K, V>` takes another path: it binds each sub-interval to an aggregated value (aggregate-on-overlap). This semantics is natural for "accumulate values on overlap" but does not directly match "multiple inserts of the same element should be idempotent, and one should be able to recover the element's full merged interval set" (see §8).

In other words: what is needed is neither a "multi-interval set" nor a "mapping from intervals to aggregated values", but a **mapping from hashable elements to the merged interval set of that element**, together with an **inverse index from each integer position to the ordered set of active elements**.

### 1.3 The reference consumer: ZMediumToMarkdown markup render

The concrete motivation for this RFC comes from `ZMediumToMarkdown` (a tool that converts Medium article JSON into Markdown). Its key component, `MarkupStyleRender`, follows this pipeline:

1. **Markup parsing**: from Medium JSON, parse each paragraph's `markups` (types: `STRONG`, `EM`, `A`, `CODE`, `ESCAPE`); each markup carries two integer positions `start` and `end` (`end` is inclusive, i.e. it covers the character at that index). The Medium API directly returns multiple overlapping markups of the same type as **independent** entries (e.g. consecutive user-applied bold `[2,5]` and `[3,7]` yield two STRONG markups); the renderer is responsible for the union itself.
2. **ESCAPE injection**: for each raw character, check `Helper.markdownEscapeNeeded?` (which detects markdown meta-characters such as `*`, `_`, `#`, `-`, `` ` ``, and `.` after a leading digit). If escaping is needed, **synthesise an `ESCAPE` markup** `[i, i+1]` and append it to the markups list. **Note**: here the Medium model uses half-open `[i, i+1]`, but the renderer's `TagChar` subtracts 1 from `endIndex` to convert it back to inclusive, and then drives the walker via the comparison `tag.endIndex == index`. This inclusive-endpoint convention is the direct historical reason §4.1 of this RFC pins inclusive semantics.
3. **Position-tag index**: at render time, walk paragraph characters index by index; at each index one must answer: "Which tags should currently be on the stack? Which tags begin at this index (push and emit `**` / `_` / `[`)? Which end at this index (pop and emit `**` / `_` / `)`)?"

The current implementation converts markups into `TagChar`s, sorts them by `startIndex`, and decides open/close at each index by linearly selecting over all tags, giving worst-case `O(L · m)`. L (character count) is typically ≤ 2000, m (markup count) typically ≤ 50, but extreme articles (including synthesised ESCAPE markups) can reach hundreds of paragraphs and tens of thousands of total markups. Within a single paragraph, when the nesting is large, the answer size `r` for `r[i]` can reach about 10.

`Rangeable` is a clean abstraction for this problem:

- `insert(STRONG_token, start: 2, end: 5)` and `insert(STRONG_token, start: 3, end: 7)` automatically union into `[(2, 7)]`; the renderer need not do this by hand.
- `r[i].objs` directly yields the list of tags active at this index, ordered by first-insert order, matching the markdown nesting intuition: earlier-introduced / longer-spanning tags belong on the outside.
- `transitions(over: 0..L-1)` directly supplies the open/close event stream the renderer needs; the main render loop degenerates to a three-step operation:
  ```
  for ev in r.transitions(over: 0..L-1):
    if ev.kind == .open: emit ev.element.start_chars
    elif ev.kind == .close: emit ev.element.end_chars
  ```
  combined with `r[i]` to reopen the residual stack at newline splits.

**Why not just patch ZMediumToMarkdown rather than build a new abstraction?** Because the "position → active-set index" pattern recurs in many downstream problems (see the §1.1 list); extracting it from the markdown renderer as a standalone, spec'd, two-language-portable container is more valuable than an in-place patch.

### 1.3.1 Secondary consumer: AVPlayer non-contiguous byte-range cache (informative)

> This subsection is **informative**. It documents a second, structurally different workload that `Rangeable` cleanly subsumes, and is intended to validate that the v1 API surface generalises beyond the §1.3 markup-render workload it was distilled from.

A common iOS engineering problem is "play-while-cache" for `AVPlayer` / `AVQueuePlayer`: the player issues HTTP/1.1 `Range` requests against a remote audio/video resource (`Range: bytes=lo-hi`), and the app wants to persist the bytes that flow through so a later play of the same asset can be served from disk. The standard implementation route is to give `AVURLAsset` a custom-scheme URL plus an `AVAssetResourceLoaderDelegate`, and intercept the player's range requests there. (See §15 reference [26].)

The hard part is **non-contiguous merging**. The player does not always request the asset linearly:

1. User plays from byte `0`; player gradually requests `[0, …]` and the app caches what arrives, ending at, say, byte `100`.
2. User seeks to mid-asset; player issues `Range: bytes=200-` and the app caches the new bytes, ending at byte `500`.
3. The on-disk cache for this asset now contains `[0, 100] ∪ [200, 500]` — two disjoint runs with a `[101, 199]` gap.
4. User scrubs back and plays `[150, 300]`. The cache layer must answer two questions on the request path, ideally in `O(\log n)`:
   - **(Q1) read-side stab:** for the request range `[150, 300]`, which **prefix** can be served immediately from disk, and where does the first gap begin?
   - **(Q2) write-side merge:** when bytes `[101, 199]` finally arrive over the network, the cache index must merge `[0, 100]`, `[101, 199]`, and `[200, 500]` into a single `[0, 500]` run, *automatically*, including the integer-adjacency case `100 / 101`.

Reference [26] explicitly punts on this: it merges only **strictly contiguous** new data into a single in-memory `Data` blob, drops anything else on the floor, and warns that supporting non-contiguous merge "would require a different storage scheme that can identify gaps, plus a query path that splits the request into 'served-from-disk' and 'fetch-from-network' sub-spans — which is very complex." It also flags that holding the whole asset as a single `Data` is OOM-prone for video-sized assets.

`Rangeable<CacheToken>` (with `CacheToken` a singleton like `.cached`) directly answers both questions and decouples the **byte-range index** from the **byte storage**:

- **Storage of bytes:** a sparse file written via `FileHandle.seek + write` at the right byte offset. This eliminates the "whole asset in `Data`" OOM risk independently.
- **Index of which byte ranges are present:** a single `Rangeable<CacheToken>` instance per asset.
  - **(Q2)** is solved by `insert(.cached, start: offset, end: offset + data.count - 1)`. §4.3's integer-adjacency rule means `[0, 100]` and `[101, 199]` automatically union; `[200, 500]` joins on the next merge. The user never writes the merge logic.
  - **(Q1)** is solved by `r[lo].objs.contains(.cached)` (point-stab for the first byte, `O(\log M + r)` per §6.3) plus a single `transitions(over: lo...hi)` to locate the first close event — the cached run's exclusive end (§4.1.1) — or the first open event when `lo` is in a gap. Splitting `[lo, hi]` into a sequence of "from disk" and "from network" sub-spans is a constant-state walk over the events.

Concretely, the rewrite collapses the original cache manager's protocol from "save bytes + try to merge if contiguous" down to two methods:

```
cachedPrefix(in: lo...hi) -> Data?              // Q1
saveDownloadedData(_, offset:)                  // Q2
```

with no manual merge code on either side; `Rangeable.insert` is the merge, and `Rangeable.transitions` plus the point subscript is the query.

**Why this case validates the v1 API.** The §1.3 markup-render workload exercises (W1) build-once-then-query-densely. This AVPlayer workload, by contrast, exercises an **interleaved insert-and-query** pattern: every player request is preceded by an insert from the previous network response, and the lazy boundary index (§5.2) is rebuilt on demand each time. The fact that the same three primitives — `insert`, `subscript[i]`, `transitions(over:)` — handle both workloads with no API surface additions is the empirical evidence that `Rangeable`'s v1 surface is the right factoring of the underlying problem, not an artefact of the markup-render origin story. The (W2) characteristic also shifts: `m_e` for the cache token can grow as the user scrubs around (dozens of disjoint runs is plausible), but `r` (the active-set size at any byte position) stays at most 1 because there is only one element in the container; the `r[i]` cost is dominated by the `O(\log M)` boundary lookup, well within budget for the request-path latency.

**Out-of-scope for v1:** removing a byte range from the cached set (e.g. an LRU eviction step that drops `[300, 500]`) is currently expressible only by rebuilding the `Rangeable` from the surviving runs returned by `getRange(of: .cached)`. §14.1 already schedules first-class removal for v2; the AVPlayer-cache workload is one of its primary motivators.

### 1.4 Workload characteristics that drove design choices

The design decisions in this RFC are heavily guided by the following workload properties; reviewers may use these to check whether each design choice is well-grounded:

- (W1) **Build-once-then-query-densely**: after a typical build phase completes, the query phase issues one query per integer in `[0, L]`. ⇒ §5.2 adopts a **lazy** rather than incremental index; §5.3 provides a **dense materialised** variant.
- (W2) **`m_e` is typically very small (< 100) but the element count is moderate (< 1000)**: ⇒ a sorted Array per-element list is acceptable (§7.1 worst-case `O(m_e)` shifting); BBST overhead is in fact worse. The RFC nevertheless **permits** implementations to choose a BBST, since it is more stable when `m_e` is large.
- (W3) **The "value" of an element's equivalence class is typically immutable in spirit**: users do not expect to mutate an element after insertion ("I've already dropped this STRONG marker in, why would I mutate it?"). ⇒ §4.6 adopts a lightweight freeze-based defensive strategy rather than a heavy deep-clone.
- (W4) **Cross-language reproducibility is a first-class requirement**: tests must be byte-identical across Ruby + Swift (and potential future Kotlin / Rust ports). ⇒ §4.5 adopts deterministic insertion-order ordering and rejects any hash-bucket or RNG-dependent scheme.

---

## §2  Terminology

> This section is **normative**. If any of the terms below appears later in the document, the definitions here govern.

- **Coordinate**: an integer, positive or negative (see §4.7). Coordinates form a discrete one-dimensional axis ℤ.
- **Interval**: a pair of integers `(lo, hi)` denoting the **closed interval** `[lo, hi]` ⊆ ℤ, with the requirement `lo ≤ hi`. When `lo = hi` it is a **singleton interval**.
- **Element**: an opaque value embedded in a `Rangeable` container. Elements are compared for equivalence by **value equality** (§4.2).
- **Active set at i**: `{ e | ∃[l,h] ∈ R(e), l ≤ i ≤ h }`, where `R(e)` is the merged interval set of element `e`.
- **Boundary event**: for a merged interval `[l, h]` of an element `e`, `(l, +e)` is an **open event** and `(h+1, −e)` is a **close event**. The **boundary-event sequence** of the entire `Rangeable` is obtained by sorting all elements' open/close events lexicographically by `(coordinate, kind)` and then applying the tie-breaking rules of §4.5.
- **Transition**: a coordinate at which at least one open or close event occurs.
- **Equivalence class**: the equivalence class of elements under the relation `==` / `eql?` per §4.2.
- **Insertion order**: the wall-monotonic ordinal (1, 2, 3, …) assigned at the first insert of each **element equivalence class**. Later inserts of an equivalent element do not update this ordinal.
- **Version counter**: a monotonically increasing integer maintained by the container; incremented after any mutation (see §11).

> Below, `R(e)` always denotes the **canonical merged interval set** of element `e`: pairwise non-overlapping, pairwise non-adjacent, sorted ascending by `lo`.

---

## §3  API Surface

API names are in English (aligned with the implementation layer). This section defines the surface and contract of user-visible types and methods; §4 defines their semantics; §6 gives the algorithms.

### 3.1 Constructor / `empty()`

```
Rangeable<Element>()
Rangeable.empty() -> Rangeable<Element>
```

The constructor creates an **empty** container: every `R(e)` is ∅, version = 0, and the boundary-event index is ∅.

**Postcondition:** for any `i`, `r[i].objs == []`; for any `e`, `getRange(of: e) == []`.

### 3.2 `insert(element, start:, end:)`

```
insert(e: Element, start: Int, end: Int) -> Void
```

**Contract:**

- *Precondition:* `start ≤ end`. Otherwise MUST raise/throw a documented `InvalidIntervalError` (a subclass of `ArgumentError` in Ruby; the case `.invalidInterval(start, end)` of an `Error` in Swift).
- *Effect:* `R(e)` ← canonicalize(`R(e)` ∪ {[start, end]}). Canonicalize is defined as: union all overlapping or integer-adjacent intervals and re-sort. Idempotent: two calls of `insert(e, s, t)` yield the same final `R(e)` as one.
- *Insertion order:* if the equivalence class of `e` has not previously appeared in the container (`R(e) == ∅` and has never been touched by any prior insert), assign it the next insertion-order ordinal. Otherwise the insertion order is unchanged.
- *Version:* if this insert changes `R(e)`, the version counter increments. An idempotent insert (same `e`, same `[start, end]`, `R(e)` unchanged) MUST NOT bump the version.
- *Mutation/aliasing:* the implementation MUST ensure that subsequent mutation of `e` by the caller after insert does not break the container's internal invariants (see §4.6).

### 3.3 `subscript [i]` / `activeAt(index:)`

```
r[i] -> Slot
r.activeAt(index: i) -> Slot
where Slot.objs: ArrayLike<Element>
```

Both forms are equivalent; the former is idiomatic sugar. Returns a `Slot` wrapper (rather than directly returning an array, so future fields such as a `transitions` subset or `count` can be added). `Slot.objs` is an **ordered, repeatedly iterable** element sequence whose order is defined by §4.5.

**Contract:**

- Legal for any `i ∈ ℤ` (including negative and positions beyond any inserted interval) at `O(log M + r)`, where `r = |Slot.objs|` and `M = Σ_e |R(e)|`.
- `Slot.objs` MUST be deterministic while the container is **unmutated** (the same `i` always returns the same sequence).

### 3.4 `getRange(of: element)` / `Element.getRange(from: r)`

```
getRange(of e: Element) -> [(Int, Int)]
// In Ruby/Swift this can also be exposed as element-side syntactic sugar:
e.getRange(from: r) -> [(Int, Int)]
```

Returns the canonical merged interval set `R(e)` of element `e` as an **ordered tuple list** (ascending by `lo`, each `(lo, hi)` a closed interval).

**Contract:**

- If `e` has never been inserted (and no element equal to it has been inserted), returns an empty list.
- The returned intervals are pairwise non-overlapping and non-adjacent (until a subsequent `insert` mutates the state).
- The returned value MAY be a defensive copy or a read-only view; callers MUST NOT rely on mutability.
- Complexity: `O(|R(e)|)` for a copy, or `O(1)` for a view.

**Sugar form alignment (normative):**

`e.getRange(from: r)` is **optional sugar** (not required for v1); its scoping in each language is:

- **Swift:** sugar is implemented via `extension Hashable { func getRange<E: Hashable>(from r: Rangeable<E>) -> [(Int, Int)] where Self == E }` (see §12.2). The original `extension Rangeable.Element where Element: Hashable` is not valid Swift (`Element` is a generic parameter).
- **Ruby:** sugar **MUST** be provided via a [Ruby refinement](https://docs.ruby-lang.org/en/3.3/Refinements.html) (`refine Object do ... end`); callers must opt in explicitly with `using Rangeable::Sugar`. The RFC **MUST NOT** expose `Object#getRange` via global open-class monkey patching, to avoid namespace pollution. Implementations MAY additionally provide a module-level free function `Rangeable.range_of(e, from: r)` as an alternative sugar.

### 3.5 `transitions(over:)`

```
transitions(over range: ClosedRange<Int>) -> [TransitionEvent]

struct TransitionEvent {
  coordinate: Int
  kind: .open | .close
  element: Element
}
```

Returns the **ordered sequence** of all boundary events within the query range `[lo, hi]`. Ordering rules (§4.5, normative):

1. Primary key: `coordinate` ascending.
2. Secondary key (same coordinate): all `.open` events **precede** all `.close` events. Rationale: at the same coordinate `i`, if an element opens and closes at the same time, it is active at `i`; therefore its open event must precede its own close event in the sequence.
3. Within the same coordinate and same kind:
   - For `.open`: ascending **insertion order** of the element (the earlier first-insert ranks first).
   - For `.close`: descending **insertion order** (LIFO, matching stack discipline; see §10 case 16).

**Range inclusion:** a boundary event `(c, kind, e)` is part of this query's result iff `lo ≤ c ≤ hi + 1`. (Note: a close event's coordinate is `h+1`, so to cover an interval ending at `hi`, the query range needs to include `hi`; the produced close event has coordinate `hi+1`, but this query still includes it because it corresponds to the closing of the "last active position" at ≤ `hi`.)

**Boundary handling:** if an interval `[l, h]` has `l < lo` but `h ≥ lo`, is there still an open event? — **No**, because the open event's coordinate is outside `[lo, hi]`. If the caller needs the information "this interval is already active at `lo`", use `r[lo].objs` to obtain the active set at `lo` as a seed for `transitions`.

### 3.5.1 Collection-style basic APIs (v1 normative)

> To meet the minimum baseline expectation of a "container", v1 specifies the following methods.

```
r.count       -> Int        // number of distinct equivalence-class elements ever inserted, i.e. |insertion_order|
r.isEmpty     -> Bool       // r.count == 0
r.each { |element, ranges| ... }    (Ruby)
for (element, ranges) in r          (Swift via Sequence conformance)
r.copy()      -> Rangeable<Element>  // explicit deep copy (see below)
```

**Contract:**

- `count`: MUST be the number of inserted element equivalence classes; not `M = Σ |R(e)|`. In the scenario where `R(e)` ends up empty after only idempotent no-op merges, since v1 has no remove, the equivalence class remains in `insertion_order`, so `count` does not regress.
- `isEmpty`: equivalent to `count == 0`; furthermore, if `isEmpty` is true, `transitions(over:)` MUST return `[]` for any range.
- `each` / iteration: the iteration order **MUST** be `insertion_order` ascending (consistent with §4.5); each yielded `ranges` is the (I1)-canonical `R(e)` as a read-only view or defensive copy. Mutation during iteration is undefined behavior (same convention as Ruby `Hash#each` / Swift `Dictionary` iteration).
- `copy()`: MUST be a **deep copy of internal state** (intervals map, insertion_order, ord, version, event_index). The replica and the original are mutually independent (v1 mutations do not affect each other). Swift's value-type semantics provide deep copy naturally (COW triggered by mutation); Ruby implementations must manually deep-clone the Hash + Array structure. Complexity `O(M + E)`.
- v1's `copy` **MUST NOT** share state by reference (callers expect mutation isolation). Persistent / share-then-COW variants are deferred to §14.2.

**Complexity:** `count`/`isEmpty` `O(1)`; iteration `O(E + Σ|R(e)|)`; `copy` `O(M + E)`.

### 3.6 Now-normative APIs (formerly v1 out-of-scope) and remaining out-of-scope APIs

This subsection enumerates the API-surface deltas relative to v1. v2 promotes two clusters of operations from v1's deferred status to normative, and retains the remainder as out-of-scope.

**v2-normative (formerly out-of-scope in v1):**

- `remove(e, start:, end:)` — now normative; see §6.6 (semantics) and §10.B (test contract).
- `remove(e:)` (a.k.a. `remove_element` / `delete`) — now normative; see §6.7 and §10.B.
- `removeAll()` / `clear()` — now normative; see §6.8 and §10.B.
- `removeRanges(start:, end:)` — now normative; see §6.9 and §10.B.
- `union(other)` — now normative; see §6.10 and §10.C.
- `intersect(other)` (a.k.a. `intersection`) — now normative; see §6.11 and §10.D.
- `difference(other)` (a.k.a. `subtracting` / `subtract`) — now normative; see §6.12 and §10.E.
- `symmetric_difference(other)` — newly added in v2 (was not even sketched in v1's §14.9); see §6.13, §10.F, and the §14.9 amendment for inclusion justification.

Each of the four set operations exists in BOTH a non-mutating "returns a new `Rangeable<Element>`" form AND a mutating "modifies self" form, in languages where both are idiomatic. Per-language naming is the implementer's responsibility (Swift `formUnion`/`union`, Ruby `union!`/`union`, Python `update`/`union`, Kotlin `unionInPlace`/`union`, Go `Union`/`Unioned`, JS `unionInPlace`/`union`); the RFC pins **semantics**, not surface spelling.

**Still out-of-scope in v2 (deferred to v3+):**

- A persistent / immutable variant of `Rangeable<Element>` (snapshot semantics) — see §14.2.
- Floating-point or non-integer coordinates — see §14.3.
- Multi-dimensional (rectangle stabbing) — see §14.4.
- Range query returning a sequence of `Slot`s (`r[lo...hi].objs`).
- Aggregate-on-overlap variant — see §14.5.

### 3.7 API ergonomics: aligning the surface to language conventions

Although this RFC is **language-neutral**, the table below recommends naming alignment for the two reference languages:

| Concept | Ruby | Swift |
|---|---|---|
| Container | `Rangeable.new` | `Rangeable<Element>()` |
| Insert | `r.insert(e, start: 2, end: 5)` | `try r.insert(e, start: 2, end: 5)` |
| Subscript | `r[i]` | `r[i]` |
| Range query (canonical) | `r.range_of(e)` or `r.getRange(of: e)` | `r.getRange(of: e)` |
| Range query (sugar) | After `using Rangeable::Sugar`, `e.getRange(from: r)` (refinement-only, not a global monkey-patch) | `extension Hashable { func getRange<E>(from:) ... where Self == E }` or free function `getRange(of:from:)` |
| Transitions | `r.transitions(over: 0..10)` | `r.transitions(over: 0...10)` |
| Error | `Rangeable::InvalidIntervalError < ArgumentError` | `Rangeable.InvalidIntervalError: Error` |
| Empty | `Rangeable.empty` | `Rangeable.empty()` |

These names are **SHOULD**; implementations may make reasonable adjustments to fit each language's idioms (e.g. Ruby keyword args, Swift trailing closure conventions). However, **MUST**: the data structures and orderings returned by the API must be exactly consistent (§4.5).

### 3.8 Public surface stability commitments (informative)

v1 treats the following as **stable** API contract (not changeable without an RFC revision):

- The signature and behavior of `insert`
- The `Slot.objs` ordering semantics of `[i]` / `activeAt(index:)`
- The return shape and ordering of `getRange`
- The return shape and ordering of `transitions`

v1 treats the following as **internal** (implementation may vary):

- The concrete numeric value of `version` (still guaranteed monotonic per §11)
- The memory layout of `event_index`
- Whether the heuristic for the §5.3 (e) variant is enabled

---

## §4  Semantics (normative)

> **This section uses RFC 2119 keywords.** For the meanings of MUST / MUST NOT / SHOULD / SHOULD NOT / MAY see *RFC 2119: Key words for use in RFCs to Indicate Requirement Levels* (S. Bradner, March 1997).

### 4.1 End is INCLUSIVE

`insert(e, start: a, end: b)` denotes the **closed interval** `[a, b]`. `b` itself MUST be considered part of `e`'s active range.

> **Formal:** for the `Rangeable r` after `insert(e, s, t)`,
> ∀ i ∈ ℤ, `e ∈ r[i].objs` ⇔ ∃ `[l,h] ∈ R(e)` with `l ≤ i ≤ h`.

**Rationale:**

1. Matches the endpoint semantics of markups in ZMediumToMarkdown (Medium's `start` / `end` are inclusive). The loop condition `tag.endIndex == index` in `MarkupStyleRender#walkCharsWithTags` is exactly an inclusive comparison; the `- 1` performed on `endIndex` in `TagChar.initialize` is direct evidence that Medium's half-open `end` is being restored to inclusive form.
2. Best matches the human intuition of "what is active on this character" (no need to mentally compute `end - 1`).
3. With inclusive endpoints, the **integer-adjacency merge rule** (§4.3) takes its cleanest form during union/merge: `hi₁ + 1 == lo₂` suffices to merge.
4. In the single-point case (`start == end`), the closed-interval notation `[k, k]` is valid and non-empty (length 1); the half-open notation `[k, k)` would be empty, forcing single points to be written `[k, k+1)`, which is unintuitive.

> **Contrast with half-open `[a, b)`:** `[a, b)` is popular in the C/C++ standard library (Lobsters: "Always use [closed, open) intervals") because length = `b - a` and the empty range has a clean representation. The corresponding adjacency-merge rule becomes `hi₁ == lo₂`, marginally simpler at merge time, but it transfers the subtlety of "end is the position *after* the interval" to every query — counter-intuitive in the markup domain when asking "which byte has STRONG?". This RFC pins inclusive semantics and **does not accept** a half-open variant; a floating-point variant (which would naturally be half-open) is future work (§13, §14).

#### 4.1.1 `transitions` close-event coordinate convention

Note: although the external API uses inclusive `[a, b]`, internally `event_index` sets the coordinate of a close event to `h + 1` (the first no-longer-active position). This is the standard sweep-line idiom because:

- It makes the semantics of "active set at i" precisely "opens minus closes accumulated up to the most recent close event before i", with no ambiguity for bsearch at segment boundaries.
- It makes integer-adjacency merging (§4.3) correspond to the sweep events `(coord = h+1, close, e)` and `(coord = h+1, open, e)` cancelling at the same coordinate (since `open` always precedes `close` per §4.5), equivalent to extending the active span of the same element without leaving a spurious gap segment.

`transitions(over: lo..hi)` uses the bound `lo ≤ ev.coord ≤ hi + 1` so that the close event of an interval ending at `hi` (whose coord = hi+1) is also returned, helping the caller handle the right-edge case.

### 4.2 Element equality contract

`Rangeable` uses **language-native value equality** to decide whether two elements belong to the same equivalence class:

- **Ruby:** `eql?` paired with `hash`. `Strong.new == Strong.new` is expected to return true (via the implementer overriding `==`, `eql?`, `hash`).
- **Swift:** `Hashable` (which implies `Equatable`). `Strong() == Strong()` is expected to return true (via `==` and `hash(into:)` being consistent).

**Formal contract:**

- (E1) **Reflexive:** `e == e` is always true.
- (E2) **Symmetric:** `e₁ == e₂` ⇒ `e₂ == e₁`.
- (E3) **Transitive:** `e₁ == e₂ ∧ e₂ == e₃` ⇒ `e₁ == e₃`.
- (E4) **Hash consistent:** `e₁ == e₂` ⇒ `hash(e₁) == hash(e₂)`.
- (E5) **Stability (§4.6):** an element should not be externally mutated after insertion; this is a joint obligation of caller and container.

> **Consequence:** `Link("a")` and `Link("b")` (different URLs) belong to different equivalence classes, **do not merge**, and each maintain an independent `R(·)`. Two instances of `Strong.new` (no ivars, `==` true, `hash` equal) belong to the same equivalence class and **do merge**.
>
> *See Shopify Engineering, "Implementing Equality in Ruby" and Apple's official Swift documentation for `Hashable`.*

### 4.3 Adjacency rule for integer coordinates

On integer coordinates, **merge on adjacency**:

> If two intervals `[a, b]` and `[c, d]` of `e` satisfy `b + 1 == c` (or `d + 1 == a`), they MUST be merged into `[min(a,c), max(b,d)]`.

**Rationale:** on a discrete integer axis there is no intermediate position between `b` and `b+1` that could carry the fact "e is not here". Treating `[2,4]` and `[5,7]` as two independent intervals would introduce an artificial, unnecessary split in `R(e)`; querying `r[i].objs` would see `e` at i=4 and i=5 but be unable to say "no e" at the (non-existent) i=4.5 — nonsensical in the discrete domain.

**Scope:** this rule MUST apply only to integer (or, more generally, **discrete totally ordered**) coordinates. For floating-point or dense rationals there is no notion of a "next coordinate", so the rule does not hold and is not specified by this RFC (see §13, §14).

### 4.4 Empty / degenerate / out-of-order intervals

- (D1) **`start > end`:** MUST raise `InvalidIntervalError`. The container state MUST remain unchanged (exception-safe).
- (D2) **`start == end`:** a valid singleton interval. After `insert(e, 5, 5)`, `r[5].objs` MUST contain `e`; `r[4].objs` and `r[6].objs` MUST NOT contain `e` (unless another interval covers them).
- (D3) **Empty container:** for any `i`, `r[i].objs == []`; for any `e`, `getRange(of: e) == []`; `transitions(over:)` returns `[]`.

> **Silent semantics:** there is no fallback that reinterprets `start > end` as a reverse range, because that would silently rewrite the user's intent; an explicit raise is more predictable than silent normalisation.

### 4.5 First-insert ordering

> This subsection is the **normative definition** of active-set and transitions ordering.

Whenever an element `e` not previously seen (by §4.2 equivalence) is `insert`ed for the first time, the container MUST assign it an integer larger than every existing insertion-order ordinal as its **insertion-order index** `ord(e)`. Subsequent inserts of `e` do not update `ord(e)`, even if a later insert absorbs the intervals of another earlier-inserted element.

**Active-set ordering (§3.3 `[i].objs`):**

> The enumeration order of the active set at position `i` MUST be `{ e ∈ active(i) }` ascending by `ord(e)`.

**Transitions tie-breaking at the same coordinate (§3.5):**

> Within a single coordinate, all open events come before all close events (`open < close`);
> within events of the same kind: opens sort by `ord(e)` ascending; closes sort by `ord(e)` **descending** (LIFO).

**Rationale:**

1. **Determinism:** insertion order is a monotonic sequence explicitly produced by program logic; it does not depend on hash buckets or randomness.
2. **Cross-language reproducibility:** Ruby and Swift dictionary/hash iteration is not guaranteed consistent across languages, but the insertion-order index is a pure integer, ensuring byte-identical output across both languages.
3. **Markdown nesting intuition:** an outer style introduced earlier is usually the logical outer container. LIFO close events match the legal markdown nesting `**` open / `_` open / `_` close / `**` close.
4. **Aligned with the reference consumer:** this ordering is the conceptual analogue of `MarkupStyleRender`'s `sort_by(&:sort)` for "multiple tags with the same `startIndex`"; but Rangeable abstracts the sort away from *style category* and uses first-insert order, which is cleaner.

> **Consequence (§10 case 13):** after `insert(S, 1, 5); insert(I, 3, 7); insert(S, 4, 8)`, `r[6].objs` is `[S, I]`. S is indeed active at position 6 (because `[4,8]` covers 6), and `ord(S) = 1` < `ord(I) = 2`, so S comes first. Note that S's interval before position 6 was `[1,5]`, which does not cover 6; but after merge `R(S) = [(1,8)]` (since `[1,5] ∪ [4,8] = [1,8]`), so S is active at 6.

#### 4.5.X Lemma — the same element cannot produce open and close at the same coord

> **Lemma 4.5.X.** For any element `e`, the event sequence produced by `build_event_index` contains **no** pair `(c, open, e)` and `(c, close, e)` at the same coord `c`.

**Proof.** `event_index` is generated directly from the (I1)-canonical `R(e) = [(l_1, h_1), …, (l_n, h_n)]`; for each entry `(l_j, h_j)` it produces the events `(Some(l_j), open, e)` and `(succ(Some(h_j)), close, e)`.

By contradiction, assume some `c` is simultaneously an open coord and a close coord of `e`, i.e. `Some(l_a) == succ(Some(h_b))` for some `a, b`.
- If `h_b < Int.max`: `succ(Some(h_b)) = Some(h_b + 1)`, so the equation `l_a == h_b + 1` means entries `b` and `a` are **integer-adjacent**. But (I1.2) requires adjacent entries of `R(e)` to satisfy `h_b + 1 < l_a` (gap ≥ 2 in lo-difference, equivalent to gap ≥ 1 on the integer axis); contradiction.
- If `h_b == Int.max`: `succ(Some(h_b)) = None`; while `l_a` is a finite `Int`, so `Some(l_a) ≠ None`; no equation holds.

Therefore no same-coord open + close exists for `e`. ∎

> **Consequence:** the reassuring claim in §4.1.1 — "same-coord opens always precede closes ⇒ extend the active span of the same element without leaving a spurious gap segment" — fundamentally relies on the §4.3 adjacency-merge rule having already, at the (I1) stage, ruled out "same element opens and closes at the same coord". Lemma 4.5.X formalises this argument.

#### 4.5.Y Lemma — correctness of relative ordering across elements at the same boundary coord

> **Lemma 4.5.Y.** For any distinct elements `e ≠ f` and any coord `c`, if `e`'s close-event coord = `c` and `f`'s open-event coord = `c`, then the §4.5 tie-breaking rule (open events precede close events within the same coord) guarantees that the active set after the sweep processes that coord is exactly `{ x | ∃ entry (l, h) ∈ R(x), l ≤ c ≤ h }`, i.e. the true active set at `c`.

**Proof sketch.** Suppose `e` closes at `c` and `f` opens at `c`. Consider two adjacent segments:
- Segment A: `(prev_coord, c - 1)` whose active set contains `e` (since `e` opened before `c` and has not yet closed) but not `f` (since `f`'s open is at `c`).
- Segment B: `(c, next_coord - 1)` whose active set does not contain `e` (since `e` closes at `c`) but contains `f` (since `f` opens at `c`).

If §4.5 processed `c`'s events with close before open, the sequence would be: first remove `e` (active transiently becomes "no e, no f"), then add `f` (active becomes "no e, with f"). Taking a snapshot **before** processing `c`'s events yields segment A's correct value; taking a snapshot afterwards yields segment B's correct value; both orderings are fine, because the sweep takes snapshots between coords, not in intermediate states within a single coord. But §4.5 specifies "open before close", i.e. first add `f`, then remove `e`, yielding "add f → active contains e + f → remove e → active contains f"; the final active set for segment B is "contains f", consistent with the requirement.

The key observation is: **the *final* active set at `c` is independent of the add/remove order** (set operations); the §4.5 open-before-close rule only ensures correct degenerate behavior in the **same-element same-coord open + close** scenario (already ruled out by Lemma 4.5.X) — namely "e is still active at c" semantics (i.e. the close of `[c, c]` is placed after all opens). For the cross-element case `e ≠ f`, the open-before-close rule affects only the user-visible ordering of the transitions sequence (§3.5), not the correctness of the active set after the sweep.

Therefore segment B's active set is the true active set at `c`. ∎

> **Consequence (reinforcing §6.5.C's proof sketch):** Lemmas 4.5.X + 4.5.Y together guarantee that §6.5.C ("Active set correctness") has no unspecified behavior at same-coord boundaries; §6.5.C may safely cite these two lemmas.

### 4.6 Mutation / aliasing (frozen-on-insert)

The mutation obligations between caller and container are:

- (M1) **The caller MUST NOT** mutate hash-affecting state of an element after inserting it. Violating this contract can corrupt the container's internal invariants (especially §4.2 hash consistency).
- (M2) **The container SHOULD** take the cheapest "freeze on insert" defense on the insert path:
  - **Ruby:** `e.dup.freeze` (note: if the element is a nested structure containing mutable containers, a deep freeze is too expensive; this RFC accepts shallow freeze as sufficient).
  - **Swift:** value semantics naturally satisfy this invariant; no additional action is required. Reference-type elements in Swift are not in this RFC's primary scope; if used, the caller alone is responsible for (M1).
- (M3) Ruby implementations MAY provide an `unsafe_insert` path that skips the freeze for performance reasons, but such a path is not within the RFC's specification.

### 4.7 Domain (integer coordinates, signed)

- (C1) Coordinates MUST be machine integers (Swift `Int`, Ruby `Integer`).
- (C2) Negative numbers MUST be supported (§10 case 18).
- (C3) The container MUST NOT implicitly modulo or clamp coordinates.
- (C4) **`Int.max` close-event coordinate (normative, unique):** the coordinate of a close event is conceptually `hi + 1`; when `hi == Int.max`, `hi + 1` overflows on machine integers. Implementations MUST use `Optional<Int>` to represent it:

  > **Internal coord type := `Optional<Int>`**, where `None` represents +∞ (semantically "greater than every finite Int").
  >
  > **Sort total order:** for any finite `Some(a)`, `Some(b)`, `Some(a) < Some(b) ⇔ a < b`; and for any finite `Some(a)`, `Some(a) < None`; `None`s are equal.

  An open event's coord is `Some(lo)`; a close event's coord is `coord.succ()`, where `succ` is defined as:

  > `succ(Some(hi)) := Some(hi + 1)` when `hi < Int.max`; `succ(Some(Int.max)) := None`; `succ(None)` is undefined (will not occur on a legal path).

  This is **MUST** (not SHOULD) because cross-language byte-identical output (§4.4 D3, §10 Test #20) requires both implementations to produce **identical** transitions sequences at the `coord == Int.max` boundary; the saturating and sentinel schools produce completely different sequences at this boundary (saturating collides a real `Int.max` open with an `Int.max` close; the sentinel scheme places the close at the end), and cannot be exchanged equivalently. This RFC adopts the sentinel scheme via `Optional<Int>` to avoid any collision with a finite coord.

  > **Consequence (normative invariant):** for any legal insert, internal event coords satisfy `Some(lo) ≤ coord(close) ≤ None`; and `coord(close) == None ⇔ hi == Int.max`. The `transitions(over: lo..hi)` query range `lo ≤ ev.coord ≤ hi+1` degenerates, when `hi == Int.max`, to `Some(lo) ≤ ev.coord ≤ None`, including all `None` close events.

- (C5) **`Int.min` endpoint underflow specification:** an open event's coord is `Some(lo)`; when `lo == Int.min`, this is `Some(Int.min)` with **no underflow** (no `lo - 1` operation is needed). `Int.min` MUST be supported as a valid `lo`. Any internal algorithm MUST NOT compute `lo - 1` (which underflows undefinedly when `lo == Int.min`); use the equivalent form `coord.succ() ≥ Some(lo)` (see §6.1) to avoid the underflow. `hi == Int.min` is also legal (singleton interval `[Int.min, Int.min]`).

  > **Consequence:** `[Int.min, Int.max]` is a legal interval (covering the entire machine-integer axis). Its open event has coord `Some(Int.min)` and close event has coord `None`. Every query MUST correctly report this element as active for any `i ∈ [Int.min, Int.max]`.

- (C6) **Host-language integer-precision caveat (informative, v2.0 patch).** Several implementation languages cannot represent the full signed-i64 axis natively:

  > **JavaScript / TypeScript:** `Number` is IEEE-754 double; integer-precise only on `[-(2^53 - 1), 2^53 - 1]` (i.e. `Number.MAX_SAFE_INTEGER == 9007199254740991`). Coordinates beyond this range MAY be silently quantised by `JSON.parse`, arithmetic, or any path that round-trips through `Number`. Implementations MAY choose to (a) accept silent quantisation and document the limit, or (b) require `BigInt` coordinates with an explicit constructor option; this RFC does NOT mandate either choice.
  >
  > **Cross-language fixtures (RFC §10.G stress tests, `cross_language.json`):** to ensure byte-identical conformance across all 6 reference languages, fixture authors SHOULD constrain test coordinates to `[-(2^53), 2^53]`. Coordinates outside this range may pass on languages with native i64 (Swift, Ruby, Kotlin, Go, Python) but silently fail equivalence on JS/TS. The `Int.min` and `Int.max` boundary tests (§10 Tests #23, #23.A, #30, #31) remain normative and use language-specific sentinels (`Number.MIN_SAFE_INTEGER` / `Number.MAX_SAFE_INTEGER` in JS, `Int.min` / `Int.max` in others), not the literal i64 bounds.
  >
  > **Other host languages:** Ruby `Integer` and Python `int` are arbitrary-precision (no upper bound); Swift `Int`, Kotlin `Int`, and Go `int` are platform-dependent (typically 64-bit signed). All five represent the cross-language fixture i64 range losslessly.

### 4.10 Empty-entry semantics (eager pruning, normative)

> **This subsection is normative and is new in v2.** It governs the post-state of every operation that can shrink `R(e)` — i.e. the four removal methods (§6.6–§6.9), the three set operations that can shrink (§6.11 intersect, §6.12 difference, §6.13 symmetric_difference), and any future op with the same effect.

**(N1) Eager pruning.** When the last interval of `R(e)` is removed by any single operation, the element `e` MUST be excised from **all three** of the container's element-keyed structures atomically with the operation that emptied it:

1. removed from the `intervals` map (i.e. `e ∉ keys(intervals)` after the op);
2. removed from `insertion_order` (i.e. `e ∉ insertion_order` after the op); and
3. removed from `ord` (i.e. `ord[e]` is undefined after the op).

After the excision, `ord` MUST be densely renumbered `1..N` over the surviving elements in their existing `insertion_order` position order, so that `ord(e) ∈ [1, count]` continues to hold (§5.1 I1, §4.5).

**(N2) Container-level consistency.** As a direct consequence of (N1), the post-state of any v2 op satisfies the dual of (I2): `keys(intervals) == set(insertion_order) == keys(ord)`, with `|insertion_order| == count == |keys(intervals)|`. There MUST NOT exist an "empty-but-present" element (i.e. `e ∈ insertion_order ∧ R(e) == ∅` is forbidden post-op).

**(N3) Idempotence on a never-present element.** Removing an element that was never inserted, or removing an interval that does not intersect any `R(e)`, MUST be a no-op: no mutation, no version bump, no `event_index` invalidation (§3.2 idempotence dual; cf. §6.6 step 4, §6.7 fast-path, §6.9 atomic-bump rule).

**Rationale (six points):**

1. **§3.2 idempotence symmetry.** Insert is idempotent on a contained interval. The natural inverse — removing a never-inserted element, or removing an empty interval-set — is a no-op. Eager pruning preserves this duality without an asymmetric "ghost element with empty `R(e)`" state. With Option B ("keep empty entry") the inverse is asymmetric: removing the last interval of `e` keeps `e` in `count`, but removing a never-inserted `e'` does not add it; this defeats the natural "insert/remove are inverses" reading.
2. **§5.1 (I1.4) was already written this way (modulo a v1-deferral caveat).** The original (I1.4) text said "if all intervals of an element are deleted, its key should be removed". v2 simply removes the v1-only parenthetical "but v1 does not provide remove, so in practice…" — see §5.1 below.
3. **`removeAll(); isEmpty == true` becomes provable.** Without (N1), `removeAll()` would still be required to clear `insertion_order` (since otherwise `count > 0`), creating an asymmetric special case. (N1) collapses this asymmetry by making **every** removal-shrinking op eagerly prune.
4. **Cross-language Hash semantics.** Ruby `Hash`/Swift `Dictionary`/Python `dict`/Kotlin `LinkedHashMap`/Go `map`/JS `Map` all naturally remove keys on delete. An "empty-but-present" key requires extra bookkeeping in every language and re-introduces the `intervals.size != insertion_order.size` ambiguity that (I2) deliberately rules out.
5. **Insert-after-remove is well-defined.** If `e` is removed entirely and then re-inserted, the second insert is logically a first-insert (per §4.5: first-insert ordinal is assigned at the first effective insert, and after eager pruning there is no prior effective insert to remember). The element receives a fresh `ord(e)` placed at the tail of `insertion_order`. See deliberate-side-effect note below.
6. **Set operations get this for free.** For `intersect` (§6.11) and `difference` (§6.12), eager pruning makes `e ∈ keys(result) ⇔ R_result(e) ≠ ∅` automatic; no separate post-op "purge phase" is needed in any implementation.

**Deliberate side effect (documented):** an interleaved sequence `r.insert(e, …); r.remove(e); r.insert(e, …)` yields a fresh `ord(e)` on the second insert; `e` is appended at the tail of `insertion_order`, not restored to its previous position. This is by design: §4.5 anchors `ord` at the **first effective insert**, and after eager pruning the prior appearance is gone (no record survives), so the second `insert(e)` is logically a first-insert. Callers that wish to preserve `ord(e)` across a transient remove MUST avoid the full-remove path (e.g. by retaining one residual interval). See test #14 (Risk R14) and §10.B for cross-language conformance probes.

**Atomicity vs. (I3) lazy index.** Eager pruning is itself a mutation; per §5.2 (I3.c) it MUST invalidate `event_index`. The `version` bump and `event_index := nil` step happen exactly once per atomic op, even when multiple elements are pruned in the same call (e.g. `removeRanges` in §6.9, or `intersect` in §6.11). This avoids `O(E)` redundant index invalidations within a single op.

---

## §5  Reference Data Structure

> This section defines the **reference data structure**. Implementations MAY substitute equivalent structures so long as externally observable behavior (API behavior + complexity bounds) remains consistent (see §5.3).

### 5.1 Per-element sorted disjoint list — invariant

```
intervals : Map<Element, SortedList<Interval>>
where Interval = (lo: Int, hi: Int)
```

**Invariant (I1) — Per-element canonical form:**

For any `e ∈ keys(intervals)`:
1. The elements of `intervals[e]` are arranged in strictly ascending order by `lo`;
2. Any two adjacent entries `(lo₁, hi₁), (lo₂, hi₂)` satisfy `hi₁ + 1 < lo₂` (neither overlapping nor integer-adjacent);
3. Each entry satisfies `lo ≤ hi`;
4. **(v2 normative; was a v1-deferred caveat).** `intervals[e]` MUST be non-empty. Empty entries MUST NOT exist in `intervals`: when the last interval of `R(e)` is removed by any operation, the key `e` MUST be excised from `intervals` (and from `insertion_order` and `ord`) before the operation returns. See §4.10 for the full eager-pruning normative statement and rationale.

**Map key equality** is defined per §4.2; Ruby uses `Hash`, Swift uses `Dictionary`, Python uses `dict`, Kotlin uses `LinkedHashMap`, Go uses `map`, JavaScript/TypeScript use `Map`.

**Insertion-order tracking:**

```
insertion_order : List<Element>     // ordered by first effective insert
// equivalently: Map<Element, Int> recording ord(e)
```

**Invariant (I2) — Element-keyed structure consistency (v2 normative; tightened from v1).**

- (I2.a) `set(insertion_order) == keys(intervals) == keys(ord)`. v1 allowed `insertion_order ⊇ keys(intervals)` because v1 never removed; v2's eager-pruning rule (§4.10) makes equality unconditional.
- (I2.b) `insertion_order` is **append-only between contiguous runs of insertions**: a run of insertions that does not pass through a remove or set operation MUST NOT reorder existing entries; only first-insert appends a new element to the tail.
- (I2.c) Operations that excise an element (§6.6 when it empties `R(e)`, §6.7, §6.8, §6.9, §6.11, §6.12, §6.13) MUST excise it from `insertion_order` AND densely renumber `ord` over the survivors `1..N` in surviving-position order.
- (I2.d) Set operations that introduce new elements (§6.10 union, §6.13 symmetric_difference for the `keys(other) ∖ keys(self)` portion) MUST tail-append those new elements in `other`'s `insertion_order` order, then renumber `ord` densely; see §6.10 / §6.13 and the per-op insertion-order rule table at §4.10's preamble.

### 5.2 Lazy boundary-event index — invariant + version counter

```
version : Int                       // initial 0
event_index : EventIndex?           // nil indicates stale / not yet built
```

`EventIndex` is an immutable snapshot of sorted sweep events, accompanied by auxiliary structures supporting `O(log M + r)` point queries and `O(log M + r)` range queries.

**Concrete design (reference):**

```
EventIndex = struct {
  events  : SortedArray<Event>       // (coordinate, kind, ord) tuples sorted per §4.5 order
  segments: SortedArray<Segment>     // partition of the integer axis into maximal segments where the active set is constant between adjacent transitions
  version : Int                      // snapshot of Rangeable.version
}
Event   = (coord: Int, kind: enum {open, close}, element: Element, ord: Int)
Segment = (lo_inclusive: Int, hi_inclusive: Int, active: List<Element>)
```

**Invariant (I3) — Lazy index synchronisation:**

- (I3.a) If `event_index == nil`, any subsequent query MUST trigger a rebuild.
- (I3.b) If `event_index != nil`, then `event_index.version == this.version` MUST hold; otherwise `event_index` MUST be rebuilt or invalidated to nil before use.
- (I3.c) Any mutation (i.e. one that actually changes `intervals` or `version`) MUST invalidate `event_index` (set it to nil or mark it stale).
- **(I3.d) Version re-check on the cache-write path (normative):** the cache-write path MUST **re-check** `r.version` within the atomic step of "deciding to publish the new build into the cache slot":

  > Let `v_start := r.version` be the snapshot at the start of the build. Within the atomic step that completes the build and writes to the cache, the implementation MUST re-read `r.version` as `v_now`. If `v_now != v_start`, the stale build MUST NOT be written to the cache (it should be discarded; the next query will re-trigger a build).

  This may be realised either (a) via a compare-and-swap (atomic CAS that writes only when `event_index == nil`; or CAS on a `(event_index, expected_version)` tuple) or (b) by serialising the build + write path through a single mutex and re-verifying the version while the lock is held. Implementations SHOULD adopt the mutex variant for simplicity; they MAY adopt the CAS variant to be lock-free.

  This invariant, together with §11 (T3), eliminates the race of "two readers building concurrently and the later writer overwriting an already stale snapshot".

**When the lazy build triggers:**

- The first call to `[i]` / `transitions` where `event_index == nil`.
- When `event_index.version != this.version`.

**Eager invalidate rather than incremental update:**

Rationale: the dominant `Rangeable` workload is "**finish the build phase, then enter a dense query phase**". An incremental update is implementationally complex, and during heavy build-time inserts would unnecessarily rebuild the index repeatedly; lazy + invalidate, by contrast, permits a one-shot `O(M log M)` build, amortised across countless subsequent `O(log M + r)` queries.

### 5.3 Optional dense materialised variant (e)

When the conditions below hold, implementations MAY transparently switch to a **dense per-coordinate active-set materialised array** variant:

- The query workload is known to be "one query per `i ∈ [0, L-1]`".
- `L` and `M` are of similar magnitude (`L ≈ M` or `L = O(M)`).
- The total interval coordinate range `max_hi - min_lo + 1` is at most a constant factor times `L`.

Concrete implementation:

```
materialised : Array<List<Element>>   // length = max_hi - min_lo + 1
// materialised[i - min_lo] = e₁, e₂, ... in §4.5 order
```

**Build:** `O(L + M)`, walking each region of each element and pushing `e` into `materialised[lo..hi]`.

**Point query:** `O(1)` (array access).

**Trade-off:** space `O(L · max_active_at_point)`; `transitions` still requires the sweep events index.

> **Normative:** this (e) variant MUST be only a **transparent substitute** for **§5.1 + §5.2**; the externally observable behavior and ordering MUST be unchanged (§4.5 applies strictly). Implementations MAY heuristically select this variant after the build phase ends, at the first query.

> **Markdown rendering fits this scenario:** L = paragraph character count, typically 2000; M = markup count, typically 50–several hundred; the workload issues one query per character. If `r[i].objs`'s hot path uses the (e) variant, the entire paragraph renders in `O(L)`.

### 5.4 Memory layout and size estimate (informative)

Let `n_e := |R(e)|`, `E := |insertion_order|`, `M := Σ n_e`. In-memory size of the reference structure:

| Field | Size (bytes) |
|---|---|
| `intervals` (Hash/Dict) | `~32 + E · (sizeof(ptr) + sizeof(Array_header))` |
| `Array<Interval>` per element | `~24 (header) + n_e · 16` (`(Int, Int)` tuple) |
| `insertion_order` (Array) | `~24 + E · sizeof(ptr)` |
| `ord` (Hash/Dict) | `~32 + E · (sizeof(ptr) + sizeof(Int))` |
| `event_index.events` | `2M · sizeof(Event)` ≈ `2M · 32` bytes (event includes coord, kind, element ref, ord) |
| `event_index.segments` | `≤ 2M · sizeof(Segment)` worst case; typically `O(M)` |
| `event_index.segments[*].active` | `Σ\|active(seg)\| · sizeof(ptr)` ≤ `2M · sizeof(ptr)` |

**Typical markdown paragraph (L = 2000, M = 50, E = 5):**

- `intervals`: `~32 + 5 · 16 = 112 B`
- 5 `Array<Interval>`s, average `n_e = 10`: `5 · (24 + 10 · 16) = 920 B`
- event_index: `~2 · 50 · 32 = 3200 B` events + `~50 · 24 = 1200 B` segments + active sets `~400 B`
- Total ~6 KB. Entirely acceptable for a build-once-then-query-densely workload.

### 5.5 Serialization considerations (informative, future)

Although v1 does not specify serialization, the following hints are reserved for future cross-language fixture exchange:

- Recommended format: JSON array of `{element_kind, element_payload, intervals: [[lo, hi], ...]}`; the drawback is that the element itself must be serialised by the caller.
- Recommended invariants: output ordered by `insertion_order` ascending; each element's `intervals` is a canonicalised sorted disjoint list.
- This format may serve as the cross-language shared fixture medium for §10 Test #20.

---

## §6  Algorithms (pseudocode — language-neutral)

> The pseudocode is in **language-neutral** form. `bsearch(list, predicate)` denotes "leftmost index satisfying predicate" (lower_bound style). `list[i]` is 0-indexed access; `list[i..j]` is an inclusive slice.

### 6.1 `insert` (normative reference pseudocode)

> **Normative pseudocode.** The following is the normative reference pseudocode for v1; any implementation conforming to this RFC MUST produce externally equivalent behavior (in particular: idempotent inserts do not bump the version, merge ordering, and the (I1) structure of `R(e)`).

```
function insert(rangeable r, element e, int start, int end):
  // 1. Pre-condition (§4.4 D1)
  if start > end:
    raise InvalidIntervalError(start, end)

  // 2. §4.6 (M2) defensive freeze (Ruby shallow freeze; Swift value-type no-op)
  e_frozen := freeze_for_insert(e)

  // 3. First-insert order tracking (§4.5)
  is_first_insert := (e_frozen not in r.intervals)
  if is_first_insert:
    r.intervals[e_frozen] := empty_sorted_list()
    r.insertion_order.append(e_frozen)
    r.ord[e_frozen] := length(r.insertion_order)

  per_element_list := r.intervals[e_frozen]
  lo, hi := start, end

  // 4. Find the leftmost interval that may "touch".
  //    "touch" := overlap or integer adjacency, equivalently `iv.hi + 1 >= lo`
  //    (i.e. succ(iv.hi) >= lo).
  //    Note: MUST NOT be written as `iv.hi >= lo - 1`, because `lo - 1` underflows
  //    when lo == Int.min (C5). Under the Optional<Int> coord model of §4.7 (C4),
  //    `succ` adds 1 for finite hi and yields None when hi == Int.max; `lo` is
  //    always a finite Int (open coords are never None), so the comparison is
  //    always well-defined.
  i := bsearch(per_element_list, λ iv → iv.hi + 1 >= lo)
  initial_i := i

  // 5. Collect all touched entries (do not mutate yet; this enables the fast-path test)
  to_merge := []
  while i < length(per_element_list) and per_element_list[i].lo <= hi + 1:
    to_merge.append(per_element_list[i])
    i += 1

  // 6. **Containment idempotent fast-path** (aligned with §3.2 and Lemma 6.5.B):
  //    if exactly one existing entry is touched and that entry fully contains
  //    [start, end], this insert is an idempotent no-op. R(e) MUST NOT change,
  //    and the version MUST NOT bump.
  //    Test: `to_merge.size == 1 and to_merge[0].lo <= start and end <= to_merge[0].hi`.
  //    If `is_first_insert` is true but execution reaches this branch, that is a
  //    contradiction (the list was empty, so `to_merge` must be empty); thus the
  //    fast-path's precondition naturally excludes the first-insert case.
  if length(to_merge) == 1
     and to_merge[0].lo <= start
     and end <= to_merge[0].hi:
    return    // R(e) unchanged, version unchanged, event_index unchanged (§5.2 I3 still satisfied)

  // 7. Real mutation path: delete the touched entries and insert the merged interval.
  for j in 0 .. length(to_merge) - 1:
    per_element_list.delete_at(initial_i)   // delete_at(initial_i) shifts the rest left
  // Compute merged lo/hi
  for iv in to_merge:
    lo := min(lo, iv.lo)
    hi := max(hi, iv.hi)
  per_element_list.insert_at(initial_i, (lo, hi))

  // 8. Bump the version and invalidate the index (§5.2 I3)
  r.version += 1
  r.event_index := nil
  return
```

**Pre-condition:** `start ≤ end` (D1).
**Post-condition:**
- (P1) `R(e_frozen)` satisfies (I1);
- (P2) `start, end ∈ [lo_after, hi_after]`, where `(lo_after, hi_after)` is the newly inserted or merged interval;
- (P3) `e_frozen ∈ insertion_order`; its `ord` is assigned at the first insert and is invariant thereafter;
- (P4) `version` strictly increases if `R(e)` changes; otherwise `version` is unchanged (**idempotent containment fast-path**);
- (P5) for boundary inputs `lo == Int.min`, `hi == Int.max`, the algorithm MUST NOT trigger underflow / overflow. The predicate `iv.hi + 1 >= lo` in step 4 uses the `succ` semantics of §4.7 (C4) and incurs no underflow (`lo` is a finite Int; `iv.hi + 1` is `Optional<Int>`; `None ≥ any finite` is always true).

**Formal proof of idempotence (P4):** assume `R(e)` already contains `(l, h)` with `l ≤ start ≤ end ≤ h` (containment).
- Step 4: the bsearch predicate `iv.hi + 1 >= start` is true at `(l, h)` (since `h + 1 > end ≥ start ⇒ h + 1 ≥ start`), and `(l, h)` is the leftmost satisfier (the previous entry's `hi'` satisfies `hi' + 1 < l ≤ start`). `initial_i := index of (l, h)`.
- Step 5: the while condition `per_element_list[i].lo ≤ hi + 1` is true at `(l, h)` (`l ≤ end + 1`); append `(l, h)`, `i += 1`. The next entry (if any) has `lo' > h + 1 ≥ end + 1` (strictly greater), so the while loop exits. `to_merge == [(l, h)]`.
- Step 6: `to_merge.size == 1` ✓; `to_merge[0].lo == l ≤ start` ✓; `end ≤ h == to_merge[0].hi` ✓. Fast-path return; no mutation, no version bump. ∎

**Safety note (C5 underflow-free):** the step-4 predicate has been rewritten to `iv.hi + 1 >= lo` (equivalent to the previous `iv.hi >= lo - 1` but without the `lo - 1` operation). When `lo == Int.min`, the previous form `lo - 1 == Int.min - 1` underflows; the new form computes `iv.hi + 1` (which under §4.7 (C4)'s `Optional<Int>` model is `None` when `iv.hi == Int.max`), and the comparison against `lo` is always well-defined. When `iv.hi == Int.max`, `iv.hi + 1 == None > Some(lo)`; the predicate is true, correctly capturing "an existing entry crossing `Int.max` must touch".

**Correctness (I1):**

- (I1.1, sorted ascending lo): the index `i` found by bsearch is the leftmost touch; after deleting the i…j segment and inserting the merged entry at position `i`, prior entries satisfy `hi + 1 < lo` (strictly less, because they do not touch; equivalent to the previous `hi < lo − 1` but underflow-free), and subsequent entries satisfy `lo > new_hi + 1` (strictly greater); ascending lo ordering is therefore preserved.
- (I1.2, no overlap, no adjacency): same; the merged entry has a strict gap > 0 from the entries before and after.
- (I1.3): `lo_new ≤ hi_new` holds (preserved by min/max).

**Complexity:**

- bsearch: `O(log m_e)`, where `m_e = |R(e)|` before this insert.
- while loop: `O(k)`, where k is the number of merged entries.
- list.delete_at + insert_at: worst-case `O(m_e)` for the sorted-array implementation (shifting); `O(log m_e + k)` for a sorted list / skiplist.
- Amortised analysis: see §7.

### 6.2 `getRange`

```
function getRange(r, e):
  if e not in r.intervals:
    return []
  return r.intervals[e]   // copy or read-only view, MAY decide
```

**Post-condition:** the return value satisfies (I1).

**Complexity:** `O(1)` view; `O(m_e)` copy.

### 6.3 `subscript [i]` (point stabbing)

```
function active_at(r, i):
  ensure_event_index_fresh(r)
  // event_index.segments is a sorted list of non-overlapping segments by lo;
  // use bsearch to find the segment containing i.
  s := bsearch_segment(r.event_index.segments, i)
  if s is nil:
    return Slot(objs: [])
  return Slot(objs: s.active)   // already sorted ascending by §4.5 ord

function ensure_event_index_fresh(r):
  if r.event_index == nil or r.event_index.version != r.version:
    r.event_index := build_event_index(r)

function build_event_index(r):
  events := []
  for (e, intervals) in r.intervals:
    for (lo, hi) in intervals:
      events.append( (Some(lo),       open,  e, r.ord[e]) )
      events.append( (succ(Some(hi)), close, e, r.ord[e]) )   // §4.7 (C4): hi == Int.max ⇒ None
  events.sort_by_lex(coordinate asc using §4.7 (C4) total order,
                     open<close,
                     ord asc-for-open / desc-for-close)

  // Linear sweep over events, materialising the active set per segment.
  segments := []
  active := empty_ordered_set_keyed_by_ord()
  prev_coord := nil   // sentinel: "no event seen yet"
  for ev in events:
    // **Invariant**: before the first event, `active` must be empty (each element's
    // open is sequenced before its own close), so when prev_coord == nil, `active`
    // is empty, making the `active not empty` test below trivially false; we will
    // not emit a spurious "(-∞, first_open_coord-1)" segment.
    if ev.coord != prev_coord and active not empty:
      segments.append( (prev_coord, predecessor(ev.coord), snapshot(active)) )
    apply ev to active   // open: insert into active; close: remove
    prev_coord := ev.coord
  // No final segment to emit (after all close events are processed, active = ∅).
  return EventIndex(events, segments, r.version)
```

**Invariant (normative):** the `segments` produced by `build_event_index` contain **no** segment with active = ∅; therefore the first segment's lo equals the coord of the earliest open event (i.e. the smallest `lo` across all elements). Readers need not handle "before the first event" specially.

**Correctness:**

- For any `i`, the set returned by `active_at(r, i)` equals `{ e | ∃[l,h] ∈ R(e), l ≤ i ≤ h }`.
- Proof sketch: the events partition the integer axis into maximal segments, within each of which the active set is constant. The active set of segment `(lo_seg, hi_seg)` equals { all open events − close events accumulated up to and including lo_seg }. Once bsearch locates the segment, the cached active list is read directly.
- If `i` falls outside every segment (i.e. nothing is active), return `[]`.

**Complexity:**

- First call (lazy build): `O(M log M)` sort + `O(M)` segment materialisation.
- Each subsequent query: `O(log |segments| + r)`, where `r = |active at i|`.

### 6.4 `transitions(over:)`

```
function transitions(r, lo_q, hi_q):
  if lo_q > hi_q:
    raise InvalidIntervalError(lo_q, hi_q)
  ensure_event_index_fresh(r)
  // event_index.events is already sorted (per §4.7 (C4) total order: finite Some(_) < None).
  // Find the first event whose coord >= Some(lo_q).
  i_start := bsearch(r.event_index.events, λ ev → ev.coord >= Some(lo_q))
  // Upper bound: query's hi_q + 1. When hi_q == Int.max the upper bound is None;
  // otherwise Some(hi_q + 1). Use the §4.7 (C4) succ: upper := succ(Some(hi_q)).
  upper := succ(Some(hi_q))
  result := []
  i := i_start
  while i < length(events) and events[i].coord <= upper:
    result.append(events[i])
    i += 1
  return result
```

**Note:** comparisons use the `Optional<Int>` total order of §4.7 (C4) (`None > any Some(_)`); no finite arithmetic underflow / overflow occurs. When `hi_q == Int.max`, `upper == None`, and the loop captures every event whose coord is `None` (i.e. the close events of elements whose original `hi == Int.max`).

**Pre-condition:** `lo_q ≤ hi_q`.
**Post-condition:** the returned events are sorted by the normative §4.5 order.

**Complexity:** `O(log M + result_size)`.

### 6.5 Invariant proofs (formal)

#### Lemma 6.5.A (Insert preserves I1)

> If every `intervals[e]` of a `Rangeable r` satisfies (I1) and the precondition `s ≤ t` of `insert(r, e, s, t)` holds, then after the insert completes `intervals[e]` still satisfies (I1).

**Proof.** Let `L := intervals[e]` = `[(l₁, h₁), …, (l_n, h_n)]` before the insert satisfy (I1), i.e. `l₁ ≤ h₁ < l₂ - 1`, `l₂ ≤ h₂ < l₃ - 1`, ….

bsearch finds `i₀` as the leftmost index satisfying `h_(i₀) ≥ s - 1`. Then for all `j < i₀`, `h_j < s - 1`, equivalently `h_j + 1 < s`, i.e. entry `j` does not touch `[s, t]` (strict gap).

The merge loop sweeps right from `i₀`, absorbing every entry with `l_j ≤ t + 1` (overlapping or adjacent to `[lo, hi]`). Suppose it absorbs `j₁, j₁+1, …, j₂`. After the loop ends (with `j₂+1 < n` or `j₂ = n`), the next entry `j₂+1` (if any) satisfies `l_(j₂+1) > hi + 1`, i.e. the gap between `j₂+1` and the new merged interval `[lo, hi]` is > 0.

The newly inserted merged interval is `[lo*, hi*]`, where `lo* = min(s, l_(j₁))` and `hi* = max(t, h_(j₂))`. Since the original `L` is sorted ascending by `lo`, entry `j₁` has the smallest `lo` (within the absorbed range) and entry `j₂` has the largest `hi`, so `lo* ≤ hi*` necessarily holds.

The final list = `[(l₁, h₁), …, (l_(j₁-1), h_(j₁-1)), (lo*, hi*), (l_(j₂+1), h_(j₂+1)), …, (l_n, h_n)]`. Verifying each clause:

- (I1.1, sorted lo): the prefix preserves the original ordering; `(l_(j₁-1), h_(j₁-1))` has `h_(j₁-1) < s - 1 ≤ lo* - 1 < lo*`, so `l_(j₁-1) < lo*`; `(lo*, hi*)` has `hi* < l_(j₂+1) - 1 < l_(j₂+1)`, so `lo* < l_(j₂+1)`. Ascending lo holds.
- (I1.2, no overlap, no adjacency): prefix preserved; the new entry vs. its predecessor: `h_(j₁-1) + 1 < s ≤ lo*`, gap ≥ 1, no touch; the new entry vs. its successor: `hi* + 1 < l_(j₂+1)`, gap ≥ 1, no touch.
- (I1.3): `lo* ≤ hi*` already verified.

Therefore (I1) is preserved. ∎

#### Lemma 6.5.B (Idempotent insert)

> If some entry of `R(e)` strictly contains `(s, t)` (i.e. there exists `(l, h) ∈ R(e)` with `l ≤ s ≤ t ≤ h`), then after `insert(r, e, s, t)`, `R(e)` is unchanged, `insertion_order` is unchanged, and `version` is unchanged (fast-path variant).

**Proof.** The bsearch predicate `iv.hi ≥ s - 1` is true at `(l, h)` (since `h ≥ t ≥ s > s - 1`), and `(l, h)` is the leftmost satisfier. The merge loop enters `(l, h)`: `lo := min(s, l) = l`, `hi := max(t, h) = h`, deletes `(l, h)`. The next iteration checks `next.lo ≤ h + 1`: since the gap between `(l, h)` and the next entry is ≥ 1 (per (I1.2)), `next.lo > h + 1`, and the loop ends. Reinsert `(l, h)`.

If the implementation uses the §6.1 cleaner variant (collect `to_merge` first, then test the fast-path), this scenario hits exactly: `to_merge == [(l, h)]` and `(s, t)` is fully contained ⇒ the fast-path returns without mutation and without bumping the version. ∎

#### Lemma 6.5.C (Active set correctness)

> The set returned by `active_at(r, i)` equals `{e | ∃[l,h] ∈ R(e), l ≤ i ≤ h}`, in ascending `ord(e)` order.

**Proof sketch.**

- The segments of `event_index` partition the integer axis into maximal pieces; within each piece the active set is constant.
- For a piece `(seg_lo, seg_hi)`, its active set is the set of elements opened but not yet closed by the inclusive start of `seg_lo`.
- By the §4.5 sweep-event ordering: opens precede closes within the same coord ⇒ for an element that opens and closes at the same coord at `i`, the open is processed first (added to active) and the close is processed later (removed from active); therefore that element belongs to the active set at instant `i`.
- `bsearch_segment(i)` finds the segment containing `i` and returns its cached active list.
- The active list is sorted by ord at build time (the `snapshot(active)` step in `build_event_index` of §6.3).

By the maximal-segment invariant and the linear correctness of the sweep, the returned set equals the defined set exactly. ∎

#### Lemma 6.5.D (Transitions correctness)

> The events returned by `transitions(r, lo_q, hi_q)` are `{ev ∈ event_index.events | lo_q ≤ ev.coord ≤ hi_q + 1}`, sorted in §4.5 ordering.

**Proof.** `event_index.events` is already sorted by §4.5 ordering at build time. bsearch finds the leftmost `coord ≥ lo_q`, and a linear scan terminates at `coord ≤ hi_q + 1`. The returned sub-array preserves the ordering automatically. ∎

### 6.6 `remove(e, start, end)` (v2 normative)

**R(e)-notation semantics:**

> `R(e) := canonicalize(R(e) ∖ [start, end])`. If the result is `∅`, the element MUST be eagerly pruned per §4.10.

**Pseudocode (mirrors §6.1 numbering convention):**

```
function remove(rangeable r, element e, int start, int end):
  // 1. Pre-condition (§4.4 D1, mirrors insert)
  if start > end:
    raise InvalidIntervalError(start, end)

  // 2. Element-presence fast-path (idempotent no-op, §4.10 N3)
  per_element_list := r.intervals[e]
  if per_element_list is None:
    return            // R(e) is already ∅; version unchanged, event_index unchanged

  // 3. Find the leftmost entry that overlaps [start, end].
  //    "overlap" := iv.hi >= start (strict overlap; adjacency does NOT subtract).
  i := bsearch(per_element_list, λ iv → iv.hi >= start)

  // 4. Quick-exit: nothing in R(e) overlaps [start, end] — idempotent no-op.
  if i == length(per_element_list) or per_element_list[i].lo > end:
    return            // R(e) unchanged, version unchanged, event_index unchanged

  // 5. Sweep all overlapping entries; produce 0..2 replacement entries per
  //    consumed entry depending on whether the cut leaves a left-residual,
  //    a right-residual, both, or neither.
  to_replace_start := i
  replacements := []
  while i < length(per_element_list) and per_element_list[i].lo <= end:
    iv := per_element_list[i]
    // Left residual: (iv.lo, start - 1) — only when iv.lo < start (so start - 1 ≥ iv.lo
    //   ≥ Int.min, no underflow). See "Underflow safety" below.
    if iv.lo < start:
      replacements.append( (iv.lo, start - 1) )
    // Right residual: (end + 1, iv.hi) — only when end < iv.hi (so end + 1 ≤ iv.hi
    //   ≤ Int.max, no overflow). See "Underflow safety" below.
    if end < iv.hi:
      replacements.append( (end + 1, iv.hi) )
    i += 1
  to_replace_end := i

  // 6. Splice: replace the consumed range [to_replace_start, to_replace_end)
  //    with the residual list (length 0..2k where k is consumed-entry count).
  per_element_list.replace_subrange(to_replace_start, to_replace_end,
                                    with replacements)

  // 7. Eager pruning (§4.10 N1) when R(e) is now ∅
  if length(per_element_list) == 0:
    r.intervals.delete(e)
    idx := r.insertion_order.index_of(e)
    r.insertion_order.delete_at(idx)
    r.ord.delete(e)
    // Renumber ord densely for elements at positions > idx
    for f in r.insertion_order[idx ..]:
      r.ord[f] -= 1

  // 8. Version + index invalidation (§5.2 I3.c)
  r.version += 1
  r.event_index := nil
  return
```

**Underflow / overflow safety (dual of §6.1 (P5), normative).** `start - 1` in step 5 is computed only when `iv.lo < start`. Since `iv.lo` is a finite Int satisfying `iv.lo < start`, we have `start > iv.lo ≥ Int.min`, hence `start > Int.min` and `start - 1` is a well-defined finite Int. Symmetrically, `end + 1` is computed only when `end < iv.hi`. Since `iv.hi` is a finite Int satisfying `end < iv.hi`, we have `end < iv.hi ≤ Int.max`, hence `end < Int.max` and `end + 1` is a well-defined finite Int. **The professor MUST verify this dual-symmetry argument explicitly: §6.1 (P5) shows insert is `Int.min`-safe via `iv.hi + 1 ≥ lo`; §6.6 here shows remove is both `Int.min`-safe (via `iv.lo < start` guard on `start - 1`) and `Int.max`-safe (via `end < iv.hi` guard on `end + 1`).**

**Invariant maintenance (proof sketch for I1).** Let `L` satisfy (I1) before the op. After step 6, each consumed `iv = (lo_iv, hi_iv)` is replaced by 0, 1, or 2 sub-intervals strictly inside `[lo_iv, hi_iv]`:
- A left residual `(lo_iv, start - 1)` exists iff `lo_iv < start`; its `hi` is `start - 1 < start`.
- A right residual `(end + 1, hi_iv)` exists iff `end < hi_iv`; its `lo` is `end + 1 > end`.

**(I1.1) sorted ascending lo.** Within a single consumed entry, the left residual has `hi = start - 1 < end + 1 = lo` of the right residual (gap of `end - start + 1 ≥ 1` integer positions). Across consecutive consumed entries `iv_j` and `iv_{j+1}`: `iv_j.hi + 1 < iv_{j+1}.lo` already held by (I1.2), so `iv_j`'s right residual `(end+1, iv_j.hi)` and `iv_{j+1}`'s left residual `(iv_{j+1}.lo, start-1)` satisfy `iv_j.hi < iv_{j+1}.lo`, hence sorted lo. **(I1.2) no overlap, no adjacency.** Consecutive residuals from the same `iv`: `(start - 1) + 1 = start ≤ end < end + 1 = lo_right`, so the gap between left-residual `hi` and right-residual `lo` is `end - start + 1 + 1 ≥ 2`, satisfying `hi_prev + 1 < lo_next`. Cross-entry residuals: inherit (I1.2) from `L`. **(I1.3) `lo ≤ hi`.** Left residual: `lo_iv ≤ start - 1` (since `lo_iv < start` ⇒ `lo_iv ≤ start - 1`); right residual: `end + 1 ≤ hi_iv` (since `end < hi_iv` ⇒ `end + 1 ≤ hi_iv`). Both hold. **(I1.4) non-empty.** If `length(per_element_list) == 0` after step 6, step 7 excises `e`; otherwise non-empty by construction. ∎

**Idempotence and version-bump rule.** If no entry of `R(e)` overlaps `[start, end]` (steps 2 or 4), the op is a no-op: no mutation, **`version` MUST NOT bump**, `event_index` MUST NOT be invalidated. This is the §6.6 dual of §6.1's containment fast-path (Lemma 6.5.B). Per §4.10 (N3) and the v2 version-bump rule, `version` is bumped exactly once when the `DisjointSet` for `e` actually changed.

**Complexity.** `O(log m_e + k)` for the bsearch + sweep (where `k` is the number of overlapped entries); `O(m_e)` worst case for the splice in a sorted-array implementation; `O(E)` for the eager-prune renumbering when the element is excised. The amortised cost over a workload with rare full-removes is dominated by the mutation cost, not the renumber.

**Test cross-references:** §10.B tests #21–#31 (no overlap, exact match, left-only residual, right-only residual, split, multi-entry sweep, full prune, no-op no-bump, `start > end`, `Int.min`, `Int.max`).

### 6.7 `remove(e)` (v2 normative)

**R(e)-notation semantics:**

> `R(e) := ∅`; `e` is eagerly pruned per §4.10 (N1).

**Pseudocode:**

```
function remove_element(rangeable r, element e):
  // 1. Idempotent no-op (§4.10 N3)
  if e not in r.intervals:
    return            // version unchanged, event_index unchanged

  // 2. Excise from intervals
  r.intervals.delete(e)

  // 3. Excise from insertion_order and renumber ord (§4.10 N1)
  idx := r.insertion_order.index_of(e)
  r.insertion_order.delete_at(idx)
  r.ord.delete(e)
  for f in r.insertion_order[idx ..]:
    r.ord[f] -= 1

  // 4. Version + index invalidation (§5.2 I3.c)
  r.version += 1
  r.event_index := nil
  return
```

**Invariant maintenance.** `intervals` and `insertion_order` shrink by exactly one entry; `ord` is densely renumbered. (I1) holds vacuously for the excised element (no longer a key); (I2) holds by step 3; (I3) holds by step 4. ∎

**Idempotence and version-bump rule.** If `e ∉ keys(intervals)` (step 1), the op is a no-op: no mutation, no version bump, no `event_index` invalidation. `version` is bumped exactly once when `e ∈ keys(intervals)`.

**Complexity.** `O(E)` worst case for the linear scan + renumber loop in step 3; `O(1)` for the dictionary delete in step 2. Implementations MAY cache an `idx` map alongside `ord` to drop the linear scan to `O(1)`, but the renumber loop is `O(E)` regardless.

**Test cross-references:** §10.B tests #32 (remove existing element prunes from `count`), #33 (remove non-existent = no-op, no bump), #34 (single-interval element), #35 (many-interval element).

### 6.8 `removeAll() / clear()` (v2 normative)

**R(e)-notation semantics:**

> For all `e`, `R(e) := ∅`. Post-condition matches §3.1's empty-container post-condition.

**Pseudocode:**

```
function clear(rangeable r):
  // 1. Idempotent no-op (§4.10 N3)
  if r.intervals is empty:
    return            // version unchanged, event_index unchanged

  // 2. Reset all element-keyed structures atomically
  r.intervals := empty_map()
  r.insertion_order := empty_list()
  r.ord := empty_map()

  // 3. Version + index invalidation (§5.2 I3.c)
  r.version += 1
  r.event_index := nil
  return
```

**Invariant maintenance.** All three element-keyed structures are simultaneously emptied; (I1)/(I2) hold vacuously (no keys); (I3) holds by step 3. ∎

**Idempotence and version-bump rule.** Clearing an empty container MUST NOT bump version. `version` is bumped exactly once when `count > 0` before the call.

**Complexity.** `O(1)` for releasing references; the underlying GC is responsible for the actual entry-by-entry freeing (which is amortised across the program lifetime, not charged to this op). For implementations that explicitly free per-entry (e.g. C++-style RAII), the cost is `O(E + M)`.

**Test cross-references:** §10.B tests #36 (clear non-empty), #37 (clear empty = no-op, no bump), #38 (post-clear `isEmpty == true`), #39 (post-clear `count == 0`), #40 (insert after clear assigns `ord = 1`).

### 6.9 `removeRanges(start, end)` (v2 normative)

**R(e)-notation semantics:**

> For every `e ∈ keys(self)`, `R(e) := canonicalize(R(e) ∖ [start, end])`. Elements with empty result are eagerly pruned per §4.10. **Atomicity:** a single `version` bump for the entire op; eager pruning happens for every element that becomes empty in this single op.

**Pseudocode:**

```
function remove_ranges(rangeable r, int start, int end):
  // 1. Pre-condition (D1)
  if start > end:
    raise InvalidIntervalError(start, end)

  any_change := false

  // 2. Snapshot the keys so we can mutate intervals while iterating.
  for e in copy(r.insertion_order):
    per_element_list := r.intervals[e]
    // Inline the §6.6 sweep+splice; SUPPRESS per-element version bump
    // and DEFER the eager-prune insertion_order rebuild until step 4.
    (changed, became_empty) := remove_subrange_in_place(per_element_list, start, end)
    if changed:
      any_change := true
      if became_empty:
        r.intervals.delete(e)
        // Note: do NOT delete_at(insertion_order, …) here — that would be O(E²).

  // 3. If nothing changed, no-op (idempotent §4.10 N3)
  if not any_change:
    return            // version unchanged, event_index unchanged

  // 4. Single-pass rebuild of insertion_order + ord (avoids O(E²))
  r.insertion_order := [e for e in r.insertion_order if e in r.intervals]
  r.ord := { r.insertion_order[i]: i + 1 for i in 0 .. length(r.insertion_order) - 1 }

  // 5. Single atomic version bump + index invalidation (§5.2 I3.c)
  r.version += 1
  r.event_index := nil
  return
```

**Atomicity (normative).** The entire op either changes nothing (no version bump, no invalidation) or produces a coherent post-state with **exactly one** version increment. The pre-condition check in step 1 happens before any mutation; step 1 raising `InvalidIntervalError` MUST leave the container completely unchanged. Internal sweep failures (e.g. allocator OOM mid-loop) are out-of-scope of this RFC; implementations SHOULD use a copy-then-swap pattern to maintain atomicity under such failures (analogous to §6.10's "in-place via copy-then-swap" rule).

**Underflow / overflow safety.** Inherited from §6.6 step 5 (per-element sweep is identical to §6.6's).

**Invariant maintenance.** Each per-element sweep preserves (I1) per the §6.6 proof. Step 4 rebuilds `insertion_order` and `ord` densely from the surviving keys, restoring (I2). Step 5 invalidates the index, restoring (I3.c). ∎

**Idempotence and version-bump rule.** No-op if no element's `R(e)` overlaps `[start, end]` (i.e. all per-element sweeps return `changed = false`). `version` MUST be bumped exactly once when `any_change` is true; not once per element.

**Complexity.** `O((Σ_e log m_e) + Σ_e k_e + E)` where `k_e` is the entries consumed for element `e`. The single-pass `insertion_order` rebuild in step 4 is `O(E)` (vs. `O(E²)` if naively done with `delete_at` per element). `event_index` invalidation is `O(1)`.

**Test cross-references:** §10.B tests #41 (broad range hits multiple elements), #42 (no overlap = no-op, no bump), #43 (range fully covers some elements, prunes them, leaves others; single bump verified).

### 6.10 `union(other)` (v2 normative)

**R(e)-notation semantics:**

> For all `e ∈ keys(self) ∪ keys(other)`, `R_result(e) := canonicalize(R_self(e) ∪ R_other(e))`. **Insertion-order rule:** preserve `self`'s `insertion_order`; for elements `e ∈ keys(other) ∖ keys(self)`, append in `other`'s `insertion_order` order at the tail.

**Pseudocode:**

```
function union(rangeable r_self, rangeable r_other):
  r_out := Rangeable.empty()

  // 1. Walk self's insertion_order: every key in self appears, possibly extended by other.
  for e in r_self.insertion_order:
    list_self  := r_self.intervals[e]
    list_other := r_other.intervals[e]   // may be None
    merged := merge_disjoint_lists(list_self, list_other ?? empty_list)
    // length(merged) > 0 always (list_self non-empty by I1.4); no prune needed.
    r_out.intervals[e]      := merged
    r_out.insertion_order.append(e)
    r_out.ord[e]            := length(r_out.insertion_order)

  // 2. Tail-append keys in other ∖ self in other's insertion_order order.
  for e in r_other.insertion_order:
    if e in r_self.intervals: continue   // already handled in step 1
    merged := copy(r_other.intervals[e])
    r_out.intervals[e]      := merged
    r_out.insertion_order.append(e)
    r_out.ord[e]            := length(r_out.insertion_order)

  // 3. Fresh container starts at version = 0; event_index lazily built on first query.
  r_out.version := 0
  return r_out

function merge_disjoint_lists(L1, L2):
  // Both inputs are (I1)-canonical. Two-pointer sweep merging by lo, with adjacency-collapse.
  out := []
  i, j := 0, 0
  while i < length(L1) and j < length(L2):
    if L1[i].lo <= L2[j].lo:
      append_or_merge(out, L1[i]); i += 1
    else:
      append_or_merge(out, L2[j]); j += 1
  while i < length(L1): append_or_merge(out, L1[i]); i += 1
  while j < length(L2): append_or_merge(out, L2[j]); j += 1
  return out

function append_or_merge(out, iv):
  if length(out) == 0 or out.last.hi + 1 < iv.lo:
    out.append(iv)
  else:
    // Overlap or integer adjacency: extend the last entry's hi.
    out.last.hi := max(out.last.hi, iv.hi)
```

**In-place form.** The mutating spelling each language exposes (e.g. Swift `formUnion`, Ruby `union!`, Python `update`) MUST use a copy-then-swap (or assign result into `self`) so the operation is atomic w.r.t. errors; `version` MUST bump **at most once** per call. If `result == self` structurally (i.e. every `e ∈ keys(other)` satisfies `R_other(e) ⊆ R_self(e)` and `keys(other) ⊆ keys(self)`), the mutating form MUST NOT bump version (idempotence dual of §3.2).

**Invariant maintenance.** `merge_disjoint_lists` produces an (I1)-canonical list by construction (sorted by `lo` via two-pointer; adjacency-collapse via `append_or_merge`). `r_out.insertion_order` ⊇ `keys(r_out.intervals)` per (I2.a) since every `intervals` key is added with a matching `insertion_order` append. `ord` is dense by construction (assigned `length(insertion_order)` immediately after each append).

**Idempotence and version-bump rule (normative).** For the non-mutating form, the source `version` MUST be unchanged; the new `Rangeable` starts at `version = 0`. For the mutating form, `version` MUST bump iff the result is structurally `!= self` (any keys added, any `R(e)` enlarged). The structural-equality check MAY be implemented either by post-construction comparison or by tracking a `dirty` flag during the sweep (any `append_or_merge` that produces a different result, or any new key addition, sets `dirty = true`).

**Complexity.** `O(M_self + M_other + E_self + E_other)`. The professor MUST verify that `merge_disjoint_lists` is `O(m_e_self + m_e_other)` per element via the two-pointer sweep — NOT `O(m_e_self · m_e_other)` from a naive nested loop. **This is the load-bearing performance claim of §7.7.**

**Test cross-references:** §10.C tests #44 (disjoint elements), #45 (overlapping intervals), #46 (adjacency-merge), #47 (idempotent subset), #48 (union of two empties), #49 (union with self), #50 (insertion_order tail-append).

### 6.11 `intersect(other)` (v2 normative)

**R(e)-notation semantics:**

> For all `e ∈ keys(self) ∩ keys(other)`, `R_result(e) := canonicalize(R_self(e) ∩ R_other(e))`. Elements with empty result are eagerly pruned per §4.10. **Insertion-order rule:** preserve `self`'s `insertion_order` over surviving keys (i.e. `keys(self) ∩ keys(other)` minus pruned-empty); densely renumber `ord`.

**Pseudocode:**

```
function intersect(rangeable r_self, rangeable r_other):
  r_out := Rangeable.empty()

  for e in r_self.insertion_order:
    if e not in r_other.intervals: continue        // not in keys(other), drop
    intersected := intersect_disjoint_lists(r_self.intervals[e],
                                            r_other.intervals[e])
    if length(intersected) == 0: continue          // eager prune (§4.10)
    r_out.intervals[e]      := intersected
    r_out.insertion_order.append(e)
    r_out.ord[e]            := length(r_out.insertion_order)

  r_out.version := 0
  return r_out

function intersect_disjoint_lists(L1, L2):
  // Both inputs (I1)-canonical. Two-pointer sweep of pairwise intersections.
  out := []
  i, j := 0, 0
  while i < length(L1) and j < length(L2):
    lo := max(L1[i].lo, L2[j].lo)
    hi := min(L1[i].hi, L2[j].hi)
    if lo <= hi:
      out.append( (lo, hi) )
    if L1[i].hi <= L2[j].hi: i += 1
    else: j += 1
  return out
```

**Lemma 6.11.A (intersect preserves (I1) without adjacency-collapse — professor MUST verify).** Suppose `intersect_disjoint_lists` produces two consecutive output entries `(lo_a, hi_a)` and `(lo_b, hi_b)` from `L1[i_a] ∩ L2[j_a]` and `L1[i_b] ∩ L2[j_b]` for some `i_a ≤ i_b`, `j_a ≤ j_b` (monotone advance of `i` and `j`). At least one of `(i_b, j_b)` strictly exceeds `(i_a, j_a)` (else the loop did not advance). WLOG `i_b > i_a`. Then `lo_b ≥ L1[i_b].lo ≥ L1[i_a].hi + 2 ≥ hi_a + 2` (using (I1.2) on `L1`), so `hi_a + 1 < lo_b`. (I1.2) holds on the output without any explicit adjacency-collapse step. ∎

**Invariant maintenance.** Per Lemma 6.11.A the output is (I1)-canonical for each element. Eager pruning (`continue` on `length == 0`) ensures (I1.4). `insertion_order` and `ord` are densely renumbered by construction (only appended when `intersected` is non-empty).

**Idempotence and version-bump rule.** Non-mutating form: source unchanged, fresh `version = 0`. Mutating form (e.g. `formIntersection`, `intersect!`, `intersection_update`): version bumps iff the result is structurally `!= self` (any keys dropped, any `R(e)` shrunk).

**Complexity.** `O(M_self + M_other)` total via the two-pointer sweep. NOT `O(M_self · M_other)`.

**Test cross-references:** §10.D tests #51 (no shared elements), #52 (shared overlapping), #53 (shared disjoint → element pruned), #54 (intersect with self), #55 (intersect with empty), #56 (multiple sub-intervals), #57 (insertion_order preservation + dense ord).

### 6.12 `difference(other)` (v2 normative)

**R(e)-notation semantics:**

> For all `e ∈ keys(self)`, `R_result(e) := canonicalize(R_self(e) ∖ R_other(e))`. Elements with empty result are eagerly pruned per §4.10. **Insertion-order rule:** preserve `self`'s `insertion_order` over surviving keys; densely renumber `ord`.

**Pseudocode:**

```
function difference(rangeable r_self, rangeable r_other):
  r_out := Rangeable.empty()

  for e in r_self.insertion_order:
    list_self  := r_self.intervals[e]
    list_other := r_other.intervals[e]   // may be None
    if list_other is None or length(list_other) == 0:
      remaining := copy(list_self)
    else:
      remaining := subtract_disjoint_lists(list_self, list_other)
    if length(remaining) == 0: continue   // eager prune (§4.10)
    r_out.intervals[e]      := remaining
    r_out.insertion_order.append(e)
    r_out.ord[e]            := length(r_out.insertion_order)

  r_out.version := 0
  return r_out

function subtract_disjoint_lists(L_a, L_b):
  // Both inputs (I1)-canonical. Two-pointer sweep, repeatedly cutting an
  // "in-flight current" entry from L_a by overlapping L_b entries.
  out := []
  i, j := 0, 0
  current_lo, current_hi := nil, nil
  while i < length(L_a):
    if current_lo is nil:
      current_lo, current_hi := L_a[i].lo, L_a[i].hi
    // Skip L_b entries strictly before the current entry.
    while j < length(L_b) and L_b[j].hi < current_lo:
      j += 1
    if j == length(L_b) or L_b[j].lo > current_hi:
      // No more L_b cuts on this current entry; commit and advance i.
      out.append( (current_lo, current_hi) )
      i += 1
      current_lo, current_hi := nil, nil
      continue
    // L_b[j] overlaps [current_lo, current_hi]; cut.
    if L_b[j].lo > current_lo:
      // Left residual: (current_lo, L_b[j].lo - 1)
      out.append( (current_lo, L_b[j].lo - 1) )
    if L_b[j].hi < current_hi:
      // Right residual remains current; advance j.
      current_lo := L_b[j].hi + 1
      j += 1
    else:
      // L_b[j] swallows the rest of the current entry; advance i.
      i += 1
      current_lo, current_hi := nil, nil
  return out
```

**Underflow / overflow safety (dual of §6.1 (P5) and §6.6, normative).** `L_b[j].lo - 1` is computed only when `L_b[j].lo > current_lo`, and `current_lo` is a finite Int, so `L_b[j].lo > Int.min` (hence `L_b[j].lo - 1` is well-defined). `L_b[j].hi + 1` is computed only when `L_b[j].hi < current_hi`, and `current_hi` is a finite Int, so `L_b[j].hi < Int.max` (hence `L_b[j].hi + 1` is well-defined). **The professor MUST verify this dual symmetry.**

**Invariant maintenance.** Every output entry is a sub-interval of some `L_a[i]`, separated from the next output entry by either a hole (L_b cut) or an L_a-internal gap (inherited (I1.2) from `L_a`). Within the same `L_a[i]`, residuals are separated by `L_b[j]`'s body, which has length ≥ 1, so `(L_b[j].lo - 1) + 1 = L_b[j].lo ≤ L_b[j].hi < L_b[j].hi + 1`, satisfying (I1.2). Cross-`L_a[i]` consecutive outputs inherit (I1.2) from `L_a`. (I1.3) holds by the cut conditions.

**Idempotence and version-bump rule.** Non-mutating: fresh `version = 0`. Mutating (e.g. `subtract`, `difference!`, `difference_update`): bump iff the result is structurally `!= self` (any `R(e)` shrunk).

**Complexity.** `O(M_self + M_other)` total via the two-pointer sweep.

**Equivalence to per-element `remove` (informative).** `r.difference(other)` is structurally equivalent to the **per-element** reduction `for e in keys(other): for ((lo, hi) in other.getRange(of: e)) r.remove(e, lo, hi)` (modulo the version-bump count: `difference` bumps once total, `remove`-loop bumps once per call with at least one mutation; but the post-state structure is identical). Test #65 verifies this equivalence as a cross-validation probe.

> **Caveat (informative, normative correction in v2.0 patch).** The equivalence is to `remove(e, lo, hi)` (per-key surgical erase, §6.6), NOT to `removeRanges(lo, hi)` (cross-element erase, §6.9). `removeRanges(lo, hi)` cuts `[lo, hi]` out of every `R(e')`; `difference(other)` cuts only `R_other(e)` out of the matching `R_self(e)`. The two diverge whenever `other` has multiple keys whose intervals would, when flattened and applied via `removeRanges`, cross-pollute keys other than their own. Use the per-key `remove` reduction for correct cross-validation.

**Test cross-references:** §10.E tests #58 (subtract disjoint), #59 (subtract self = empty), #60 (left residuals), #61 (right residuals), #62 (split = both residuals), #63 (multi-`L_a` span), #64 (insertion_order preservation), #65 (difference = removeRanges-loop equivalence).

### 6.13 `symmetric_difference(other)` (v2 normative)

**R(e)-notation semantics:**

> `R_result(e) := canonicalize(R_self(e) △ R_other(e))` for all `e ∈ keys(self) ∪ keys(other)`. Elements with empty result are eagerly pruned per §4.10. **Insertion-order rule:** preserve `self`'s `insertion_order` for `e ∈ keys(self)`; tail-append `e ∈ keys(other) ∖ keys(self)` in `other`'s `insertion_order` order; densely renumber `ord`. **Algebraic identity (also normative):** `R_self(e) △ R_other(e) = (R_self(e) ∖ R_other(e)) ∪ (R_other(e) ∖ R_self(e))` per element.

**Pseudocode:**

```
function symmetric_difference(rangeable r_self, rangeable r_other):
  r_out := Rangeable.empty()

  // 1. self-primary keys: for each e ∈ keys(self), compute the per-element
  //    sym-diff via the (a ∖ b) ∪ (b ∖ a) identity, then merge the two
  //    one-sided residuals (which CAN be adjacent — see "Why merge required" below).
  for e in r_self.insertion_order:
    list_self  := r_self.intervals[e]
    list_other := r_other.intervals[e] ?? empty_list
    a := subtract_disjoint_lists(list_self,  list_other)   // self ∖ other
    b := subtract_disjoint_lists(list_other, list_self)    // other ∖ self
    sym := merge_disjoint_lists(a, b)                      // adjacency-collapse safe
    if length(sym) == 0: continue                          // eager prune (§4.10)
    r_out.intervals[e]      := sym
    r_out.insertion_order.append(e)
    r_out.ord[e]            := length(r_out.insertion_order)

  // 2. other-only keys: copy R_other(e) directly (b ∖ a degenerates to b when a is empty).
  for e in r_other.insertion_order:
    if e in r_self.intervals: continue       // already handled in step 1
    sym := copy(r_other.intervals[e])
    if length(sym) == 0: continue            // unreachable since I1.4, but defensive
    r_out.intervals[e]      := sym
    r_out.insertion_order.append(e)
    r_out.ord[e]            := length(r_out.insertion_order)

  r_out.version := 0
  return r_out
```

**Why `merge_disjoint_lists` is required (normative; not a sorted-concat).** A naive reading might assume `a` and `b` are always disjoint AND non-adjacent, so a sorted concat would suffice. But the two one-sided residuals can be **adjacent** (gap of exactly 1 integer position), so the `(I1.2)` invariant requires `merge_disjoint_lists`'s adjacency-collapse step.

**Worked example (the adjacency case).** Take `R_self(e) = [(0, 5)]` and `R_other(e) = [(2, 8), (10, 15)]`:
- `a = subtract(self, other) = [(0, 1)]` (left residual after subtracting `(2, 8)`; the right residual `(9, 5)` is empty since `5 < 9`).
- `b = subtract(other, self) = [(6, 8), (10, 15)]` (`(2, 8) ∖ (0, 5) = (6, 8)`; `(10, 15)` is untouched by `(0, 5)`).
- `a` and `b` are NOT adjacent here: `1 + 1 = 2 < 6`. Sorted concat would happen to be correct. Output `sym = [(0, 1), (6, 8), (10, 15)]`. ✓

**Worked example (the truly adjacent case).** Take `R_self(e) = [(0, 5)]` and `R_other(e) = [(6, 10)]`:
- `a = [(0, 5)]` (`(0, 5)` is disjoint from `(6, 10)`, no cut).
- `b = [(6, 10)]` (symmetric).
- `a.last.hi + 1 = 6 == b.first.lo`. **Adjacent.** Sorted concat would produce `[(0, 5), (6, 10)]` which violates (I1.2); `merge_disjoint_lists` correctly collapses to `[(0, 10)]`. ✓
- Sanity check: `R_self △ R_other = (R_self ∪ R_other) ∖ (R_self ∩ R_other) = [(0, 10)] ∖ ∅ = [(0, 10)]`. ✓ matches.

**This is why `merge_disjoint_lists` (with its `append_or_merge` adjacency-collapse) is required and a sorted-concat is insufficient.**

**Invariant maintenance.** Each `subtract_disjoint_lists` call produces an (I1)-canonical list (per §6.12). `merge_disjoint_lists` then produces an (I1)-canonical list (per §6.10). Eager pruning ensures (I1.4). `insertion_order` and `ord` are densely renumbered by construction.

**Idempotence and version-bump rule.** Non-mutating: source unchanged, fresh `version = 0`. Mutating (e.g. `formSymmetricDifference`, `symmetric_difference!`, `symmetric_difference_update`): bump iff the result is structurally `!= self`. Note `r.symmetric_difference(r) == empty Rangeable`; the mutating form on `self` MUST clear all keys and bump version once.

**Complexity.** `O(M_self + M_other)` per element-loop iteration, summed to `O(M_self + M_other)` total. Each per-element call invokes two `subtract_disjoint_lists` (each `O(m_e_self + m_e_other)`) plus one `merge_disjoint_lists` (also `O(m_e_self + m_e_other)`), giving per-element `O(m_e_self + m_e_other)`; summed over keys, total is `O(M_self + M_other)`.

**Algebraic cross-validation (normative).** Implementations MAY validate against the dual identity `A △ B = (A ∪ B) ∖ (A ∩ B)`. Both forms MUST produce per-element `R(e)` byte-identical results; they MAY produce different `insertion_order` if the implementer is sloppy, so the canonical `insertion_order` rule (self-primary then tail-append) is the **only** spec-compliant ordering — see §14.9 amendment for justification.

**Test cross-references:** §10.F tests #66 (sym-diff with empty = self), #67 (sym-diff with self = empty), #68 (per-element residuals from both sides), #69 (commutativity modulo insertion_order), #70 (associativity), #71 (insertion_order tail-append).

---

## §7  Complexity Analysis

> This section gives **formal** worst-case, amortised, and output-sensitive bounds, and uses the potential method (CLRS Ch. 17) to prove that amortised insert is `O(log m_e)` under a typical build-phase workload.

### 7.1 Worst-case insert

Let `m_e := |R(e)|` before this insert. Worst-case time:

| Sub-step | Time |
|---|---|
| `start > end` check | `O(1)` |
| Map lookup `intervals[e]` | `O(1)` amortised (hash) |
| bsearch | `O(log m_e)` |
| while loop merging `k` entries | `O(k)` comparisons + `O(k · m_e)` for Array shifting, `O(k log m_e)` for a BBST |
| insert_at | `O(m_e)` Array / `O(log m_e)` BBST |
| event_index invalidation | `O(1)` |

Therefore the worst case is `O(m_e + k)` for the sorted-array reference implementation, and `O(log m_e + k)` for a sorted-tree implementation.

### 7.2 Amortised insert (Φ-potential argument)

This section gives an **amortised analysis** of `insert`. We use the potential method of CLRS §17.3 (the dual form of Tarjan's 1985 credit / banker's argument): the amortised cost of each operation is defined as

$$
\hat c_i \;:=\; c_i + \Phi(D_i) - \Phi(D_{i-1})
$$

where `c_i` is op `i`'s actual cost and `Φ` is a potential function on the state. For any sequence `op_1, op_2, …, op_n`:

$$
\sum_{i=1}^{n} \hat c_i \;=\; \sum_{i=1}^{n} c_i + \Phi(D_n) - \Phi(D_0).
$$

If `Φ(D_0) = 0` and `Φ ≥ 0`, then `Σ c_i ≤ Σ ĉ_i`, so the amortised bound is an upper bound on the actual sum.

#### 7.2.1 Setup and actual-cost model

We use a sorted Array as the reference data structure (a natural fit for both languages' standard libraries). For each `insert(e, s, t)`:

- Let `m_e := |R(e)|` before the insert.
- Both the bsearch and the linear merge-loop comparisons are `O(\log m_e + k)`, where `k` is the number of existing entries absorbed (merged).
- The Array's `delete_at(i) × k` plus `insert_at(i, ⋅)` has total shift cost `O((m_e - i_0) + k) = O(m_e)` worst case, where `i_0` is the position found by bsearch.

Hence the actual cost satisfies:

$$
c_i \;\le\; c_1 \cdot \big(\log m_e + k + (m_e - i_0)\big).
$$

#### 7.2.2 Choice of potential function (two-term potential)

We use a **two-term potential** to absorb the merge cost and the end-shift cost separately into Φ:

$$
\Phi(D) \;:=\; \alpha \cdot M(D) \;+\; \beta \cdot \mathrm{TailSum}(D),
$$

where:
- `M(D) := Σ_e |R(e)|`, the total number of merged entries across the whole container;
- `TailSum(D) := Σ_e (m_e − i_0_e^*)`, where `i_0_e^* ∈ [0, m_e]` is the bsearch landing position of the **most recent insert** into `e`'s list (initialised to `m_e`, the end of the list, so that `m_e − i_0_e^* = 0`). Intuitively, TailSum bounds, from above, the cumulative number of entries that would need to be shifted if subsequent inserts continued to land near the front.
- `α, β` are constants to be chosen; below we set `α := 2c_1` and `β := 2c_1`.

**Credit intuition:** each entry, on entering the list, carries `α + β` credits;
- the `α` portion: pays for the comparison + free cost when this entry is later merge-deleted.
- the `β` portion: pays the amortised one-time shift cost when a later insert "end-shifts past" this entry.

**Properties:**
- `Φ(D_0) = 0` (empty container: M = 0, TailSum = 0).
- `Φ(D) ≥ 0` always (M ≥ 0, TailSum ≥ 0; `m_e − i_0_e^* ∈ [0, m_e]`).

#### 7.2.3 Amortised cost derivation (two-term Φ)

Let op `i` be the insert `insert(e, s, t)`, with `m_e := |R(e)|` before. Let `i_0` be the bsearch landing index, let the merge loop absorb `k` entries, and let `m_e' = m_e − k + 1` after the merge.

**Actual cost** (per §7.2.1):
$$
c_i \;\le\; c_1 \cdot \big(\log m_e \;+\; k \;+\; (m_e - i_0)\big).
$$

**ΔM:** `M' − M = m_e' − m_e = 1 − k`, so `α · ΔM = α(1 − k)`.

**ΔTailSum:** for `e' ≠ e`, both `m_(e')` and `i_0_(e')^*` are unchanged, so `Σ_(e'≠e) (m_(e') − i_0_(e')^*) = const`. For `e`:
- Let op `i`'s **new** `i_0_e^*` := `i_0` (the bsearch landing index); the new `m_e' = m_e − k + 1`.
- Let op `i`'s **old** `i_0_e^*` := `i_0_e^{*\text{old}}`; the gap `m_e − i_0_e^{*\text{old}} ≥ 0` (a ≥ 0 invariant; the initial value for an `e` that has never been inserted is `0` per init).
- We take the upper bound only (loose-but-correct):
  $$
  \Delta\mathrm{TailSum}_e \;=\; (m_e' - i_0) \;-\; (m_e - i_0_e^{*\text{old}}) \;\le\; (m_e' - i_0).
  $$
- And `m_e' − i_0 = (m_e − k + 1) − i_0`.

Therefore `β · ΔTailSum ≤ β · (m_e − k + 1 − i_0)`.

**ΔΦ:**
$$
\Delta\Phi \;\le\; \alpha(1 − k) \;+\; \beta \cdot (m_e − i_0 + 1 − k).
$$

**Amortised cost:**
$$
\hat c_i \;=\; c_i + \Delta\Phi \;\le\; c_1 \log m_e + c_1 k + c_1 (m_e − i_0)
                                            + \alpha (1 − k) + \beta (m_e − i_0 + 1 − k).
$$

Choosing `α := 2 c_1` and `β := −c_1`... that won't do, since `β` must be non-negative (otherwise Φ ≥ 0 is broken). Instead we choose `β := c_1` and pay the end-shift cost `c_1(m_e − i_0)` **out of the credit released by ΔTailSum**, rewriting as:

> **Restated credit mechanism:** each entry, on entering the list, deposits `(α + β)` credits. When the entry is merge-deleted (k entries each release `α + β` credits), the released `α` portion pays this insert's `c_1 · k` (comparison + free); the released `β · k` portion pays the shift cost of *non-merged* entries that are end-shifted past.
>
> This mechanism is equivalent to releasing the `(m_e − i_0)` shift cost from a decrement of `TailSum`, but is clearer in the ΔΦ expression below:

Adopt the rewritten form `Φ' := α · M + β · TailSum`, with a "dual deposit" (α + β) per entry. Per op:
- New entry deposit: `+α + β` (once).
- The `k` merged entries release: `+α k + β k` (used to pay the merge `c_1 k` + the end-shift `c_1 (m_e − i_0)`).
- Net credit consumed by the merge cost: `c_1 k` ≤ `α k` (taking `α := c_1`).
- Net credit consumed by the end-shift cost: `c_1 (m_e − i_0)` ≤ `β · k_(touched-by-shift)` — no, this is not right; it should be `β · (m_e − i_0)` taken directly from the TailSum decrement.

We adopt a more direct form: use the *decrement* of `β · TailSum` as the release for the shift. Concretely:

$$
\Delta(\beta · \mathrm{TailSum}_e) \;=\; \beta \cdot \big[(m_e' - i_0) - (m_e - i_0_e^{*\text{old}})\big].
$$

In the worst case "always end-insert at the front", `i_0 = 0`, `m_e' − i_0 = m_e − k + 1`, `m_e − i_0_e^{*\text{old}} = 0` (initially zero, or the previous insert landed at the end of the list); `Δ(β·TailSum_e) = β(m_e − k + 1)`. **This is in the opposite direction from the desired release `−β · (m_e − i_0)`**, so the two-term Φ is monotonically increasing along the head-insert sequence — Φ itself is climbing, and the shift cost is prepaid by the increase in Φ:

$$
\hat c_i \;=\; c_i + \Delta\Phi \;\le\; c_1 \log m_e + c_1 k + c_1 (m_e − i_0)
                                          + \alpha(1 − k) + \beta(m_e − i_0 + 1 − k).
$$

Choosing `α := 2 c_1`, `β := 2 c_1`:
- `c_1 k + α(1 − k) = c_1 k + 2 c_1 − 2 c_1 k = 2 c_1 − c_1 k ≤ 2 c_1` (k ≥ 0).
- `c_1 (m_e − i_0) + β(m_e − i_0 + 1 − k) = c_1 (m_e − i_0) + 2 c_1 (m_e − i_0) + 2 c_1 − 2 c_1 k = 3 c_1 (m_e − i_0) + 2 c_1 − 2 c_1 k`.

Summing: `ĉ_i ≤ c_1 log m_e + 3 c_1 (m_e − i_0) + 4 c_1 − 3 c_1 k`.

`(m_e − i_0)` is still not eliminated — this exactly captures: **a single op's end-shift cost cannot be cancelled by its own ΔΦ; it must be amortised across ops**. We instead use the following *bookkeeping* proof (accounting form, equivalent to the two-term-Φ potential method):

**Proof by deposit-on-insert / pay-on-shift accounting.**

For any op `i`, decompose the actual cost `c_i = c_1 log m_e + c_1 k + c_1 (m_e − i_0)` into three parts:

1. **`c_1 log m_e` (bsearch)**: charged directly to op `i`'s amortised budget.
2. **`c_1 k` (merge loop)**: each of the `k` deleted entries releases `c_1` credit (each entry deposited `c_1` "for-merge-on-death" when it was originally inserted).
3. **`c_1 (m_e − i_0)` (end shift)**: each op `i` deposits a **new** `c_1` credit on each of the `(m_e − i_0)` entries it end-shifts past (Σ = `c_1(m_e − i_0)` new deposit); meanwhile the *current* shift cost is paid by the `c_1` credits previously deposited at those `(m_e − i_0)` entries by their own past ops.

The crux of this accounting: **the number of times each entry e_j is end-shifted in its lifetime ≤ the number of subsequent ops that wrote in front of it**. But every op that writes in front also deposits a fresh credit at e_j (the "new deposit" in step 3), so each shift of e_j is paid by some matching credit.

Formally: an entry `e_j` is inserted at op `t_j`; subsequently, for any op `t_j+1, t_j+2, …` whose `i_0 ≤ index(e_j)`, e_j is shifted past once. On each such event the op deposits `c_1` credit at e_j, which e_j uses to pay for that shift. Total deposit at e_j = (#shifts past e_j) × c_1 = the actual shift cost.

Thus amortised:
$$
\hat c_i \;=\; \underbrace{c_1 \log m_e}_{\text{bsearch}} + \underbrace{0}_{\text{merge by released credit}} + \underbrace{c_1 (m_e − i_0)}_{\text{deposit for future shifts}} + 0 = c_1 (\log m_e + (m_e − i_0)).
$$

But `(m_e − i_0)` can still reach `m_e`! This is because **per-op amortised cost is not individually bounded; only over-sequence amortised is**: for any sequence of `n` ops, total deposit `= Σ c_1 (m_e − i_0)_i`, an upper bound on the total "end-shift work" across ops; but each entry e_j's deposit is consumed only once (when it is shifted past); so total deposit ≤ `c_1 · M(final) · (number of times each entry is shifted)`.

**Explicit run on the worst-case "always insert at the front" sequence:**

Let `e` be a fixed element; op `i` performs `insert(e, s_i, t_i)` with `s_1 > t_1 + 2 > s_2 > t_2 + 2 > s_3 > ...` (strictly decreasing, no touch, no merge). Then for op `i`:
- `m_e = i − 1` (each of the prior `i − 1` ops added one entry).
- `i_0 = 0` (bsearch finds the leftmost; the new entry goes at the front).
- `k = 0` (no merge).
- Actual shift cost = `c_1 · (m_e − i_0) = c_1 (i − 1)`.

Total over `n` ops: `Σ_{i=1}^n c_1 (i − 1) = c_1 · n(n−1)/2 = O(n²)`.

In the amortised budget Σ ĉ_i: each op pre-deposits `c_1 (m_e − i_0) = c_1 (i − 1)`, summing to `c_1 n(n−1)/2`, which exactly matches the actual cost. But the per-op amortised cost `(i − 1)` is not `O(log m_e)`.

**This shows the original Lemma 7.2.A's `O(log m_e)` amortised bound does not hold on a head-insert sequence.** The corrected bound is:

> **Lemma 7.2.A* (Corrected Amortised Insert Bound, sorted-array reference).** Without remove operations, with sorted Array as the reference structure, the total time of `n` insert ops is `Σ c_i = O(Σ_i \log m_{e_i} + Σ_i k_i + W_n)`, where `W_n := Σ_i (m_{e_i} − i_{0,i})` is the input-dependent "end-shift work" term. For a balanced workload (inserts roughly random / appended at the end / not persistently head-inserting), `W_n = O(n)` and the amortised bound `O(\log m_e)` holds; for the worst-case head-insert sequence, the amortised cost degrades to `O(m_e)` per op, `O(M²)` total — an **inherent** property of sorted arrays (Ω(M) shift on each prepend).
>
> **Proof.** Σ c_i ≤ c_1 (Σ log m_e + Σ k_i + Σ (m_e − i_0)_i). The first two terms: bsearch and merge cost are amortised by deposit-on-insert / release-on-merge of `c_1` credit per entry, totalling `O(Σ log m_e + 2 c_1 · Σ_{ops creating an entry})` = `O(Σ log m_e + n)`. The third term: `W_n` is input-dependent, at Θ(n²) in the head-insert worst case. ∎

#### 7.2.4 Explicit run of the worst-case head-insert sequence (Φ computation)

Take Φ' := `α M + β · TailSum`, `α := 2c_1`, `β := 2c_1`, on the head-insert sequence above.

| op i | m_e (before) | i_0 | k | m_e' (after) | TailSum (before) | TailSum (after) | Δ M | Δ TailSum |
|---|---|---|---|---|---|---|---|---|
| 1 | 0 | 0 | 0 | 1 | 0 | 1 | +1 | +1 |
| 2 | 1 | 0 | 0 | 2 | 1 | 2 | +1 | +1 |
| 3 | 2 | 0 | 0 | 3 | 2 | 3 | +1 | +1 |
| ... | ... | 0 | 0 | ... | ... | ... | +1 | +1 |
| i | i−1 | 0 | 0 | i | i−1 | i | +1 | +1 |

(Note: for head-inserts, op i's "new `i_0_e^*` := 0", so the upper bound on the next op's `(m_e − i_0)` would be `m_e' − 0 = m_e'`; in the table we record TailSum as TailSum_e = m_e − i_0_e^* = m_e − 0 = m_e using the **most recent op's i_0**, so TailSum_e (after op i) = i.)

For each op `i`, `ΔΦ' = 2c_1 · 1 + 2c_1 · 1 = 4c_1`. For each op `i`, `c_i = c_1 (log(i−1) + 0 + (i−1)) = c_1 (log(i−1) + i − 1)`.

`ĉ_i = c_i + ΔΦ' = c_1 (log(i−1) + i − 1) + 4c_1 = O(i) per op`.

⇒ `Σ ĉ_i = O(n²)`, consistent with the actual `Σ c_i = c_1 · n(n−1)/2 + Σ log(i) = O(n²)`. **The two-term Φ does bound the total cost, but the per-op amortised bound is `O(i)`, not `O(\log m_e)`**.

**Conclusion (corrected from round 1):** the sorted-array reference structure has per-op amortised insert `Θ(m_e)` and total `Θ(M²)` under a "persistent head-insert" adversarial workload. The `O(\log m_e)` per-op amortised bound only holds under a balanced-workload assumption (W1 build-once, inserts roughly random / end-appended). Implementations worried about adversarial workloads SHOULD switch to a BBST (R-B) (§7.2.5).

#### 7.2.5 Why sorted-array remains best on a typical markdown workload

Real markdown / scheduling workloads (W2: `m_e` typically < 100; inserts emitted by the parser in source order) do not constitute a worst-case head-insert sequence; in the typical case `i_0` is close to `m_e` (i.e. append-tail style), `(m_e − i_0)` is `O(1)`, and per-op amortised `O(\log m_e)` holds. The **normative text** in §7.2.7 already specifies that an implementation SHOULD switch to a BBST when `m_e > 1000`; this RFC does not provide a sorted-array variant with worst-case `O(\log m_e)` per-op in v1 (see §14.x).

#### 7.2.5 Variant for a BBST implementation

If the implementation switches to a sorted balanced tree (e.g. red-black tree, AVL, or the language's BTreeMap):

- `c_i ≤ c_1 (log m_e + k log m_e)` (k tree-deletes each `O(log m_e)`, one tree-insert `O(log m_e)`).
- `ΔΦ = α(1 - k)`.
- Choosing `α := 2c_1 \log M_{\max}`, `hat c_i ≤ c_1 \log m_e + c_1 k \log m_e - 2c_1 k \log M_{\max} + \alpha`.
- When `m_e ≤ M_max`, `c_1 k \log m_e ≤ c_1 k \log M_{\max} ≤ \tfrac{1}{2} \cdot 2c_1 k \log M_{\max}`, leaving `≤ c_1 \log m_e + \alpha = O(\log m_e + \log M_{\max})`.

#### 7.2.6 Robustness of the potential-method argument (anticipating reviewer pushback)

> **Possible critique:** "Your two-term Φ does not achieve `O(\log m_e)` per-op in the head-insert worst case; therefore Lemma 7.2.A is wrong."
>
> **Reply:** corrected in §7.2.3–§7.2.4. The original Lemma 7.2.A (single-term Φ) only held under a balanced workload; the head-insert adversarial sequence indeed degrades sorted-array to `Θ(m_e)` per-op and `Θ(M²)` total. The corrected Lemma 7.2.A* makes this explicit as the input-dependent term `W_n` and defers the worst case to the BBST switch in §7.2.7.

> **Possible critique:** "k can be arbitrarily large (one insert may merge 100 entries); how does the amortised bound hold?"
>
> **Reply:** for the merge cost `c_1 · k` — each of those 100 entries had previously deposited `α` credit at its own insert; total release = `100 · α`, which pays this insert's `100 · c_1`, leaving `c_1 \log m_e + a single c_1` as the op's amortised cost. The merge component is always `O(\log m_e)` per-op amortised; only the end-shift component has worst-case degradation.

> **Possible critique:** "Once v2 adds a remove operation, this bound breaks immediately."
>
> **Reply:** correct. Remove can increase `M` and breaks the monotonicity of "Φ never goes back". v2 will need a redesigned Φ (e.g. `Φ := M log M` or a binary-counter style); this is open future work in §14.1.

### 7.2.7 Sorted array vs. BBST: rationale for the choice

Implementations may pick one of two underlying structures:

- **(R-A) Sorted Array**: amortised merge cost `O(\log m_e + k)` (per Lemma 7.2.A*); end-shift cost worst-case `O(m_e)` per-op, worst sequence `Θ(M²)` total (head-insert adversarial); under a typical markdown / scheduling workload (W2), end-shift is `O(1)` per-op. Strengths: cache-friendly, native to language stdlibs (Ruby `Array#bsearch_index`, Swift `Array`).
- **(R-B) Sorted Tree / BTreeMap**: worst-case `O(\log m_e + k)` for **all** ops including head-insert. Strength: robust to adversarial workloads; weaknesses: larger constant factor, more cross-language implementation variation.

The RFC requires only (i) correctness, (ii) amortised merge cost `O(\log m_e + k)` (both R-A and R-B satisfy), and (iii) end-shift cost either input-dependent (R-A) or worst-case `O(\log m_e)` (R-B). Implementations SHOULD default to (R-A); for scenarios where `m_e` is expected to exceed ~1000 or where the workload may include adversarial head-insert, implementations SHOULD switch to (R-B).

### 7.3 Query bounds + output sensitivity

| Query | Worst-case |
|---|---|
| `subscript [i]` (lazy already built) | `O(\log\|\text{segments}\| + r)`, `r = \|active(i)\|` |
| `subscript [i]` (lazy first-time build) | `O(M \log M + L \cdot r̄)`, where `r̄` = average active-set size |
| `getRange(of: e)` | `O(\|R(e)\|)` copy / `O(1)` view |
| `transitions(over:)` | `O(\log M + \|\text{result}\|)` |

> **Output-sensitive lower bound:** for stabbing queries, in the pointer-machine model any query returning `r` results must take `Ω(r)` time (the output itself requires `r` writes). Our `subscript [i]` meets this lower bound. For range queries on transitions, similarly `Ω(\|result\|)`. See Afshani, Arge & Larsen for rectangle-stabbing lower bounds (1D is a special case).

### 7.4 Lower bounds (Ω(m) when m intervals stab one point)

If the same coordinate `i` is covered by intervals from `m` distinct elements, then the size of `r[i].objs` is `m`; returning this result requires at least `Ω(m)` time. Our `O(log |segments| + m)` is optimal.

For `insert`, the lower bound is `Ω(\log m_e)` for large `m_e` (comparison-based search lower bound); our `O(\log m_e + k)` matches the lower bound `Ω(log m_e + k)` in an amortised sense (each of `k` deletions requires `\Omega(1)`).

### 7.5 Variant (e) trade-off

| Quantity | (a) main structure | (e) materialised |
|---|---|---|
| Space | `O(M)` | `O(L · r̄)` |
| Build | `O(M log M)` lazy | `O(L + M)` eager |
| Point query | `O(log M + r)` | `O(1)` |
| Range / transitions | `O(log M + result)` | `O(log M + result)` (still via the events index) |
| Suitable for | Sparse / random queries | Dense queries (markdown) |

**Normative:** implementations MAY heuristically choose (a) or (e) after the build completes; the externally observable API behavior MUST be identical.

### 7.6 Removal complexity (v2 normative)

| Op | Worst-case |
|---|---|
| `remove(e, start, end)` (§6.6) | `O(log m_e + k + E)` where `k` is the number of overlapping entries swept and `E` is the cost of the eager-prune `ord` renumber (only when the element is fully excised) |
| `remove(e)` (§6.7) | `O(E)` for the linear scan + renumber loop |
| `removeAll() / clear()` (§6.8) | `O(1)` (assigning empty maps; per-entry GC is amortised) |
| `removeRanges(start, end)` (§6.9) | `O((Σ_e log m_e) + Σ_e k_e + E)` where the `O(E)` term is the single-pass `insertion_order` rebuild (avoids the `O(E²)` naive `delete_at` per element) |

**Eager-prune amortisation argument (informative).** The `O(E)` `ord` renumber cost in `remove(e, start, end)` (when the element is fully excised) is amortised away by the dominant lifecycle: an element that lives through `n` inserts before being fully removed pays `O(n log m_e)` for inserts and `O(E)` once on excision; per-op `O(E / n)` ≤ `O(1)` when `n ≥ E`. For workloads with rare full-removes (the typical case — see §1.4 W1), the `O(E)` renumber is amortised away.

**Bound is normative; implementations MUST NOT exceed.** Implementations MUST NOT use a naive `O(m_e²)` per-element sweep; the bsearch + linear sweep is mandatory. Implementations SHOULD NOT trigger `O(E²)` renumbering by calling `remove(e)` in an `O(E)` loop when `removeRanges` or `clear` would suffice; the §6.9 single-pass rebuild and §6.8 atomic clear are provided exactly to avoid this.

### 7.7 Set-operations complexity (v2 normative)

| Op | Worst-case |
|---|---|
| `union(other)` (§6.10) | `O(M_self + M_other + E_self + E_other)` |
| `intersect(other)` (§6.11) | `O(M_self + M_other)` |
| `difference(other)` (§6.12) | `O(M_self + M_other)` |
| `symmetric_difference(other)` (§6.13) | `O(M_self + M_other)` |

**Two-pointer linear-merge MUST be used (normative; not negotiable).** All four set ops are specified to use a two-pointer sweep over `(L_self, L_other)` per element. Implementations MUST NOT use a nested-loop `O(m_e_self · m_e_other)` per-element strategy; that produces a total `O(M_self · M_other)` bound, which on workloads of `M_self = M_other = 10000` is 100× slower than the spec'd `O(M_self + M_other) = O(20000)`. **The professor MUST verify each language's `merge_disjoint_lists`, `intersect_disjoint_lists`, and `subtract_disjoint_lists` implementations are linear in the sum of input sizes.** Phase 3 per-language test suites SHOULD include a benchmark probe: `M_self = M_other = 1000` MUST complete `union` in `< 10 ms` on a typical developer laptop (informative bound; the load-bearing claim is the asymptotic `O(M_self + M_other)`).

**Per-element pairwise dispatch (normative).** All four set ops iterate over `keys(self) ∪ keys(other)` (for `union`, `symmetric_difference`) or `keys(self) ∩ keys(other)` (for `intersect`) or `keys(self)` (for `difference`); per element, only one two-pointer sweep is invoked. The `keys` iteration order MUST match the per-op insertion-order rule (§4.10 / per-op pseudocode in §6.10–§6.13).

**Mutating vs. non-mutating cost.** The non-mutating forms (returning a new `Rangeable`) and the mutating forms (modifying `self` in place) have the **same** asymptotic cost per the table above. The mutating form's atomicity (copy-then-swap pattern, §6.10) adds a constant-factor overhead but does not change the asymptotic class.

**Eager-prune cost amortisation in set ops.** `intersect`, `difference`, and `symmetric_difference` may prune elements (drop those with empty result `R(e)`); the dense `ord` renumber is `O(E_result)` and absorbed into the `O(M_self + M_other)` total since `E_result ≤ E_self + E_other ≤ M_self + M_other` (using (I1.4): each surviving element has `m_e ≥ 1`).

---

## §8  Comparison with Alternatives

The table below is a comprehensive comparison of v1 candidate structures; the **rationale for choosing (a)** appears in the last column.

| Structure | Insert | Point stab | Range / transitions | Space | Native idempotent / merge | Cross-language deterministic | Reason rejected |
|---|---|---|---|---|---|---|---|
| **(a) Per-element sorted disjoint list + lazy event index** | `O(log m_e + k)` amort. | `O(log M + r)` | `O(log M + result)` | `O(M)` | **Yes** | **Yes** | (**Selected**) |
| (b) Augmented BBST / Interval tree (CLRS 14.3) | `O(log N)` | `O(log N + r)` | `O(log N + result)` | `O(N)` (N = insert count, undeduped) | No: each stab requires post-hoc group-by-element + merge, not cheap | Yes | Does not naturally fit element-dimension merge semantics |
| (c) Segment tree + coordinate compression | `O(log U)` build + `O(log U)` per update | `O(log U + r)` | `O(log U + result)` | `O(U log U)` | No | Yes (offline) | Requires offline coordinate compression; unsuitable for streaming insert |
| (d) Skip list (per-element) | expected `O(log m_e)` | expected `O(log M + r)` | expected `O(log M + result)` | `O(M)` | Yes | **No**: probabilistic level promotion makes cross-language RNG hard to align | Cross-language byte-identical fails |
| (e) Per-coordinate materialised array | `O(L)` rebuild | `O(1)` | `O(result)` via events | `O(L · r̄)` | Yes | Yes | **A transparent variant of (a), not a replacement** |
| (f) Roaring bitmap (per element) | amort. `O(\|range\|)` for the run container | `O(log M + r)` | `O(log M + result)` | `O(M)` | Partial (natural for a single bitmap union; nothing built-in across elements) | Yes | Designed for 32-bit dense integers, over-engineered; does not directly support an element dimension |
| (g) Boost.ICL `interval_map<K, V>` | `O(log N)` | `O(log N)` | yes | `O(N)` | No: aggregate-on-overlap accumulates values, orthogonal to "element membership" semantics | Yes | Value accumulation ≠ element-set membership |
| (h) Boost.ICL `interval_set<K>` per element + outer `Map<Element, interval_set>` | `O(log m_e + k)` | `O(M)` brute | `O(M)` brute | `O(M)` | Yes | Yes | **Isomorphic** to (a), but lacks the lazy event index → point stab degrades |

**Three keys to (a)'s win:**

1. **Per-element merge semantics fit naturally**: `R(e)` is "the merged intervals of e"; `getRange` returns it directly. Both BBST and segment tree need post-hoc grouping.
2. **Deterministic + cross-language portable**: every tie-breaking rule depends only on integer insertion order + lexicographic comparison; no hash buckets, no RNG.
3. **Lazy index fits the build-once-then-query pattern perfectly**: a continuous batch of inserts during build does not rebuild the index; the query phase pays a one-shot `O(M log M)`.

> **Why the augmented interval tree is rejected, in detail:** the interval tree of CLRS §14.3 treats intervals as **first-class entities** (each insert is an independent interval node) and supports a single-overlap query in `O(log N)`. But finding **all** overlapping intervals takes `O(min(N, k log N))`, and the result still needs element-level grouping + merge before we can answer `r[i].objs`. For `getRange(of: e)` it would walk the entire tree. Our (a) structure makes the element dimension the outer Map, making the N → M reduction (N = raw inserts, M = total merged intervals) the baseline; this reduction is often 5–10× on the markdown workload.

> **Why the segment tree is rejected, in detail:** a segment tree first requires coordinate compression to bound U (the coordinate range), which requires all endpoints to be known up front — offline only. In streaming (online) insert mode it requires a dynamic or implicit segment tree, with high implementation complexity and difficult cross-language reproducibility.

> **Why the skip list is rejected, in detail:** although deterministic skip lists exist (Munro & Sedgewick), (i) mainstream skip-list textbooks all use randomisation, (ii) the promotion rules of deterministic skip lists differ significantly across language implementations, (iii) a red-black tree is equivalently available in multiple language stdlibs, so there is no need to adopt a skip list.

> **Why Roaring bitmap is rejected, in detail:** Roaring is the state-of-the-art compression for dense 32-bit integer sets. It splits `[0, 2^32)` into 2^16 "chunks", each chunk using one of array / bitset / run container. This is perfect for "the integer set of a single element", but our problem is not compressing an integer set; it is an element-set indexed by integer. Wrapping Roaring in an outer `Map<Element, RoaringBitmap>` is a viable alternative (row H), but (i) our `m_e` is typically very small (< 100), so Roaring's chunk overhead is net negative; (ii) we still need a separate event index to support point stab.

> **Why Boost.ICL is rejected, in detail:** Boost.ICL `interval_map<K, V>` performs aggregate-on-overlap at overlapping ranges: inserting `[2,5] -> Strong` then `[3,7] -> Strong` with `+=` would have ICL record `[2,2] -> Strong, [3,5] -> Strong+Strong, [6,7] -> Strong` (Strong accumulated twice). For boolean "is the element active?" logic, we do not want value accumulation; we want "multiple inserts of the same element idempotently union". `split_interval_map` is even less suitable, splitting everywhere. Boost.ICL `interval_set<K>` is on the right track, but it requires one instance per element with an outer Map around it, and ICL provides no native query for "active set per coordinate" → we are back to (a).

#### 8.1 In-depth comparison with Boost.ICL `interval_map<int, set<Element>>`

A reader might immediately object: "Why not just use Boost.ICL `interval_map<int, std::set<Element>>`? The value is a set, and aggregate-on-overlap is replaced by set union (`set::insert`) instead of the default `+=`." This scheme is semantically correct, but is **still unsuitable as the v1 reference design of this RFC** for the following reasons:

1. **Poor cross-language portability**: Boost.ICL is a C++ template-only library; Ruby and Swift have no direct equivalents. A Ruby/Swift port would amount to reimplementing all of ICL — far more effort than the sorted-disjoint-list design of §5.1.
2. **Each coord segment's value is a sorted set with no first-insert order**: `std::set` is sorted by `std::less<Element>`, which requires Element to provide `<`. Elements do not necessarily have a natural total order (there is no "Strong < Italic" between Strong and Italic); forcing a hash or type-name ordering violates §4.5's deterministic insertion-order requirement.
3. **Element-dimension queries degrade**: `getRange(of: Strong())` on ICL would walk the entire `interval_map`, filter each segment's set for Strong, and union the results; `O(N · m_set)` rather than `O(|R(Strong)|)`.
4. **Space overhead**: ICL `interval_map` stores a copy of each distinct value-set; for markdown, the redundancy of "STRONG always lives in a set with STRONG" appears repeatedly. The (a) structure of this RFC stores "the interval set of e" and "the active set at i" **dually** (one in `intervals[e]`, the other in `event_index.segments[s].active`), lazily synchronised — space-optimal.

#### 8.2 Comparison with the sweep-line style `std::map<int, ...>`

Another alternative is to directly use `Map<int, ChangeSet>` (key: coordinate; value: `(opens, closes)` pair) — the so-called sweep-line event map. This scheme works, but:

- It is **equivalent to §5.2's `event_index`**; the difference is only "`event_index` as a lazy derived view, `intervals[e]` as the source of truth" vs. "the event map as the source of truth and reconstructing `R(e)` from it in `O(N)`". The latter degrades `getRange` and makes `insert`'s idempotence check harder (you must walk events to find every open/close of the element).
- This RFC therefore pins the source of truth at the element dimension (per-element list), with the event index derived (§5.2).

#### 8.3 Why no functional / persistent structure (Okasaki style)

Both Okasaki's *Functional Data Structures* and Driscoll et al. 1986's path copying could make this structure persistent. But v1 pins a single-mutating-instance model; persistent variants are deferred to §14.2. Reasons:

1. Ruby's GC interacts poorly with the sharing of functional structures (`dup` of each frozen Array in Ruby is not free).
2. Swift's COW already provides a half-free persistence illusion (the caller writes `var copy = r` and gets it); the marginal benefit of an additional functional API is limited.
3. This RFC's lazy-event-index pattern is friendly to immutable "freeze after build"; if a caller wants an immutable view, freezing the entire instance after the build suffices.

---

## §9  Edge Cases (normative table)

| # | Case | Specification |
|---|---|---|
| 1 | Empty container point query | `r[i].objs == []` for any `i ∈ ℤ`. |
| 2 | Single-point interval (`start == end`) | (D2): `r[s].objs` contains `e`; `r[s−1]` and `r[s+1]` do not. |
| 3 | `start > end` | (D1): MUST raise `InvalidIntervalError`; state unchanged. |
| 4 | Idempotent insert with `start == end` | The second insert has no side effect (version unchanged, insertion_order unchanged). |
| 5 | Negative coordinates | (C2): MUST be supported; after `insert(e, -10, -5)`, `r[-7].objs` contains `e`. |
| 6 | Same-element overlapping insert | `[2,5] ∪ [3,7] = [2,7]` (§4.3 does not require adjacency; overlap already suffices to merge). |
| 7 | Same-element integer-adjacent insert | (§4.3): `[2,4] ∪ [5,7] = [2,7]`. |
| 8 | Same-element nested insert | `[2,10] ∪ [4,6] = [2,10]`; the inner entry is absorbed. |
| 9 | Same-element non-adjacent insert | `[2,4] ∪ [6,7] = [(2,4), (6,7)]`; two entries are kept (5 between 4 and 6 is a gap). |
| 10 | Different elements active at the same coordinate | `r[i].objs` is ordered by `ord` ascending; different elements do not affect each other's `R(·)`. |
| 11 | Equal-by-equality elements merge | Two `Link("a")` instances merge; `Link("a")` and `Link("b")` do not. |
| 12 | `Int.max` close-event coord | (C4): MUST be represented as `None` (the `+∞` sentinel of `Optional<Int>`); coord ordering: `None > all finite Some(_)`; `coord.succ()` rules are in §4.7 (C4). |
| 13 | Many same-coordinate open events | All ordered by `ord` ascending in `transitions`. |
| 14 | Many same-coordinate close events | All ordered by `ord` **descending** in `transitions` (LIFO). |
| 15 | Caller mutates an element after insert | Violates §4.6 (M1); undefined behavior. Implementations MAY defensively freeze, but it is not MUST. |
| 16 | Hash collision but `==` false | No merge; different equivalence classes; the outer Map handles this via hash bucket + `==` comparison. |
| 17 | `transitions(over: lo, hi)` with `lo > hi` | MUST raise `InvalidIntervalError`. |
| 18 | `getRange` called on a never-inserted element | Returns `[]` (not nil; not a throw). |
| 19 | Query after zero mutations | Lazy build triggers; `event_index` is freshly empty. |
| 20 | Empty-string / nil element | The element must conform to the language's Hashable; in Ruby `nil.hash` is legal, but `nil` is typically a user bug; implementations MAY reject it. |
| 21 | `start == end + 1` in a transitions query | The legal precondition of `transitions(over: lo..hi)` is `lo ≤ hi`; `lo == hi + 1` is treated as `lo > hi` (D1) and MUST raise `InvalidIntervalError` (see case 17, Test #22). |
| 22 | `Int.min` as `lo` in an insert | (C5): MUST be supported. After `insert(e, Int.min, k)`, for any `i ∈ [Int.min, k]`, `e ∈ r[i].objs` MUST hold. Internal algorithms MUST NOT compute `Int.min − 1` (underflow). |
| 23 | Full-axis interval `[Int.min, Int.max]` | (C4 + C5): legal. close event coord = `None`; for any finite `i`, `r[i].objs` contains `e`. |
| 24 | No-op `remove(e, start, end)` (no overlap) | (§4.10 N3, §6.6 step 4): MUST NOT bump `version`; MUST NOT invalidate `event_index`. |
| 25 | `remove(e)` on never-inserted element | (§4.10 N3, §6.7 step 1): MUST NOT bump `version`; MUST NOT invalidate `event_index`. |
| 26 | `removeAll() / clear()` on empty container | (§4.10 N3, §6.8 step 1): MUST NOT bump `version`; MUST NOT invalidate `event_index`. |
| 27 | `removeRanges(start, end)` with no overlap on any element | (§4.10 N3, §6.9 step 3): MUST NOT bump `version`; MUST NOT invalidate `event_index`. Verified by checking `any_change == false`. |
| 28 | `remove(e, start, end)` triggers eager prune | (§4.10 N1, §6.6 step 7): `e` MUST be excised from `intervals`, `insertion_order`, `ord`; subsequent `count` decreases by 1; subsequent elements' `ord` shift down by 1. |
| 29 | `difference(other)` fully consumes element | (§4.10 N1, §6.12): if `R_self(e) ⊆ R_other(e)`, then `e ∉ keys(result)`; `count_result < count_self`. |
| 30 | `intersect(other)` with empty `Rangeable` | (§6.11): result is empty (`count == 0`); both `keys(other) == ∅` ⇒ no key in `keys(self) ∩ keys(other)`. |
| 31 | `union(empty, empty)` | (§6.10): result is empty (`count == 0`); fresh `version = 0`. |
| 32 | `union(self, self)` | (§6.10 idempotence dual): mutating form MUST NOT bump `version`; non-mutating form returns a structurally-equal new `Rangeable`. |
| 33 | `removeRanges` raises mid-element (atomicity) | (§6.9): the precondition check (`start > end`) raises BEFORE any mutation; container state MUST be unchanged on raise. (Mid-loop allocator failures are out-of-scope; copy-then-swap pattern recommended.) |
| 34 | Symmetric difference: two-side residuals adjacent | (§6.13 worked example): `R_self = [(0, 5)]`, `R_other = [(6, 10)]` ⇒ `a = [(0, 5)]`, `b = [(6, 10)]`, `a.last.hi + 1 == b.first.lo` ⇒ `merge_disjoint_lists` MUST collapse to `[(0, 10)]`. (Sorted concat is INSUFFICIENT.) |
| 35 | `insert` after full `remove(e)` reassigns `ord` | (§4.10 deliberate side effect, §4.5 first-insert ordinal): `r.insert(e, …); r.remove(e); r.insert(e, …)` ⇒ second insert is a first-insert; `ord(e) == count` (tail of `insertion_order`). |
| 36 | Set-op `keyFn` mismatch (JS-specific) | Languages where `keyFn` is configurable (JS) MUST raise an error when `self._keyFn !== other._keyFn`; MUST NOT silently coerce. |
| 37 | `Int.max`-bounded `intMaxSentinel` propagation across set-op | Implementations whose data structure uses an `intMaxSentinel` (Go, JS, optionally Kotlin) MUST propagate the sentinel into result containers from non-mutating set ops; transitions on the result MUST produce identical close-coord behavior at `Int.max`. **Sentinel-conflict resolution (normative, v2.0 patch):** when `self.intMaxSentinel` and `other.intMaxSentinel` are both set, the result container MUST inherit `self.intMaxSentinel` (the **left-wins** rule — set ops are not algebraically commutative on configuration metadata, so the receiver/lhs is canonical). When the two values disagree (`self.intMaxSentinel != other.intMaxSentinel`), implementations SHOULD emit a warning (e.g. log entry, debug-only `assert`) but MUST NOT throw — the operation is well-defined under the left-wins rule, and the only observable downstream consequence is at the `transitions(over: …)` boundary on a coordinate equal to one of the two sentinels. When `other.intMaxSentinel` is set but `self.intMaxSentinel` is unset, the result inherits `nil` / unset (left-wins, no auto-promotion). |

---

## §10  Test Contract (normative)

> **This section is the normative v1 test contract.** The Ruby and Swift implementations MUST pass all 20 tests, and each test's output MUST be byte-identical (under a reasonable tuple/array serialisation convention). Each test is expressed in `Given / When / Then` form.

### Test #1 — Empty
**Given:** `r := Rangeable<Strong>()`
**When:** `r[0].objs` and `Strong().getRange(from: r)`
**Then:** both are `[]`.

### Test #2 — Single insert
**Given:** `r := Rangeable<Strong>()`; `r.insert(Strong(), start: 2, end: 5)`
**When:** `r[2].objs`, `r[5].objs`, `r[6].objs`, `r[1].objs`
**Then:** respectively `[Strong]`, `[Strong]`, `[]`, `[]`.

### Test #3 — Inclusive end
**Given:** `r := Rangeable<Strong>()`; `r.insert(Strong(), 3, 8)`
**When:** `r[8].objs`, `r[9].objs`
**Then:** respectively `[Strong]`, `[]`.

### Test #4 — Single-point
**Given:** `r := Rangeable<Strong>()`; `r.insert(Strong(), 4, 4)`
**When:** `r[3].objs`, `r[4].objs`, `r[5].objs`
**Then:** respectively `[]`, `[Strong]`, `[]`.

### Test #5 — Same-element overlap merge
**Given:** `r.insert(Strong(), 2, 5); r.insert(Strong(), 3, 7)`
**When:** `Strong().getRange(from: r)`
**Then:** `[(2, 7)]`.

### Test #6 — Same-element adjacency merge
**Given:** `r.insert(Strong(), 2, 4); r.insert(Strong(), 5, 7)`
**When:** `Strong().getRange(from: r)`
**Then:** `[(2, 7)]`.
**Justification:** §4.3 merge on integer adjacency (4+1 == 5).

### Test #7 — Same-element non-adjacent disjoint
**Given:** `r.insert(Strong(), 2, 4); r.insert(Strong(), 6, 7)`
**When:** `Strong().getRange(from: r)`
**Then:** `[(2, 4), (6, 7)]`.
**Justification:** 4+1 = 5 ≠ 6, non-adjacent; two entries are retained.

### Test #8 — Same-element nested
**Given:** `r.insert(Strong(), 2, 10); r.insert(Strong(), 4, 6)`
**When:** `Strong().getRange(from: r)`
**Then:** `[(2, 10)]`.

### Test #9 — Idempotent insert
**Given:** `r.insert(Strong(), 2, 5); r.insert(Strong(), 2, 5)`
**When:** `Strong().getRange(from: r)` and querying `r.version` (if the API exposes it)
**Then:** `[(2, 5)]`; the version MUST NOT increase on the second insert.

### Test #10 — Different elements coexist
**Given:** `r.insert(Strong(), 2, 5); r.insert(Italic(), 3, 7)`
**When:** `r[3].objs`, `r[6].objs`, `Strong().getRange(from: r)`, `Italic().getRange(from: r)`
**Then:** respectively `[Strong, Italic]`, `[Italic]`, `[(2, 5)]`, `[(3, 7)]`.

### Test #11 — Equal-by-equality elements merge
**Given:**
```
r.insert(Link("a"), 2, 5)
r.insert(Link("a"), 4, 8)
r.insert(Link("b"), 6, 9)
```
**When:** `Link("a").getRange(from: r)`, `Link("b").getRange(from: r)`
**Then:** respectively `[(2, 8)]`, `[(6, 9)]`.
**Justification:** the two `Link("a")` instances merge via `==` / `eql?` equivalence; `Link("b")` belongs to a different equivalence class.

### Test #12 — First-insert order at point
**Given:** `r.insert(Strong(), 1, 10); r.insert(Italic(), 1, 10); r.insert(Code(), 1, 10)`
**When:** `r[5].objs`
**Then:** `[Strong, Italic, Code]`.
**Justification:** insertion order is S=1, I=2, C=3.

### Test #13 — Order preserved through merge
**Given:**
```
r.insert(Strong(), 1, 5)
r.insert(Italic(), 3, 7)
r.insert(Strong(), 4, 8)
```
**When:** `r[6].objs`
**Then:** `[Strong, Italic]`.

**Justification (formal, §4.5):** the insertion-order index is fixed once at the element's **first** insert. Event trace:
- step 1: `insert(Strong, 1, 5)` ⇒ Strong is a new element; assign `ord(Strong) = 1`; `R(Strong) = [(1,5)]`.
- step 2: `insert(Italic, 3, 7)` ⇒ Italic is a new element; assign `ord(Italic) = 2`; `R(Italic) = [(3,7)]`.
- step 3: `insert(Strong, 4, 8)` ⇒ Strong already exists; `ord` is not updated (still 1); `R(Strong) = canonicalize([(1,5), (4,8)]) = [(1,8)]` (since 5+1=6 > 4, `(1,5)` and `(4,8)` **overlap** — 4 ≤ 5 — and merge into `(1,8)`).

After the merge: `R(Strong) = [(1,8)]`, `R(Italic) = [(3,7)]`, `ord = {Strong: 1, Italic: 2}`.

At i = 6: active set = `{e | ∃[l,h] ∈ R(e), l ≤ 6 ≤ h}` = `{Strong, Italic}` (both cover 6). Per §4.5 "ascending ord" ⇒ `[Strong, Italic]`.

> **The key semantics this case pins down:** subsequent merging inserts of the same element (even ones that extend the interval to a previously uncovered position) **do not update ord**. Although Strong becomes active at position 6 only after step 3, its first-insert-order index remains 1 and always precedes Italic's `ord = 2`. This guarantees the "**outer-first by chronology**" intuition for active-set ordering: an element declared earlier is always treated as the outer container.
>
> **Alternative considered:** if `ord` were instead "the ord of the most recent insert" (recency), this case would be `[Italic, Strong]` (Italic ord=2 < Strong recency=3). The RFC rejects this scheme because: (i) it breaks the "outer style is always outer" markdown intuition; (ii) recency is poorly defined under idempotent inserts; (iii) if a merging insert triggered an ord update, the §3.2 idempotence contract would be hard to maintain.

### Test #14 — Transitions over a range
**Given:** the state of Test #10: `insert(Strong(), 2, 5); insert(Italic(), 3, 7)`
**When:** `r.transitions(over: 0..10)`
**Then:** in §4.5 ordering:
```
[
  (coord: 2, kind: open,  element: Strong),
  (coord: 3, kind: open,  element: Italic),
  (coord: 6, kind: close, element: Strong),  // Strong's close coord = hi+1 = 6
  (coord: 8, kind: close, element: Italic),  // Italic's close coord = 8
]
```

### Test #15 — Transitions same-start
**Given:** `r.insert(Strong(), 3, 5); r.insert(Italic(), 3, 7)`
**When:** `r.transitions(over: 0..10)`
**Then:**
```
[
  (3, open,  Strong),   // ord=1 first
  (3, open,  Italic),   // ord=2 second
  (6, close, Strong),
  (8, close, Italic),
]
```

### Test #16 — Transitions same-end (LIFO)
**Given:** `r.insert(Strong(), 3, 5); r.insert(Italic(), 3, 5)`
**When:** `r.transitions(over: 0..10)`
**Then:**
```
[
  (3, open,  Strong),    // ord=1 first
  (3, open,  Italic),    // ord=2 second
  (6, close, Italic),    // close: ord descending, ord=2 first
  (6, close, Strong),    // close: ord=1 second
]
```

### Test #17 — `start > end` raises
**Given:** `r := Rangeable<Strong>()`
**When:** `r.insert(Strong(), 5, 2)`
**Then:** raises `InvalidIntervalError`; the container MUST remain empty.

### Test #18 — Negative start
**Given:** `r.insert(Strong(), -2, 3)`
**When:** `r[-1].objs`, `r[0].objs`, `r[3].objs`, `r[4].objs`
**Then:** respectively `[Strong]`, `[Strong]`, `[Strong]`, `[]`.

### Test #19 — Insert/read interleave (rebuild correctness)
**Given:**
```
r.insert(Strong(), 1, 3)
read1 := r[2].objs        // triggers lazy build
r.insert(Strong(), 5, 7)  // must invalidate event_index
read2 := r[6].objs        // must rebuild
```
**When:** `read1`, `read2`, `Strong().getRange(from: r)`
**Then:** `[Strong]`, `[Strong]`, `[(1, 3), (5, 7)]`.
**Justification:** event_index invalidation is correct.

### Test #20 — Property test (stress)
**Given:**
```
RNG seed := fixed (e.g. 42)
elements := [Strong(), Italic(), Code(), Link("x"), Link("y")]
ops := [generate 1000 random (start, end, element) with start ≤ end, |coord| ≤ 200]
build r by performing each insert
```
**When:** for each i ∈ [-200, 200]: `r[i].objs`
**Then:** equals `brute_force_active_set(i)`, defined as:
```
brute_force_active_set(i) :=
  let touched_elements := [e in insertion_order order if any inserted (e, lo, hi) has lo ≤ i ≤ hi]
  return touched_elements (with duplicates removed, preserving first occurrence)
```

**Note:** this test MUST use a fixed RNG seed for cross-language reproducibility; Ruby's `Random.new(42)` and Swift's `SystemRandomNumberGenerator` are not consistent, so implementations MAY share a pre-generated list of `(start, end, element_index)` triples between Ruby and Swift (e.g. written to JSON ahead of time).

### 10.A  Boundary / Idempotency Tests

> Tests #21–#23 below are elevated to **normative** (round 2 revision); #24–#27 are informative recommended hardening tests.

### Test #21 — Idempotent insert does NOT bump version (normative)
**Given:** `r.insert(Strong(), 2, 5); v1 := r.version`
**When:** `r.insert(Strong(), 2, 5); v2 := r.version`
**Then:** `v1 == v2`.
**Justification:** §3.2 idempotence contract + §6.1 cleaner-variant containment fast-path (Lemma 6.5.B).

### Test #21.A — Idempotent insert with strict containment (normative)
**Given:** `r.insert(Strong(), 2, 10); v1 := r.version`
**When:** `r.insert(Strong(), 4, 6); v2 := r.version`
**Then:** `Strong().getRange(from: r) == [(2, 10)]`; `v1 == v2`.
**Justification:** the containment fast-path (`to_merge[0].lo == 2 ≤ 4`, `6 ≤ to_merge[0].hi == 10`) ⇒ no mutation, no version bump.

### Test #22 — `transitions(over: lo, hi)` with `lo > hi` raises (normative)
**Given:** the state of Test #2
**When:** `r.transitions(over: 5..2)` (or the language's reverse-range expression)
**Then:** raises `InvalidIntervalError`.

### Test #23 — `Int.min` as lo in insert (normative)
**Given:** `r := Rangeable<Strong>()`; `r.insert(Strong(), Int.min, Int.min + 5)`
**When:** `r[Int.min].objs`, `r[Int.min + 5].objs`, `r[Int.min + 6].objs`, `Strong().getRange(from: r)`
**Then:**
- `r[Int.min].objs == [Strong]`
- `r[Int.min + 5].objs == [Strong]`
- `r[Int.min + 6].objs == []`
- `Strong().getRange(from: r) == [(Int.min, Int.min + 5)]`
**Justification:** §4.7 (C5) `Int.min` endpoint underflow-free invariant; §6.1 internal bsearch predicate `iv.hi + 1 ≥ lo` MUST NOT compute `lo − 1`.

### Test #23.A — `Int.max` as hi in insert (normative)
**Given:** `r := Rangeable<Strong>()`; `r.insert(Strong(), 100, Int.max)`
**When:** `r.transitions(over: 50..(Int.max))`
**Then:** transitions contains
- `(coord: Some(100), open, Strong)`
- `(coord: None, close, Strong)` (the close-event sentinel for `Int.max`)

The order is `[open, close]`.
**Justification:** §4.7 (C4) close-event coord is `Optional<Int>`; `hi == Int.max` ⇒ close coord = `None`; in the total order, `None > any Some(_)`.

### Test #24 — `[i].objs` returns the same list across repeated reads (no mutation in between) (informative)
**Given:** the state of Test #10
**When:** `a := r[3].objs; b := r[3].objs`
**Then:** `a == b` and structurally equal (same elements in the same order).

### Test #25 — Lazy build is invariant under no-op intermediate query (informative)
**Given:** `r.insert(Strong(), 1, 5)`
**When:** `r[2].objs` (triggers build); read `r[2].objs` again
**Then:** both reads return the same value; event_index version is unchanged on the second read.

### Test #26 — Cross-element merge does not pollute insertion order (informative)
**Given:**
```
r.insert(Strong(), 1, 3)
r.insert(Italic(), 2, 4)
r.insert(Strong(), 3, 5)   // merges Strong's intervals to [(1,5)]
```
**When:** `r[2].objs`
**Then:** `[Strong, Italic]` (Strong ord=1 < Italic ord=2; the order is not changed by step 3's merging insert).

### Test #27 — Empty `transitions` outside any interval (informative)
**Given:** `r.insert(Strong(), 10, 20)`
**When:** `r.transitions(over: 0..5)`
**Then:** `[]`.

### Test #28 — Self-referential element (Hashable but mutable struct frozen on insert) (informative)
**Given (Ruby only):**
```ruby
m = Markup.new(...)
r.insert(m, 1, 5)
m.foo = "changed"  // violates §4.6 (M1) — caller bug
```
**When:** `r[2].objs`
**Then:** the implementation SHOULD still return the element copy frozen at insert time; its `foo` is the original value before "changed". Not applicable to Swift (value semantics).

> **Note:** Test #28 is informative; it describes the defensive behavior of §4.6 (M2). Owing to language differences, only the Ruby implementation is normatively required to perform `dup.freeze`; the Swift implementation passes trivially via value semantics.

---

### 10.B  Removal Tests (v2 normative; #21–#43)

> **Note (numbering reset for v2 families):** Tests #21–#28 in §10.A above are renamed `#21A–#28A` in cross-references when ambiguity arises with v2's #21–#80; in this RFC the v1 §10.A tests retain their #21–#28 numbers (covering insert/transitions boundary + idempotency tests) and the v2 tests #21–#80 below cover removal + set ops. Implementations MUST disambiguate by section prefix (`§10.A #21` vs `§10.B #21`) when reporting test results. The cross-language fixture (`cross_language.json` v2) uses fully-qualified test IDs `removal_no_overlap`, `union_disjoint_keys`, etc., to avoid this ambiguity.

#### Test #21 — `remove(e, start, end)` no overlap (no-op, no version bump)
**Given:** `r.insert(Strong(), 10, 20); v0 := r.version`
**When:** `r.remove(Strong(), 0, 5); v1 := r.version`
**Then:** `Strong().getRange(from: r) == [(10, 20)]`; `v0 == v1`; `r.count == 1`.
**Justification:** §6.6 step 4 quick-exit; §4.10 (N3) idempotence.

#### Test #22 — `remove(e, start, end)` exact match consumes one entry
**Given:** `r.insert(Strong(), 10, 20)`
**When:** `r.remove(Strong(), 10, 20)`
**Then:** `r.count == 0`; `r.isEmpty == true`; `Strong()` is excised from `insertion_order`.
**Justification:** §6.6 step 5 left-residual NOT created (`iv.lo == start`), right-residual NOT created (`end == iv.hi`); replacements `== []`; step 7 eager-prunes.

#### Test #23 — `remove(e, start, end)` leaves left residual only
**Given:** `r.insert(Strong(), 0, 10)`
**When:** `r.remove(Strong(), 5, 100)`
**Then:** `Strong().getRange(from: r) == [(0, 4)]`.
**Justification:** §6.6 step 5: `iv.lo (0) < start (5)` ⇒ left residual `(0, 4)`; `end (100) > iv.hi (10)` ⇒ NO right residual.

#### Test #24 — `remove(e, start, end)` leaves right residual only
**Given:** `r.insert(Strong(), 0, 10)`
**When:** `r.remove(Strong(), -100, 5)`
**Then:** `Strong().getRange(from: r) == [(6, 10)]`.
**Justification:** §6.6 step 5: `iv.lo (0) >= start (-100)` ⇒ NO left residual; `end (5) < iv.hi (10)` ⇒ right residual `(6, 10)`.

#### Test #25 — `remove(e, start, end)` splits one entry into two
**Given:** `r.insert(Strong(), 0, 10)`
**When:** `r.remove(Strong(), 3, 6)`
**Then:** `Strong().getRange(from: r) == [(0, 2), (7, 10)]`.
**Justification:** §6.6 step 5 produces both residuals: `(0, 2)` and `(7, 10)`; (I1.2) holds with gap `7 - 2 - 1 = 4`.

#### Test #26 — `remove(e, start, end)` spans multiple entries
**Given:** `r.insert(Strong(), 0, 5); r.insert(Strong(), 10, 15); r.insert(Strong(), 20, 25)`
**When:** `r.remove(Strong(), 3, 22)`
**Then:** `Strong().getRange(from: r) == [(0, 2), (23, 25)]`.
**Justification:** §6.6 step 5 sweep: `(0, 5)` produces left residual `(0, 2)`; `(10, 15)` is fully consumed (no residuals); `(20, 25)` produces right residual `(23, 25)`.

#### Test #27 — `remove(e, start, end)` spans entire `R(e)`, prunes element
**Given:** `r.insert(Strong(), 0, 5); r.insert(Strong(), 10, 15); r.insert(Italic(), 7, 8); v0 := r.version`
**When:** `r.remove(Strong(), -100, 100); v1 := r.version`
**Then:** `r.count == 1`; `keys(r) == [Italic]`; `Italic` has `ord == 1` (renumbered from 2); `v1 == v0 + 1`.
**Justification:** §6.6 step 5 fully consumes both Strong entries; step 7 eager-prunes Strong; Italic's `ord` shifts from 2 to 1.

#### Test #28 — `remove(e, start, end)` no-op MUST NOT bump version
**Given:** `r.insert(Strong(), 10, 20); v0 := r.version`
**When:** `r.remove(Strong(), 30, 40); v1 := r.version`; `r.remove(Italic(), 0, 5); v2 := r.version`
**Then:** `v0 == v1 == v2`.
**Justification:** §6.6 step 4 (no overlap) and §6.6 step 2 (`Italic` not in keys) are both no-ops per §4.10 (N3).

#### Test #29 — `remove(e, start, end)` with `start > end` raises
**Given:** `r.insert(Strong(), 0, 10)`
**When:** `r.remove(Strong(), 7, 3)`
**Then:** raises `InvalidIntervalError`; `r` unchanged (`Strong().getRange(from: r) == [(0, 10)]`).
**Justification:** §6.6 step 1 pre-condition.

#### Test #30 — `remove(e, start, end)` with `start == Int.min` (underflow safe)
**Given:** `r.insert(Strong(), Int.min, Int.min + 100)`
**When:** `r.remove(Strong(), Int.min, Int.min + 50)`
**Then:** `Strong().getRange(from: r) == [(Int.min + 51, Int.min + 100)]`.
**Justification:** §6.6 step 5 left-residual is NOT created (`iv.lo == Int.min == start`), so `start - 1 = Int.min - 1` is NEVER computed. (Underflow safety dual of §6.1 (P5).)

#### Test #31 — `remove(e, start, end)` with `end == Int.max` (overflow safe)
**Given:** `r.insert(Strong(), 0, Int.max)`
**When:** `r.remove(Strong(), 1000, Int.max)`
**Then:** `Strong().getRange(from: r) == [(0, 999)]`.
**Justification:** §6.6 step 5 right-residual NOT created (`iv.hi == Int.max == end`), so `end + 1 = Int.max + 1` is NEVER computed.

#### Test #32 — `remove(e)` excises element
**Given:** `r.insert(Strong(), 0, 5); r.insert(Italic(), 7, 12); r.insert(Code(), 15, 20); v0 := r.version`
**When:** `r.remove(Italic()); v1 := r.version`
**Then:** `r.count == 2`; `keys(r) == [Strong, Code]`; `r.ord(Strong) == 1`; `r.ord(Code) == 2` (renumbered from 3); `v1 == v0 + 1`.
**Justification:** §6.7 step 3: linear scan + renumber.

#### Test #33 — `remove(e)` on never-inserted element MUST NOT bump version
**Given:** `r.insert(Strong(), 0, 5); v0 := r.version`
**When:** `r.remove(Italic()); v1 := r.version`
**Then:** `r.count == 1`; `v0 == v1`.
**Justification:** §6.7 step 1 + §4.10 (N3).

#### Test #34 — `remove(e)` on element with single interval
**Given:** `r.insert(Strong(), 5, 10)`
**When:** `r.remove(Strong())`
**Then:** `r.isEmpty == true`; `r.count == 0`.
**Justification:** §6.7 fully excises.

#### Test #35 — `remove(e)` on element with many intervals
**Given:** `r.insert(Strong(), 0, 5); r.insert(Strong(), 10, 15); r.insert(Strong(), 20, 25)` (three disjoint Strong runs)
**When:** `r.remove(Strong())`
**Then:** `r.isEmpty == true`; `Strong().getRange(from: r) == []`.
**Justification:** §6.7 step 2 deletes the entire `intervals[Strong]` list regardless of size.

#### Test #36 — `removeAll() / clear()` on non-empty container
**Given:** `r.insert(Strong(), 0, 5); r.insert(Italic(), 7, 12); v0 := r.version`
**When:** `r.removeAll(); v1 := r.version`
**Then:** `r.isEmpty == true`; `r.count == 0`; `Strong().getRange(from: r) == []`; `Italic().getRange(from: r) == []`; `v1 == v0 + 1`.
**Justification:** §6.8 step 2 clears all element-keyed structures.

#### Test #37 — `removeAll() / clear()` on empty container MUST NOT bump version
**Given:** `r := Rangeable<Strong>(); v0 := r.version`
**When:** `r.removeAll(); v1 := r.version`
**Then:** `r.isEmpty == true`; `v0 == v1`.
**Justification:** §6.8 step 1 idempotence.

#### Test #38 — Post-clear `r.isEmpty == true`
**Given:** `r.insert(Strong(), 0, 5)`
**When:** `r.removeAll()`
**Then:** `r.isEmpty == true`; subsequent `r[3].objs == []`; `r.transitions(over: 0..10) == []`.
**Justification:** §6.8 + §3.5.1 `isEmpty` contract.

#### Test #39 — Post-clear `r.count == 0`
**Given:** `r.insert(Strong(), 0, 5); r.insert(Italic(), 7, 12)`
**When:** `r.removeAll()`
**Then:** `r.count == 0`; iterating `r.each { |e, ranges| ... }` yields nothing.
**Justification:** §6.8 + §3.5.1 `count` contract.

#### Test #40 — Insert after `clear` assigns `ord = 1`
**Given:** `r.insert(Strong(), 0, 5); r.insert(Italic(), 7, 12); r.removeAll()`
**When:** `r.insert(Code(), 100, 110)`
**Then:** `r.count == 1`; `r.ord(Code) == 1` (NOT 3, despite Strong / Italic being inserted before clear).
**Justification:** §6.8 fully resets `insertion_order` and `ord`; the next insert is a first-insert per §4.5.

#### Test #41 — `removeRanges(start, end)` hits multiple elements
**Given:** `r.insert(Strong(), 0, 10); r.insert(Italic(), 5, 15); r.insert(Code(), 100, 110); v0 := r.version`
**When:** `r.removeRanges(3, 8); v1 := r.version`
**Then:** `Strong().getRange(from: r) == [(0, 2), (9, 10)]`; `Italic().getRange(from: r) == [(9, 15)]`; `Code().getRange(from: r) == [(100, 110)]`; `v1 == v0 + 1` (single bump, NOT three).
**Justification:** §6.9 step 5 single atomic bump.

#### Test #42 — `removeRanges(start, end)` no overlap MUST NOT bump version
**Given:** `r.insert(Strong(), 0, 10); r.insert(Italic(), 50, 60); v0 := r.version`
**When:** `r.removeRanges(20, 30); v1 := r.version`
**Then:** `Strong().getRange(from: r) == [(0, 10)]`; `Italic().getRange(from: r) == [(50, 60)]`; `v0 == v1`.
**Justification:** §6.9 step 3 `any_change == false` ⇒ no-op per §4.10 (N3).

#### Test #43 — `removeRanges` fully covers some elements (mixed prune + retain)
**Given:** `r.insert(Strong(), 0, 5); r.insert(Italic(), 10, 20); r.insert(Code(), 25, 30); v0 := r.version`
**When:** `r.removeRanges(0, 30); v1 := r.version`
**Then:** `r.isEmpty == true`; `r.count == 0`; `v1 == v0 + 1`.
**(Variant)** With `removeRanges(8, 22)` instead: `Strong().getRange(from: r) == [(0, 5)]`; `Italic` is fully pruned; `Code().getRange(from: r) == [(25, 30)]`; `r.count == 2`; `r.ord(Strong) == 1`; `r.ord(Code) == 2` (renumbered from 3); single version bump.
**Justification:** §6.9 step 4 atomic rebuild of `insertion_order` + dense `ord` renumber.

### 10.C  Union Tests (v2 normative; #44–#50)

#### Test #44 — `union` with disjoint elements
**Given:** `r1 := Rangeable<Markup>(); r1.insert(Strong(), 0, 5)`; `r2 := Rangeable<Markup>(); r2.insert(Italic(), 10, 15)`
**When:** `r3 := r1.union(r2)`
**Then:** `r3.count == 2`; `r3.insertion_order == [Strong, Italic]`; `Strong().getRange(from: r3) == [(0, 5)]`; `Italic().getRange(from: r3) == [(10, 15)]`; `r3.version == 0`; `r1.version` and `r2.version` unchanged.

#### Test #45 — `union` with same element, overlapping intervals
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Strong(), 5, 15)`
**When:** `r3 := r1.union(r2)`
**Then:** `Strong().getRange(from: r3) == [(0, 15)]`; `r3.count == 1`.
**Justification:** §6.10 `merge_disjoint_lists` collapses overlapping `(0, 10)` and `(5, 15)`.

#### Test #46 — `union` adjacency-merge
**Given:** `r1.insert(Strong(), 0, 5)`; `r2.insert(Strong(), 6, 10)`
**When:** `r3 := r1.union(r2)`
**Then:** `Strong().getRange(from: r3) == [(0, 10)]`.
**Justification:** §4.3 integer-adjacency rule via `append_or_merge` (§6.10): `5 + 1 == 6`.

#### Test #47 — `union` with idempotent subset (mutating form MUST NOT bump version)
**Given:** `r1.insert(Strong(), 0, 10); r1.insert(Italic(), 20, 30)`; `r2.insert(Strong(), 3, 7)` (`R_r2(Strong) ⊆ R_r1(Strong)`); `v0 := r1.version`
**When:** `r1.formUnion(r2)` (or language-specific mutating spelling); `v1 := r1.version`
**Then:** `Strong().getRange(from: r1) == [(0, 10)]`; `Italic().getRange(from: r1) == [(20, 30)]`; `v0 == v1`.
**Justification:** §6.10 idempotence dual of §3.2 — when `result` is structurally equal to `self`, no version bump.

#### Test #48 — `union` of two empties = empty
**Given:** `r1 := Rangeable<Markup>(); r2 := Rangeable<Markup>()`
**When:** `r3 := r1.union(r2)`
**Then:** `r3.isEmpty == true`; `r3.count == 0`; `r3.version == 0`.

#### Test #49 — `union` with self
**Given:** `r1.insert(Strong(), 0, 5); r1.insert(Italic(), 10, 15); v0 := r1.version`
**When:** `r2 := r1.union(r1)`
**Then:** `r2` is structurally equal to `r1` (same `insertion_order`, same `intervals`); `r2.version == 0`; `r1.version == v0` (source unchanged for non-mutating).
**(Mutating variant)** `r1.formUnion(r1)` ⇒ `r1.version == v0` (no bump per §6.10 idempotence dual).

#### Test #50 — `union` insertion-order tail-append
**Given:** `r1.insert(A, 0, 1); r1.insert(B, 2, 3)`; `r2.insert(C, 4, 5); r2.insert(B, 10, 11); r2.insert(D, 12, 13)`
**When:** `r3 := r1.union(r2)`
**Then:** `r3.insertion_order == [A, B, C, D]`; specifically `r3.ord(A) == 1`, `r3.ord(B) == 2`, `r3.ord(C) == 3`, `r3.ord(D) == 4`.
**Justification:** §6.10 step 1 walks `r1.insertion_order` → `[A, B]`; step 2 walks `r2.insertion_order` → `[C, B, D]` and skips `B` (already in `r1`), appending `C` then `D` in `r2`'s order. **NOT** alphabetical, NOT hash-bucket.

### 10.D  Intersect Tests (v2 normative; #51–#57)

#### Test #51 — `intersect` with no shared elements
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Italic(), 5, 15)`
**When:** `r3 := r1.intersect(r2)`
**Then:** `r3.isEmpty == true`; `r3.count == 0`.
**Justification:** §6.11: `keys(r1) ∩ keys(r2) == ∅`.

#### Test #52 — `intersect` with shared elements, overlapping intervals
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Strong(), 5, 15)`
**When:** `r3 := r1.intersect(r2)`
**Then:** `Strong().getRange(from: r3) == [(5, 10)]`; `r3.count == 1`.
**Justification:** §6.11 `intersect_disjoint_lists`: `max(0, 5) = 5`, `min(10, 15) = 10`.

#### Test #53 — `intersect` with shared elements, disjoint intervals (element pruned)
**Given:** `r1.insert(Strong(), 0, 5)`; `r2.insert(Strong(), 100, 200)`
**When:** `r3 := r1.intersect(r2)`
**Then:** `r3.isEmpty == true`; `Strong()` is NOT in `keys(r3)`.
**Justification:** §6.11 eager prune (`length(intersected) == 0` ⇒ `continue`); §4.10 (N1).

#### Test #54 — `intersect` with self
**Given:** `r1.insert(Strong(), 0, 5); r1.insert(Italic(), 10, 15); v0 := r1.version`
**When:** `r2 := r1.intersect(r1)`
**Then:** `r2` is structurally equal to `r1`; `r2.version == 0`; `r1.version == v0`.

#### Test #55 — `intersect` with empty `Rangeable`
**Given:** `r1.insert(Strong(), 0, 5); r1.insert(Italic(), 10, 15)`; `r2 := Rangeable<Markup>()`
**When:** `r3 := r1.intersect(r2)`
**Then:** `r3.isEmpty == true`.

#### Test #56 — `intersect` produces multiple sub-intervals per element
**Given:** `r1.insert(Strong(), 0, 5); r1.insert(Strong(), 10, 15); r1.insert(Strong(), 20, 25)`; `r2.insert(Strong(), 3, 22)`
**When:** `r3 := r1.intersect(r2)`
**Then:** `Strong().getRange(from: r3) == [(3, 5), (10, 15), (20, 22)]`.
**Justification:** §6.11 `intersect_disjoint_lists` two-pointer sweep produces three sub-intervals: `(3, 5)`, `(10, 15)`, `(20, 22)`.

#### Test #57 — `intersect` insertion-order preservation + dense `ord` renumber
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15); r1.insert(C, 20, 25); r1.insert(D, 30, 35)`; `r2.insert(A, 0, 5); r2.insert(C, 21, 24); r2.insert(E, 100, 200)`
**When:** `r3 := r1.intersect(r2)`
**Then:** `r3.insertion_order == [A, C]` (B and D dropped from `keys(r1) ∖ keys(r2)`; E dropped from `keys(r2) ∖ keys(r1)`); `r3.ord(A) == 1`; `r3.ord(C) == 2` (densely renumbered, NOT 3 from `r1`'s original `ord(C) == 3`); `Strong().getRange(from: r3)` for `A == [(0, 5)]`, `C == [(21, 24)]`.

### 10.E  Difference Tests (v2 normative; #58–#65)

#### Test #58 — `difference` with disjoint elements (returns self structurally)
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Italic(), 5, 15)`
**When:** `r3 := r1.difference(r2)`
**Then:** `r3` is structurally equal to `r1` (Strong intact; `Italic` is not in `keys(r1)` so untouched); `r3.version == 0`.

#### Test #59 — `difference` with self = empty
**Given:** `r1.insert(Strong(), 0, 10); r1.insert(Italic(), 20, 30)`
**When:** `r2 := r1.difference(r1)`
**Then:** `r2.isEmpty == true`; `r2.count == 0`.

#### Test #60 — `difference` creates left residuals
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Strong(), 5, 100)`
**When:** `r3 := r1.difference(r2)`
**Then:** `Strong().getRange(from: r3) == [(0, 4)]`.

#### Test #61 — `difference` creates right residuals
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Strong(), -100, 5)`
**When:** `r3 := r1.difference(r2)`
**Then:** `Strong().getRange(from: r3) == [(6, 10)]`.

#### Test #62 — `difference` creates both residuals (split)
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Strong(), 3, 6)`
**When:** `r3 := r1.difference(r2)`
**Then:** `Strong().getRange(from: r3) == [(0, 2), (7, 10)]`.

#### Test #63 — `difference` spans multiple `L_a` entries
**Given:** `r1.insert(Strong(), 0, 5); r1.insert(Strong(), 10, 15); r1.insert(Strong(), 20, 25)`; `r2.insert(Strong(), 3, 22)`
**When:** `r3 := r1.difference(r2)`
**Then:** `Strong().getRange(from: r3) == [(0, 2), (23, 25)]`.
**Justification:** §6.12 `subtract_disjoint_lists`: `(0, 5)` cut by `(3, 22)` ⇒ `(0, 2)`; `(10, 15)` fully consumed; `(20, 25)` cut ⇒ `(23, 25)`.

#### Test #64 — `difference` insertion-order preservation
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15); r1.insert(C, 20, 25); r1.insert(D, 30, 35)`; `r2.insert(B, 9, 16); r2.insert(E, 100, 200)`
**When:** `r3 := r1.difference(r2)`
**Then:** `r3.insertion_order == [A, C, D]` (B fully consumed and pruned; E ignored since not in `keys(r1)`); `r3.ord(A) == 1`, `r3.ord(C) == 2`, `r3.ord(D) == 3`.

#### Test #65 — `difference` ≡ per-element `remove`-loop equivalence (cross-validation probe)
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Strong(), 3, 6)` (single-element scenario; `r2.intervals[Strong] == [(3, 6)]`)
**When:** `r3 := r1.difference(r2)`; `r4 := r1.copy(); for ((lo, hi) in r2.getRange(of: Strong())) { r4.remove(Strong(), lo, hi) }`
**Then:** `r3.insertion_order == r4.insertion_order == [Strong]`; `Strong().getRange(from: r3) == Strong().getRange(from: r4) == [(0, 2), (7, 10)]`.
**Justification:** §6.12 informative equivalence note (per-element form). The equivalence target is `remove(e, lo, hi)` (per-key surgical erase), NOT `removeRanges(lo, hi)` (cross-element erase). The `removeRanges` form is NOT equivalent in multi-element scenarios because `removeRanges(lo, hi)` cuts that range out of EVERY key, while `difference(other)` only uses `R_other(e)` to cut `R_self(e)` for matching `e` (per-key surgical erase). Counter-example: `r1.Italic == [(5, 15)]`, `r2.Strong == [(0, 10)], r2.Italic == [(12, 18)]` — `r1.difference(r2)` gives `Italic == [(5, 11)]` (only cut by `r2.Italic`), but `r4.removeRanges(0, 10); r4.removeRanges(12, 18)` would give `Italic == [(11, 11)]` (also cut by `r2.Strong`'s range). (Note: `r3.version == 0` since fresh; `r4.version == r1.version + 1` since one `remove` bump. Equivalence is on STRUCTURE, not version count.)

### 10.F  Symmetric Difference Tests (v2 normative; #66–#71)

#### Test #66 — `symmetric_difference` with empty = self structurally
**Given:** `r1.insert(Strong(), 0, 5); r1.insert(Italic(), 10, 15)`; `r2 := Rangeable<Markup>()`
**When:** `r3 := r1.symmetric_difference(r2)`
**Then:** `r3.insertion_order == [Strong, Italic]`; `Strong().getRange(from: r3) == [(0, 5)]`; `Italic().getRange(from: r3) == [(10, 15)]`; `r3.version == 0`.

#### Test #67 — `symmetric_difference` with self = empty
**Given:** `r1.insert(Strong(), 0, 5); r1.insert(Italic(), 10, 15)`
**When:** `r2 := r1.symmetric_difference(r1)`
**Then:** `r2.isEmpty == true`; `r2.count == 0`.

#### Test #68 — `symmetric_difference` per-element residuals from both sides
**Given:** `r1.insert(Strong(), 0, 10)`; `r2.insert(Strong(), 5, 15)`
**When:** `r3 := r1.symmetric_difference(r2)`
**Then:** `Strong().getRange(from: r3) == [(0, 4), (11, 15)]`.
**Justification:** §6.13: `a = subtract(r1, r2) = [(0, 4)]`; `b = subtract(r2, r1) = [(11, 15)]`; `merge_disjoint_lists(a, b) = [(0, 4), (11, 15)]` (non-adjacent).

#### Test #69 — `symmetric_difference` commutativity (modulo insertion_order)
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15)`; `r2.insert(B, 12, 17); r2.insert(C, 20, 25)`
**When:** `r3 := r1.symmetric_difference(r2)`; `r4 := r2.symmetric_difference(r1)`
**Then:** for every key `e ∈ keys(r3)`: `getRange(of: e, from: r3) == getRange(of: e, from: r4)` (per-element `R(e)` is identical). Specifically: `A == [(0, 5)]`, `B == [(10, 11), (16, 17)]`, `C == [(20, 25)]`. **However:** `r3.insertion_order == [A, B, C]` (self-primary `r1`); `r4.insertion_order == [B, C, A]` (self-primary `r2`). **The `insertion_order` IS NOT commutative; this is by design (§4.10 self-primary rule).**

#### Test #70 — `symmetric_difference` associativity
**Given:** `r1.insert(A, 0, 10)`; `r2.insert(A, 5, 15)`; `r3.insert(A, 10, 20)`
**When:** `r_left := r1.symmetric_difference(r2).symmetric_difference(r3)`; `r_right := r1.symmetric_difference(r2.symmetric_difference(r3))`
**Then:** `getRange(of: A, from: r_left) == getRange(of: A, from: r_right) == [(0, 4), (10, 10), (16, 20)]`. Both `r_left.insertion_order` and `r_right.insertion_order` are `[A]` (single shared key throughout). `r_left.ord(A) == r_right.ord(A) == 1`.

**Worked derivation (left association `(r1 △ r2) △ r3`):**
- `r1 △ r2`: `a = subtract([(0,10)], [(5,15)]) = [(0, 4)]`; `b = subtract([(5,15)], [(0,10)]) = [(11, 15)]`; `merge(a, b) = [(0, 4), (11, 15)]` (gap `4+1 = 5 < 11`, no collapse). So `R(A) = [(0, 4), (11, 15)]`.
- `(r1 △ r2) △ r3`: `a = subtract([(0,4), (11,15)], [(10,20)]) = [(0, 4)]` (the `(11,15)` entry is fully consumed by `(10,20)`; the `(0,4)` entry has no overlap with `(10,20)` so it survives intact); `b = subtract([(10,20)], [(0,4), (11,15)]) = [(10, 10), (16, 20)]` (the cut by `(11,15)` leaves left residual `(10, 10)` and right residual `(16, 20)`); `merge(a, b) = [(0, 4), (10, 10), (16, 20)]` (gaps `4+1 = 5 < 10` and `10+1 = 11 < 16`, no collapse).

**Worked derivation (right association `r1 △ (r2 △ r3)`):**
- `r2 △ r3`: `a = subtract([(5,15)], [(10,20)]) = [(5, 9)]`; `b = subtract([(10,20)], [(5,15)]) = [(16, 20)]`; `merge(a, b) = [(5, 9), (16, 20)]` (gap `9+1 = 10 < 16`, no collapse). So `R(A) = [(5, 9), (16, 20)]`.
- `r1 △ (r2 △ r3)`: `a = subtract([(0,10)], [(5,9), (16,20)]) = [(0, 4), (10, 10)]` (cut by `(5,9)` produces left residual `(0, 4)` and right residual `(10, 10)`; then `(16, 20)` does not touch `(10, 10)`); `b = subtract([(5,9), (16,20)], [(0,10)]) = [(16, 20)]` (the `(5, 9)` entry is fully consumed by `(0, 10)`; `(16, 20)` survives); `merge(a, b) = [(0, 4), (10, 10), (16, 20)]`.

**Justification (why associativity holds in closed-interval-with-integer-adjacency semantics).** Symmetric difference is XOR on characteristic functions (`χ_{A△B}(x) = χ_A(x) ⊕ χ_B(x)`); XOR is associative, so the underlying integer set `{x | x ∈ R(A) at position x}` is associative regardless of grouping. Canonicalization (the (I1) merge-on-adjacency rewrite for integer coords) is a deterministic function of the integer set, so byte-identical canonical `R(A)` follows from set-equal results. ∎

#### Test #71 — `symmetric_difference` insertion-order tail-append for keys ∈ other ∖ self
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15)`; `r2.insert(C, 20, 25); r2.insert(D, 30, 35)`
**When:** `r3 := r1.symmetric_difference(r2)`
**Then:** `r3.insertion_order == [A, B, C, D]`; `r3.ord(A) == 1`, `r3.ord(B) == 2`, `r3.ord(C) == 3`, `r3.ord(D) == 4`.
**Justification:** §6.13 step 1 walks `r1.insertion_order` → `[A, B]` (no overlap with `r2`, so unchanged); step 2 walks `r2.insertion_order` → `[C, D]` (none in `r1`, so tail-appended in `r2`'s order).

### 10.G  Set-op Insertion-order Stress Tests (v2 normative; #72–#80)

> These tests exist specifically to catch subtle bugs where an implementation accidentally relies on `Hash` iteration order (Ruby/Python `dict`) or hash-bucket order (Swift `Dictionary`) instead of the spec-mandated `insertion_order`. They also stress the dense `ord` renumber rule across multiple consecutive set ops.

#### Test #72 — Dense `ord` renumber after multi-element prune
**Given:** `r1.insert(A, 0, 1); r1.insert(B, 2, 3); r1.insert(C, 4, 5); r1.insert(D, 6, 7); r1.insert(E, 8, 9)`; `r2.insert(B, 100, 200); r2.insert(D, 100, 200)`
**When:** `r3 := r1.intersect(r2)`
**Then:** `r3.isEmpty == true` (B's intersect with `(100, 200)` is empty since `R_r1(B) = [(2, 3)]`; same for D); `r3.count == 0`.
**Justification:** §6.11 eager-prune cascades for both B and D.

#### Test #73 — `union` then `intersect` chain preserves `insertion_order`
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15)`; `r2.insert(C, 20, 25); r2.insert(B, 12, 17)`; `r3.insert(B, 0, 100); r3.insert(C, 0, 100)`
**When:** `r_chain := r1.union(r2).intersect(r3)`
**Then:** intermediate `r1.union(r2).insertion_order == [A, B, C]`. Then intersect with `r3` (which has only B, C in keys): `r_chain.insertion_order == [B, C]` (A dropped from `keys(r1∪r2) ∩ keys(r3)`); `r_chain.ord(B) == 1`, `r_chain.ord(C) == 2` (densely renumbered).

#### Test #74 — Set-op result `ord` is correct even if input `Rangeable` had pruned elements
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15); r1.insert(C, 20, 25); r1.remove(B)` (so `r1.insertion_order == [A, C]`, `r1.ord(C) == 2`)
**When:** `r2 := r1.union(empty)`
**Then:** `r2.insertion_order == [A, C]`; `r2.ord(A) == 1`; `r2.ord(C) == 2`.
**Justification:** §6.10 walks the *current* `r1.insertion_order` (post-prune); the pre-prune `ord(C) == 3` is irrelevant.

#### Test #75 — `difference` then `union` recovers `insertion_order` correctly
**Given:** `r1.insert(A, 0, 10); r1.insert(B, 20, 30); r1.insert(C, 40, 50)`; `r2.insert(B, 0, 100)` (fully consumes B)
**When:** `r3 := r1.difference(r2).union(r1)`
**Then:** `r3.insertion_order == [A, C, B]`. Worked: `r1.difference(r2)` has `keys == [A, C]` (B pruned); union with `r1` re-introduces B at the tail per §6.10 step 2 rule.

#### Test #76 — Union of three with overlapping keys
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15)`; `r2.insert(B, 20, 25); r2.insert(C, 30, 35)`; `r3.insert(C, 40, 45); r3.insert(D, 50, 55)`
**When:** `r_chain := r1.union(r2).union(r3)`
**Then:** `r_chain.insertion_order == [A, B, C, D]`; specifically `r_chain.ord(A) == 1`, `r_chain.ord(B) == 2`, `r_chain.ord(C) == 3`, `r_chain.ord(D) == 4`. Per-element intervals: `A == [(0, 5)]`; `B == [(10, 15), (20, 25)]`; `C == [(30, 35), (40, 45)]`; `D == [(50, 55)]`.

#### Test #77 — `symmetric_difference` two algebraic-form equivalence
**Given:** `r1.insert(A, 0, 10); r1.insert(B, 20, 30)`; `r2.insert(A, 5, 15); r2.insert(C, 40, 50)`
**When:** `r_form1 := r1.symmetric_difference(r2)` (direct `(A∖B) ∪ (B∖A)` form per §6.13); `r_form2 := r1.union(r2).difference(r1.intersect(r2))` (alternate `(A∪B) ∖ (A∩B)` form)
**Then:** for every key `e ∈ keys(r_form1) ∪ keys(r_form2)`: `getRange(of: e, from: r_form1) == getRange(of: e, from: r_form2)` (per-element identical). **Note:** `insertion_order` MAY differ between the two forms; the canonical (spec-mandated) is `r_form1`'s form. Implementations that internally use form 2 MUST post-normalise to match form 1's `insertion_order` (self-primary then tail-append).

#### Test #78 — Insert-after-remove `ord` reassignment (R14)
**Given:** `r.insert(A, 0, 5); r.insert(B, 10, 15)` (so `r.ord(A) == 1`, `r.ord(B) == 2`)
**When:** `r.remove(A)` (so `r.insertion_order == [B]`, `r.ord(B) == 1`); then `r.insert(A, 100, 110)`
**Then:** `r.insertion_order == [B, A]`; `r.ord(B) == 1`; `r.ord(A) == 2` (NOT 1; A is now a NEW first-insert at the tail).
**Justification:** §4.10 deliberate side effect.

#### Test #79 — Cross-op `ord` consistency (intersect after union)
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15); r1.insert(C, 20, 25)`; `r2.insert(B, 12, 17); r2.insert(D, 30, 35)`
**When:** `r_union := r1.union(r2)`; `r3.insert(B, 0, 100); r3.insert(D, 0, 100); r3.insert(A, 0, 100)`; `r_intersect := r_union.intersect(r3)`
**Then:** `r_union.insertion_order == [A, B, C, D]`. `r_intersect.insertion_order == [A, B, D]` (C dropped since not in `keys(r3)`); densely renumbered: `r_intersect.ord(A) == 1`, `r_intersect.ord(B) == 2`, `r_intersect.ord(D) == 3`.

#### Test #80 — Empty result eager prune across set-op chain
**Given:** `r1.insert(A, 0, 5); r1.insert(B, 10, 15)`; `r2.insert(A, 100, 200); r2.insert(B, 100, 200)` (no overlap with `r1` for either key)
**When:** `r3 := r1.intersect(r2)`; `r4 := r3.union(r1)`
**Then:** `r3.isEmpty == true` (both A and B intersect to empty, both pruned). `r4.insertion_order == [A, B]` (recovered from `r1` via union step 2, since `r3` has no keys); `r4.ord(A) == 1`, `r4.ord(B) == 2`.

---

## §11  Threading / Mutation Guarantees

> v1 pins a **single-writer / multi-reader** model.

- (T1) If multiple threads share a `Rangeable`, **concurrent** reads MUST be safe (the immutable referenced `event_index` is read-only after build).
- (T2) **Concurrent writes** and **concurrent read+write** are **not** part of v1's safety guarantee. Callers MUST synchronise externally (e.g. with an external lock).
- (T3) The lazy event-index build is **idempotent**: if two reader threads trigger a build concurrently, each builds one. **The cache-write path MUST satisfy the §5.2 (I3.d) version re-check invariant**, i.e. re-compare `event_index.version == r.version` within the atomic step of cache write; otherwise the stale build MUST be discarded. Implementations SHOULD adopt one of the two patterns below:

  - **(T3.M) Mutex pattern (recommended):** the entire `ensure_event_index_fresh` path (check → build → write) holds a cache mutex. The mutex guarantees that no concurrent writer advances `r.version` while the build runs (per (T2)), so the write path naturally satisfies (I3.d).
  - **(T3.C) CAS pattern:** snapshot `v_start` before the build; on completion, `compare_and_swap(event_index, expected: nil_or_stale, new: built)`; if the CAS fails or an immediate re-read shows `r.version != v_start`, retry or discard.

  Ruby's GVL model naturally satisfies (T3.M) (only one thread is executing Ruby code at a time); but in fiber / Ractor scenarios an explicit Mutex is still required. Swift implementations SHOULD wrap the cache-write path with `OSAllocatedUnfairLock` or `NSLock`; under stricter invariants (`@unchecked Sendable` + atomic) they MAY adopt (T3.C).

  > **Normative consequence:** a stale write arriving late after a racy build MUST NOT appear in user-visible behavior; any query's `event_index` MUST correspond to some `r.version` snapshot reachable as of the query's time.
- (T4) The version-counter operation must be atomic (a wall-monotonic counter suffices under the single-writer model).

> **Difference from Boost.ICL:** Boost.ICL explicitly declares itself single-thread; this RFC slightly extends to single-writer / multi-reader. The reason is that the typical markdown workload is "parser thread inserts everything once, then a renderer pool reads concurrently".

### 11.1 Memory consistency model (informative)

For Swift (typically Acquire/Release semantics via `os_unfair_lock` or `OSAllocatedUnfairLock`), the publish order for the cached `event_index` pointer SHOULD be:

1. Complete the build of `event_index` (write to a local var);
2. Use a Release store to write the pointer into `self.eventIndex`;
3. Readers use an Acquire load to read `self.eventIndex` ⇒ ensuring they see the fully built result.

For Ruby, the GVL provides implicit atomic store/load; however, in Ractor or fiber-based concurrency scenarios the caller MUST add an explicit Mutex.

### 11.2 Precise definition of "single writer"

The precise (T2) "single writer" invariant:

> **(SW1)** At any instant, at most one thread is calling `insert` (or any v2 mutation) on `r`.
> **(SW2)** A reader thread's `[i]` / `transitions` / `getRange` MUST NOT execute concurrently with a writer thread's `insert`; callers MUST establish ordering via an external happens-before edge (e.g. lock release/acquire, `Thread.join`).

A common legal pattern in practice:

```
# Phase 1 (single thread builds)
r = Rangeable.new
markups.each { |m| r.insert(m.token, start: m.start, end: m.end) }
r.freeze   # informative: signals "no further mutation"

# Phase 2 (worker pool reads concurrently)
workers.each { |w| w.spawn { ... uses r[i].objs ... } }
```

The happens-before relation across the Phase 1 → Phase 2 transition is provided by `freeze` / barrier / pool spawn.

---

## §12  Implementation Notes

### 12.1 Ruby

```ruby
class Rangeable
  Interval = Struct.new(:lo, :hi)
  InvalidIntervalError = Class.new(ArgumentError)

  attr_reader :version

  def initialize
    @intervals       = {}      # element => Array<Interval>, ascending by lo
    @insertion_order = []      # Array<element>
    @ord             = {}      # element => Integer
    @version         = 0
    @event_index     = nil
  end

  def insert(element, start:, end:)
    raise InvalidIntervalError, "start (#{start}) > end (#{end})" if start > self.end_arg(end_)
    e = element.dup.freeze rescue element  # 4.6 M2, shallow freeze
    list = @intervals[e]
    if list.nil?
      list = []
      @intervals[e] = list
      @insertion_order << e
      @ord[e] = @insertion_order.length
    end
    lo, hi = start, end
    i = list.bsearch_index { |iv| iv.hi >= lo - 1 } || list.length
    merged = false
    while i < list.length && list[i].lo <= hi + 1
      lo = [lo, list[i].lo].min
      hi = [hi, list[i].hi].max
      list.delete_at(i)
      merged = true
    end
    list.insert(i, Interval.new(lo, hi))
    @version += 1 unless _idempotent_no_op?(merged, ...)  # simplified; see §6.1 cleaner variant
    @event_index = nil
    self
  end

  def [](i) Slot.new(_active_at(i)) end

  def getRange(of: element)
    list = @intervals[element]
    return [] unless list
    list.map { |iv| [iv.lo, iv.hi] }
  end

  def transitions(over:)
    _ensure_event_index
    # bsearch + slice up to hi+1
    # ...
  end

  private

  def _ensure_event_index
    return if @event_index && @event_index[:version] == @version
    @event_index = _build_event_index
  end

  def _build_event_index
    events = []
    @intervals.each do |e, list|
      list.each do |iv|
        events << [iv.lo,     :open,  e, @ord[e]]
        events << [iv.hi + 1, :close, e, @ord[e]]
      end
    end
    events.sort_by! do |coord, kind, _, ord|
      [coord, kind == :open ? 0 : 1, kind == :open ? ord : -ord]
    end
    segments = _materialise_segments(events)
    { events: events, segments: segments, version: @version }
  end

  def _active_at(i)
    _ensure_event_index
    seg = _segment_containing(i)
    seg ? seg[:active].dup : []
  end

  # ...
end
```

**Ruby implementation notes:**

- `Hash` key equality uses `eql?` + `hash`, exactly matching §4.2. Elements must implement `eql?` and `hash` (in most cases `Struct` provides them automatically).
- `Array#bsearch_index` (Ruby ≥ 2.3) provides an `O(log n)` lower_bound on a sorted Array.
- `dup.freeze` is a shallow freeze; for nested mutable structures the caller remains responsible for (M1).
- The keyword argument `end:` of `insert` does not collide with Ruby's keyword `end` (it is allowed in a method's parameter position).

### 12.2 Swift

```swift
public struct Rangeable<Element: Hashable> {
  public struct InvalidIntervalError: Error {
    public let start: Int
    public let end: Int
  }

  public struct Slot {
    public let objs: [Element]
  }

  public struct TransitionEvent {
    public enum Kind { case open, close }
    public let coordinate: Int
    public let kind: Kind
    public let element: Element
  }

  private var intervals: [Element: [(lo: Int, hi: Int)]] = [:]
  private var insertionOrder: [Element] = []
  private var ord: [Element: Int] = [:]
  public private(set) var version: Int = 0
  private var eventIndex: EventIndex?  // class-wrapped for COW invalidation control

  public init() {}

  public mutating func insert(_ e: Element, start: Int, end: Int) throws {
    guard start <= end else {
      throw InvalidIntervalError(start: start, end: end)
    }
    if intervals[e] == nil {
      intervals[e] = []
      insertionOrder.append(e)
      ord[e] = insertionOrder.count
    }
    var list = intervals[e]!  // COW; this triggers copy iff shared
    var lo = start, hi = end
    let i0 = list.firstIndex(where: { $0.hi >= lo - 1 }) ?? list.count
    var i = i0
    var changed = false
    while i < list.count, list[i].lo <= hi + 1 {
      lo = min(lo, list[i].lo)
      hi = max(hi, list[i].hi)
      list.remove(at: i)
      changed = true
    }
    list.insert((lo: lo, hi: hi), at: i0)
    intervals[e] = list
    if changed || /* confirm not an idempotent no-op */ true {
      version += 1
      eventIndex = nil
    }
  }

  public subscript(i: Int) -> Slot {
    mutating get { /* ensure event index, return Slot */ }
  }

  public func getRange(of e: Element) -> [(Int, Int)] {
    intervals[e] ?? []
  }

  // ...
}

// Sugar: e.getRange(from: r)
// Note: the original `extension Rangeable.Element where Element: Hashable {…}` is **not valid Swift**
// (`Element` is a generic parameter, not a type; one cannot reach it with dot-syntax in an outer extension).
// Two valid alternatives:
//
// (A) Extend Hashable directly to provide a free-style sugar:
public extension Hashable {
    func getRange<E: Hashable>(from r: Rangeable<E>) -> [(Int, Int)] where Self == E {
        r.getRange(of: self)
    }
}
//
// (B) Provide a module-level free function (no protocol pollution):
public func getRange<E: Hashable>(of e: E, from r: Rangeable<E>) -> [(Int, Int)] {
    r.getRange(of: e)
}
//
// This RFC adopts (A) as the reference sugar; (B) is a backup (for callers who do
// not wish to pollute the Hashable namespace).
```

**Swift implementation notes:**

- `Hashable`'s contract (`==` implies same `hash`) satisfies §4.2 (E1–E4).
- `Dictionary` and `Array` are value types with COW, naturally satisfying §4.6 (M2).
- `firstIndex(where:)` is `O(n)`; in high-`m_e` scenarios it SHOULD be replaced with a binary-search helper (the standard library has no built-in lower_bound; a 22-line helper is required).
- `subscript get` is a `mutating get` because the lazy build mutates `eventIndex`; if a non-mutating `get` is desired, wrap `eventIndex` with a `class` reference.
- `Sendable` convention: under v1's single-writer / multi-reader model, the entire struct SHOULD be annotated `@unchecked Sendable` with synchronisation controlled by the caller; a more correct version SHOULD use an `actor` wrapper in v2.

---

## §13  Non-goals (v2 OUT OF SCOPE)

> **v2 update.** Items previously listed here as non-goals — Removal (`remove(e, start:, end:)`, `remove(e:)`, `removeAll() / clear()`, `removeRanges(start:, end:)`) and Set Operations between two `Rangeable`s (`union`, `intersect`, `difference`, `symmetric_difference`) — are **now normative in v2**; see §6.6–§6.13 (algorithms), §10.B–§10.G (test contracts #21–#80), §3.6 (API surface), §4.10 (eager-pruning semantics), §7.6 / §7.7 (complexity bounds). The remaining non-goals are unchanged from v1 and listed below.

The following are **explicitly excluded** from v2:

1. **Persistent / Immutable / Snapshot semantics:** no `r.snapshot()` or functional `inserting(...)`. Rationale: see §14.2.
2. **Multi-dimensional / rectangle stabbing:** v2 is 1D only.
3. **Floating-point / continuous coordinates:** the §4.3 adjacency-merge rule does not hold on dense rationals.
4. **Aggregate value (Boost.ICL `interval_map<K, V>` style):** no `+=` aggregate-on-overlap.
5. **`r[lo...hi].objs` (range subscript):** not in v2; callers SHOULD assemble the result themselves with `transitions(over:)`.
6. **Logical-clock semantics (versioning, invalidation history):** the `version` counter is for internal cache invalidation; it is **not** a monotonic logical clock, and callers MUST NOT rely on it for conflict resolution.

---

## §14  Future Work

### 14.1 Removal (historical pointer)

> **Implemented in v2.** See §6.6 (`remove(e, start, end)`), §6.7 (`remove(e)`), §6.8 (`removeAll() / clear()`), §6.9 (`removeRanges(start, end)`); §4.10 for the eager-pruning rule that anchors all four ops; §7.6 for complexity bounds; §10.B (#21–#43) for the test contract.
>
> **Resolution of the v1-noted "potential function" complication.** v1's §7.2 analysis used a monotone potential `Φ := M + …` that depends on the assumption "M never decreases" (insert-only). Remove can decrease M (full prune of an element) or increase M (split one entry into two), so the v1 monotone-Φ argument no longer applies. v2 sidesteps this by **not requiring a monotone amortised bound for remove**: §7.6 specifies direct per-op bounds (`O(log m_e + k + E)` for `remove(e, start, end)`, `O(E)` for `remove(e)`, `O(1)` for `clear()`), which are `O(log m_e + k)` plus an eager-prune renumber term that is amortised over the element lifecycle (informative argument in §7.6). No redesigned `Φ` is needed; the spec's claim is a per-op direct bound, not an amortised-via-potential bound.

### 14.2 Persistent / Immutable variant

- **Path copying** (Driscoll, Sarnak, Sleator, Tarjan 1986) as the simplest baseline: each insert clones one path from the root to the modified leaf, with all other subtrees shared. Space `O(log m_e)` per insert.
- **Fat-node + companion node:** `O(1)` amortised overhead per modification + `O(1)` access slowdown.
- **Motivation:** editor / collaboration scenarios need history snapshots; v1's mutating-in-place model does not provide this.

### 14.3 Floating-point / continuous coordinates (v3)

- Drop the §4.3 adjacency-merge rule; merge only on strict overlap.
- Use the half-open notation `[a, b)` to avoid the `a ≤ x ≤ b` vs. `a ≤ x < b` boundary confusion.
- Use ε-tolerance for approximate comparison or exact rational arithmetic.

### 14.4 Two-dimensional extension (rectangle stabbing)

- `Rangeable<Element>` ⇒ `RangeableRect<Element>` supporting `r[x, y].objs`.
- Structure: a multi-level segment tree (CLRS §14 + de Berg et al. 5.1); space `O(M log M)`, query `O(log² M + r)`.

### 14.5 Aggregate variant

- Provide `RangeableAgg<Element, Value>` with `+=` to accumulate values, mimicking Boost.ICL `interval_map`.
- Both containers coexist: `Rangeable` is set membership; `RangeableAgg` is weighted.

### 14.6 Roaring-style compression

- For very large `m_e`, the per-element list could switch to a Roaring run container (consecutive sorted runs). Space `O(R)` where `R` is the number of incompressible runs.

### 14.7 Concurrent version

- `actor RangeableActor<Element>` for Swift; Ractor or a Mutex wrapper for Ruby.

### 14.8 Streaming queries

- A lazy-iterator version of `transitions(over:)` (Swift `AsyncSequence` / Ruby `Enumerator`).

### 14.9 Set operations between two `Rangeable<Element>` (historical pointer + sym-diff amendment)

> **Implemented in v2.** See §6.10 (`union`), §6.11 (`intersect`), §6.12 (`difference`), §6.13 (`symmetric_difference`); §4.10 for the eager-pruning rule that ensures empty results are pruned cleanly across all four ops; §7.7 for complexity bounds (all four ops are `O(M_self + M_other)` via two-pointer sweep); §10.C–§10.G (#44–#80) for test contracts. The v1-noted open questions about `insertion_order` rules and empty-`R(e)` handling are now both resolved (per-op rules in §4.10's tables and §6.10–§6.13 pseudocode).

#### 14.9.1 Symmetric difference inclusion (v2 amendment, justification)

v1's §14.9 sketched only `union` / `intersect` / `difference`. v2 additionally promotes **`symmetric_difference`** to normative. Justification (three points):

1. **Closed Boolean closure.** `symmetric_difference` is the unique additional Boolean set operation closed over `Rangeable<E>` beyond the three v1-sketched ops. The four-element Boolean lattice on two sets is `{∅, A∖B, B∖A, A∩B}`, whose pairwise unions yield the closure: `union = (A∖B) ∪ (A∩B) ∪ (B∖A)`, `intersect = A∩B`, `difference = A∖B`, `symmetric_difference = (A∖B) ∪ (B∖A)`. Without `symmetric_difference`, the closure is incomplete; library callers cannot express "the parts that differ between two snapshots" in a single op.
2. **Negligible implementation cost.** `symmetric_difference` is implementable in `O(M_self + M_other)` (per §7.7) via the same two-pointer sweep that powers `union` and `intersect`. The §6.13 pseudocode reuses `subtract_disjoint_lists` (from §6.12) and `merge_disjoint_lists` (from §6.10) directly; no new primitive algorithm is required.
3. **Concrete caller demand.** Library callers that compare two `Rangeable` snapshots (e.g. ZMediumToMarkdown comparing annotated-vs-canonical markup overlays; AVPlayer cache comparing on-disk vs in-flight byte ranges) need the "edits between snapshots" semantics. `union`, `intersect`, `difference` cannot express this in a single call.

**Algebraic identity used in the spec proof (also normative):** `A △ B := (A ∪ B) ∖ (A ∩ B) = (A ∖ B) ∪ (B ∖ A)`. Both forms compute identically under per-element `R(e)` algebra, allowing dual implementations for cross-validation (Test #77). The §6.13 pseudocode adopts the second form `(A ∖ B) ∪ (B ∖ A)` as the canonical reference, because it shares primitives with §6.12 and avoids the materialise-then-subtract round-trip of the first form.

#### 14.9.2 Insertion-order rules for set ops (v2-resolved)

The v1 §14.9 paragraph noted "the insertion_order merge rule MUST be specified … this RFC defers the choice to the v2 spec". v2 resolves this with the following normative table (also reproduced in §4.10's preamble cross-references and per-op pseudocode in §6.10–§6.13):

| Op | Insertion-order rule | Justification |
|---|---|---|
| `union(other)` | self primary; `keys(other) ∖ keys(self)` tail-appended in other's `insertion_order` order | Self-primary is the natural caller intent ("extend my container with other's contents"); tail-append for new keys preserves the §4.5 first-insert-ordinal monotonicity within each contributor |
| `intersect(other)` | preserve self's `insertion_order` over `keys(self) ∩ keys(other)`, drop pruned-empty; densely renumber `ord` 1..N | Self-primary symmetry with union; renumbering keeps `ord` contiguous so binary searches over `ord` stay correct |
| `difference(other)` | preserve self's `insertion_order` over surviving keys; drop pruned-empty; densely renumber `ord` | Difference cannot introduce other's elements, so other's order is irrelevant |
| `symmetric_difference(other)` | self primary on `keys(self) ∖ keys(other)` portion; tail-append `keys(other) ∖ keys(self)` portion in other's `insertion_order` order; for shared keys (where intervals can still cancel non-trivially), keep self's position; drop pruned-empty; densely renumber `ord` | Combines the union and difference rules; symmetric-up-to which side is `self` (Test #69 validates the documented self-primary asymmetry) |

**Renumbering is normative:** any op that drops elements MUST renumber `ord` `1..N` over the survivors in surviving-position order. Otherwise the `ord(e) ∈ [1, count]` invariant (§4.5, §5.1 I2.a) breaks, and `event_index`'s ord-based comparisons (§5.2, §6.3 `build_event_index`) produce the wrong order. The `insertion_order` array is rewritten from scratch from the survivors; the old numerical `ord` values are not preserved.

### 14.10 Worst-case `O(\log m_e)` per-op insert (improvement to the sorted-array variant)

§7.2.4 already formally records that sorted-array degrades to `O(m_e)` per-op under a head-insert adversarial sequence. Possible improvements:
- **(W1) Order-statistic tree:** replace the sorted Array with a weight-balanced tree (Reed 1973) or Tarjan's splay tree (1985); all ops are worst-case `O(\log m_e + k)`.
- **(W2) Logarithmic method (Bentley & Saxe 1980):** split `R(e)` into `O(\log m_e)` exponentially sized sorted arrays; amortised insert is `O(\log² m_e)` per op.
- **(W3) Skip list (probabilistic):** rejected in §8 (d) by the cross-language byte-identical requirement; can be considered if the cross-language requirement is relaxed.

---

## §15  References

> This section lists the **authoritative sources** cited during the research phase of this RFC. Each entry includes a verifiable medium (book name/chapter + ISBN, paper DOI or arXiv ID, or an official URL — at least one of these).

### Books

1. **CLRS, 4th edition**: Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, MIT Press, 2022. ISBN: 978-0-262-04630-5.
   - **§14.3 Interval trees** — augmented red-black tree with max endpoint, single-overlap stabbing in `O(log n)`.
   - **§17 Amortized Analysis** — aggregate / accounting / **potential method** (Φ-function), used in §7.2 of this RFC.
2. **de Berg, Cheong, van Kreveld, Overmars**, *Computational Geometry: Algorithms and Applications*, 3rd edition, Springer, 2008. ISBN: 978-3-540-77973-5. **§10 Interval, Segment, and Range Trees**.
3. **Tarjan, R. E.**, *Data Structures and Network Algorithms*, SIAM CBMS-NSF Regional Conference Series, 1983. ISBN: 978-0-89871-187-5. (Background on amortised credit/potential method.)
3a. **Mehlhorn, K. & Sanders, P.**, *Algorithms and Data Structures: The Basic Toolbox*, Springer, 2008. ISBN: 978-3-540-77977-3. (Reference on sorted-array vs. balanced tree trade-offs, segment tree fundamentals.)
3b. **Sedgewick, R. & Wayne, K.**, *Algorithms*, 4th ed., Addison-Wesley, 2011. ISBN: 978-0-321-57351-3. (General reference for amortised analysis, binary search, sweep-line.)

### Papers

4. **Bentley, J. L. & Ottmann, T.** (1979). "Algorithms for Reporting and Counting Geometric Intersections." *IEEE Transactions on Computers*, C-28(9), 643–647. DOI: 10.1109/TC.1979.1675432.
5. **Driscoll, J. R.; Sarnak, N.; Sleator, D. D.; Tarjan, R. E.** (1989). "Making Data Structures Persistent." *Journal of Computer and System Sciences*, 38(1), 86–124. DOI: 10.1016/0022-0000(89)90034-2.
6. **Pugh, William** (1990). "Skip Lists: A Probabilistic Alternative to Balanced Trees." *Communications of the ACM*, 33(6), 668–676. DOI: 10.1145/78973.78977.
7. **Munro, J. I.; Papadakis, T.; Sedgewick, R.** (1992). "Deterministic Skip Lists." *SODA '92*, pp. 367–375.
8. **Chambi, S.; Lemire, D.; Kaser, O.; Godin, R.** (2016). "Better bitmap performance with Roaring bitmaps." *Software: Practice and Experience*, 46(5), 709–719. DOI: 10.1002/spe.2325. arXiv: 1402.6407.
9. **Lemire, D.; Ssi-Yan-Kai, G.; Kaser, O.** (2016). "Consistently faster and smaller compressed bitmaps with Roaring." *Software: Practice and Experience*, 46(11), 1547–1569. DOI: 10.1002/spe.2402. arXiv: 1603.06549.
10. **Allen, J. F.** (1983). "Maintaining Knowledge about Temporal Intervals." *Communications of the ACM*, 26(11), 832–843. DOI: 10.1145/182.358434.
11. **Afshani, P.; Arge, L.; Larsen, K. G.** (2010). "Orthogonal range reporting in three and higher dimensions." *FOCS 2009*. Subsequent journal: 2017. (Output-sensitive lower bounds for stabbing/range reporting.)

### Documentation / Standards

12. **Bradner, S.** (1997). *RFC 2119: Key words for use in RFCs to Indicate Requirement Levels*. IETF. URL: https://datatracker.ietf.org/doc/html/rfc2119.
13. **Boost C++ Libraries** — *Boost.Icl: The Interval Container Library*, Joachim Faulhaber. URL: https://www.boost.org/libs/icl. Specifically the document *Addability, Subtractability and Aggregate on Overlap*: https://www.boost.org/doc/libs/1_84_0/libs/icl/doc/html/boost_icl/concepts/aggrovering.html.
14. **Apple Swift Documentation** — *Hashable Protocol*. URL: https://developer.apple.com/documentation/swift/hashable. (`Equatable`/`Hashable` contract used in §4.2.)
15. **Ruby Core Documentation** — `Hash`, `Object#eql?`, `Object#hash`. URL: https://docs.ruby-lang.org/en/3.3/Hash.html.
16. **Roaring Bitmap Format Specification**, RoaringBitmap consortium. URL: https://github.com/RoaringBitmap/RoaringFormatSpec. (4096 array/bitset threshold; run container layout.)
17. **Wikipedia: Bentley–Ottmann algorithm**. URL: https://en.wikipedia.org/wiki/Bentley%E2%80%93Ottmann_algorithm.
18. **Wikipedia: Allen's interval algebra**. URL: https://en.wikipedia.org/wiki/Allen%27s_interval_algebra.
19. **Wikipedia: Interval tree**. URL: https://en.wikipedia.org/wiki/Interval_tree.
20. **Wikipedia: Segment tree**. URL: https://en.wikipedia.org/wiki/Segment_tree.
21. **Wikipedia: Persistent data structure**. URL: https://en.wikipedia.org/wiki/Persistent_data_structure.
22. **MIT 6.046J Lecture 11 — Amortized Analysis** (Spring 2012). URL: https://ocw.mit.edu/courses/6-046j-design-and-analysis-of-algorithms-spring-2012/. (Potential method worked examples.)
23. **CMU 15-451 Lecture Notes — Amortized Analysis**, A. Blum & N. Bier. URL: https://www.cs.cmu.edu/afs/cs/academic/class/15451-s10/www/lectures/lect0203.pdf.
24. **Bowdoin CS 231 — Augmented Search Trees (CLRS 14)**. URL: https://tildesites.bowdoin.edu/~ltoma/teaching/cs231/spring14/Lectures/10-augmentedTrees/augtrees.pdf.

### Reference consumer

25. **ZMediumToMarkdown** (this org), file `lib/Parsers/MarkupStyleRender.rb` and `lib/Models/Paragraph.rb`. Concrete production source of the "what's active at index i?" workload that motivates this RFC.
26. **ZhgChgLi (2021).** *AVPlayer 實踐本地 Cache 功能大全* (AVPlayer/AVQueuePlayer with `AVURLAsset` and `AVAssetResourceLoaderDelegate`). Article on `medium.com/zrealm-ios-dev` (post id `6ce488898003`); reference open-source implementation: `github.com/ZhgChgLi/ZPlayerCacher`. The "non-contiguous Data merge" problem the article explicitly defers (`mergeDownloadedDataIfIsContinuted` only handles strictly contiguous tails) is the workload analysed in §1.3.1.

---

## §16  Review History

> _Reserved for professorial reviewer commentary. Each entry SHOULD record: reviewer initials, date, version of RFC reviewed, verdict (APPROVED / REVISIONS REQUESTED / REJECTED), and inline comments._

| Round | Reviewer | Date | RFC Version | Verdict | Comments |
|---|---|---|---|---|---|
| 1 | Algorithms Faculty (independent) | 2026-05-09 | v1.0 | REJECTED | 6 MUST-FIX, 6 SHOULD-FIX, 2 QUESTIONS |
| 2 | Algorithms Faculty (independent) | 2026-05-09 | v1.0 | APPROVED | All 6 MUST-FIX addressed; v1.0 finalised |
| 3 | Phase 2 教授 agent (Opus) | 2026-05-10 | v2.0 (this draft) | APPROVED with 1 MUST-FIX applied in-line | All 7 Phase-2 checks pass; Top 3 uncertainties resolved (I2.a–I2.d self-consistent; §6.13 worked-example arithmetic verified; Test #70 expected output computed and patched into §10.F). 1 MUST-FIX applied directly to Test #70 (RFC §10.F line 2288 area: associativity expected `R(A) = [(0, 4), (10, 10), (16, 20)]` plus full left/right derivations). 0 SHOULD-FIX blocking; minor wording polish noted in detailed report. Ready for Phase 3 6-language parallel implementation. |
| 4 | Phase 6 教授 agent (Opus 4.7, 1M context) | 2026-05-10 | v2.0 (post-implementation, post-conformance) | APPROVED (with 4 in-line RFC patches; 0 MUST-FIX blocking) | All 4 Phase-6 checks pass; cross-language consistency verified (8 method pairs across 6 languages, naming honors per-language idioms, semantics aligned); algorithmic equivalence confirmed via Phase-5 4973-line `cross_language.json` byte-identical conformance (200 ops + 126 probes + 20 set_ops + chain_ops associativity all green); v1 backward-compat preserved (Swift 38, Ruby 28, Python 117, Kotlin 30+, Go existing, JS 117 baseline tests all green). 4 in-line RFC patches applied: Test #65 worked example rewritten to single-element form with explicit `removeRanges` counter-example; §6.12 informative note re-targeted to per-element `remove`-loop equivalence with caveat paragraph; §9 case 37 sentinel-conflict resolution rule formalised (left-wins, SHOULD-warn, MUST-NOT-throw); §4.7 (C6) added documenting JS/TS `Number.MAX_SAFE_INTEGER` quantisation and fixture-coordinate guidance. All 6 implementations' deviations classified as ALLOWED (RFC scope) or SHOULD-FIX (v2.1 polish); zero MUST-FIX. Ready for v2 merge. See detailed report for naming-difference table and per-language deviation classification. |

### Round 1 — REJECTED (2026-05-09)

**Reviewer:** Algorithms Faculty (independent review)
**Judgment:** REJECTED
**Summary:** 6 MUST-FIX (Optional<Int> sentinel for cross-language byte-identical, Int.min underflow in §6.1 bsearch, Φ-potential proof algebra error, cleaner variant pseudocode incomplete, missing Lemma 4.5.X/Y, lazy build concurrency invariant); 6 SHOULD-FIX; 2 QUESTIONS-FOR-AUTHOR resolved as containment fast-path & Optional<Int>.

### Round 2 — APPROVED (2026-05-09)

**Reviewer:** Algorithms Faculty (independent review)
**Judgment:** APPROVED
**Summary:** all six MUST-FIX items are substantively addressed: (1) §4.7 (C4) Optional<Int> with `None == +∞` is elevated to MUST and the total order plus `succ` are fully defined; (C5) Int.min rules and Tests #23 / #23.A are added; (2) §6.1 step 4 is rewritten as `iv.hi + 1 >= lo` and (P5) explicitly cites the §4.7 (C4) succ semantics, underflow-free; (3) §7.2 adopts the two-term potential `α·M + β·TailSum` and explicitly walks the head-insert worst case; Lemma 7.2.A* corrects the amortised result from `O(log m_e)` to "balanced workload `O(log m_e)`, adversarial `Θ(m_e)`", and provides (R-B) BBST plus §14.10 order-statistic tree / Bentley-Saxe logarithmic method as RFC-internal normative-equivalent alternatives — reframing the issue as the inherent physical property of a sorted array (prepend necessarily Ω(m_e) shift), not an implementation flaw; (4) the §6.1 cleaner containment fast-path pseudocode covers idempotence / reverse-containment / multi-touch / first-insert corner cases; (5) Lemma 4.5.X (ruling out same-element same-coord open+close) and 4.5.Y (correctness of the sweep across elements at the same coord) have rigorous proofs; (6) §5.2 (I3.d) cache-write-path version re-check together with §11 (T3.M) Mutex / (T3.C) CAS patterns eliminate the stale-write race under the (T2) single-writer model. The newly added citations to Mehlhorn-Sanders, Sedgewick-Wayne, and Tarjan 1983 have been verified to exist.

NOTES: the sample code in the §12.1 Ruby and §12.2 Swift implementation hints still retains the `lo - 1` form without being updated in lockstep; §12 is informative while the §6.1 pseudocode is normative, and the author has used the footnote "simplified; see §6.1 cleaner variant" as guidance — acceptable, but reference implementations SHOULD use the underflow-safe form. The §5.2 event tuple `coord: Int` is an informative shorthand; the Optional<Int> in §4.7 (C4) and the §6.3 pseudocode is normative. v1.1 should harmonise the text.

### Round 3 — APPROVED (2026-05-10)

**Reviewer:** Phase 2 教授 agent (Opus 4.7, 1M context)
**RFC Version:** v2.0
**Judgment:** APPROVED (with 1 MUST-FIX applied in-line by reviewer to Test #70)
**Scope:** Removal API §6.6–§6.9, Set Operations API §6.10–§6.13, Empty-entry semantics §4.10, (I1.4) / (I2.a–d) tightening in §5.1, complexity bounds §7.6 / §7.7, edge-case table §9 rows #24–#37, test contracts §10.B–§10.G #21–#80, historical pointers in §13 / §14.1 / §14.9 with the §14.9.1 sym-diff inclusion amendment.

**Verdict per Phase-2-plan check (1–7):**

1. **(I1) and (I3) preservation across all 8 method pseudocodes — APPROVED.** §6.6 step 6 splice produces residuals with intra-entry gap `(end-start+1) ≥ 1` integer positions and inter-entry gap inherited from input (I1.2). §6.10 `merge_disjoint_lists` adjacency-collapse via `append_or_merge` is correct. §6.11 Lemma 6.11.A holds for both `i_b > i_a` and `j_b > j_a` (symmetric WLOG). §6.12 cut produces gap = `L_b[j].hi - L_b[j].lo + 2 ≥ 2`. §6.13 calls `merge_disjoint_lists` after the two `subtract`s (necessary, see worked example #2). All 4 mutating set-op forms invalidate `event_index` via the implied copy-then-swap into a fresh container; explicit invalidation present in §6.6–§6.9 step 8/4/3/5.

2. **Underflow / overflow safety dual to §6.1 (P5) — APPROVED.** §6.6 step 5: `start - 1` only when `iv.lo < start` (so `start > Int.min`); `end + 1` only when `end < iv.hi` (so `end < Int.max`). §6.12 `subtract_disjoint_lists`: `L_b[j].lo - 1` only when `L_b[j].lo > current_lo` (finite); `L_b[j].hi + 1` only when `L_b[j].hi < current_hi` (finite). Tests #30 (Int.min start) and #31 (Int.max end) verify the boundary cases. §4.7 (C4) Int.max close-coord handled via `Optional<Int>` `None` sentinel; §4.5.X Lemma rules out same-element open+close collision after split-residuals.

3. **Idempotence (no version bump on no-op) for all 8 ops — APPROVED.** §6.6 steps 2 and 4 quick-exit; §6.7 step 1; §6.8 step 1; §6.9 step 3 (`any_change == false`). §6.10–§6.13 mutating forms have explicit normative "MUST NOT bump when result == self structurally". Tests #21, #28, #33, #37, #42, #47, #49 cover the no-bump contracts.

4. **Insertion-order rule for 4 set ops aligned with §4.5 — APPROVED.** §6.10 union: walks `r_self.insertion_order` then tail-appends `keys(other) ∖ keys(self)` in `r_other.insertion_order` order. §6.11 intersect: walks `r_self.insertion_order`, eager-prunes empty, dense `ord` renumber. §6.12 difference: same as intersect. §6.13 sym-diff: step 1 walks `r_self.insertion_order` (covers shared keys with self's position); step 2 tail-appends other-only keys. All four match the §14.9.2 normative table. No implicit `Hash`/`Map` iteration order dependency anywhere.

5. **Cross-language reproducibility — APPROVED.** All 60 v2 tests in §10.B–§10.G have concrete expected outputs (after the Test #70 fix). Zero "implementation defined" loopholes in the v2 normative sections (the only `MAY differ` is in Test #77 with explicit "MUST post-normalise" requirement). Pseudocode uses language-neutral primitives (bsearch, two-pointer sweep) with explicit step numbering. §10.G stress tests #72–#80 cover the subtle cases where Hash iteration could leak.

6. **Eager-pruning × event_index invalidation — APPROVED.** §4.10 final paragraph explicitly normative: "Eager pruning is itself a mutation; per §5.2 (I3.c) it MUST invalidate event_index." All four removal pseudocodes have explicit `r.event_index := nil` (§6.6 step 8, §6.7 step 4, §6.8 step 3, §6.9 step 5). Set-op non-mutating forms produce fresh containers with `version = 0`, so cached `event_index.events[*].ord` cannot be stale (lazily rebuilt on first query with fresh `r.ord`).

7. **Sym-diff two algebraic forms produce identical insertion_order — APPROVED.** Reviewer manually traced four cases (incl. r1=[A], r2=[C,B,D] and r1=[B], r2=[A,B]) verifying both `(A∖B) ∪ (B∖A)` (§6.13) and `(A∪B) ∖ (A∩B)` (Test #77 alt-form) produce identical `insertion_order` when each constituent op follows the §14.9.2 rule. Test #77's "MAY differ" caveat is overcautious — for canonical pseudocode they always agree; the caveat exists only as a guard against sloppy alt-form implementations using Hash iteration in their union/intersect step. No conflict with §6.13.

**Verdict per Phase-1 Top-3 uncertainties:**

- **Top 3 #1 ((I2.a)–(I2.d) self-consistency) — APPROVED.** All 8 pseudocodes maintain (I2.a)–(I2.d). No conflict with §4.5 first-insert ordering (post-prune re-insert is logically a first-insert, documented in §4.10 deliberate side effect). v1 §6.1–§6.5 implicit reasoning (no remove ⇒ insertion_order ⊇ keys(intervals)) is now strengthened to equality and consistent with all v1 normative claims (which all assumed `R(e) ≠ ∅` for `e ∈ keys`).

- **Top 3 #2 (§6.13 worked example arithmetic) — APPROVED.** Reviewer manually re-derived both worked examples in §6.13: Example 1 (r1=[(0,5)], r2=[(2,8),(10,15)]) yields `a=[(0,1)]`, `b=[(6,8),(10,15)]`, merge=`[(0,1),(6,8),(10,15)]` — adjacency check `1+1=2 < 6` correct, sorted-concat happens to be no-op. Example 2 (r1=[(0,5)], r2=[(6,10)]) yields `a=[(0,5)]`, `b=[(6,10)]`, merge=`[(0,10)]` — adjacency `5+1=6 == lo of b` so `merge_disjoint_lists` is required (sorted-concat would violate (I1.2)). The two-example pedagogy (one no-op merge case + one truly-adjacent case) is appropriate; sharper single-example consolidation possible but not required.

- **Top 3 #3 (Test #70 associativity) — MUST-FIX APPLIED.** Reviewer manually computed both left-association `(r1 △ r2) △ r3` and right-association `r1 △ (r2 △ r3)` for `r1=[A:(0,10)], r2=[A:(5,15)], r3=[A:(10,20)]`. Both produce `R(A) = [(0,4), (10,10), (16,20)]`. Edited Test #70 in §10.F to add full expected output, both worked derivations, and a justification paragraph proving sym-diff associativity holds in closed-interval-with-integer-adjacency semantics (XOR of characteristic functions is associative; canonicalization is a deterministic function of the integer set).

**MUST-FIX summary (1 item, applied in-line by reviewer):**
- Test #70 expected output completed; both associativity derivations added; associativity-justification paragraph added.

**SHOULD-FIX (0 blocking; for future polish):**
- §6.10–§6.13 mutating-form pseudocode is implicit ("copy-then-swap"); Phase 3 implementations should ensure the post-swap `event_index` is `nil` (which falls out naturally if the fresh non-mutating result is constructed via `Rangeable.empty()` and then assigned). Not a spec defect; just a Phase 3 implementation note.
- §12.1 Ruby and §12.2 Swift sample code in informative §12 still uses `lo - 1` form (carried over from v1; already noted in Round 2 NOTES). Suggest harmonising to `iv.hi + 1 ≥ lo` form in a future v2.1 doc-only patch.

**Ready for Phase 3.** Spec density is sufficient for 6 language teams to produce byte-identical results. The eager-pruning + dense `ord` renumber rule is the load-bearing cross-language conformance contract; §10.G stress tests #72–#80 provide explicit fixture probes for it.

### Round 4 — APPROVED with 4 in-line RFC patches (2026-05-10)

**Reviewer:** Phase 6 教授 agent (Opus 4.7, 1M context)
**RFC Version:** v2.0 (post-Round-3, post-implementation, post-Phase-5 conformance)
**Judgment:** APPROVED with 4 in-line RFC patches; 0 MUST-FIX blocking; ready for v2 merge.
**Scope:** Cross-language consistency review of all 6 reference implementations (Swift, Ruby, Python, Kotlin, Go, JS/TS) for the v2 surface (Removal API §6.6–§6.9, Set Operations API §6.10–§6.13). Phase-5 4973-line `cross_language.json` byte-identical conformance result accepted as algorithmic equivalence proof; this round focuses on naming, idiom, deviation classification, RFC bug patches, and v1 backward-compat.

**Verdict per Phase-6-plan check (1–4):**

1. **Cross-language consistency — APPROVED.** All 8 v2 method pairs (4 removal + 4 set ops, each in mutating + non-mutating form for the latter) are present in every language. Naming honors per-language idioms and aligns with the language's own standard set library (Swift `SetAlgebra` lexicon, Ruby bang convention, Python `set` lexicon, Kotlin self-coined `*InPlace` suffix, Go PascalCase + `-ed` suffix, JS/TS Stage-4 `Set` proposal). The mutating-vs-non-mutating distinction is unambiguous in every language. Operator overloads conform to per-language conventions: Ruby `| & - ^`; Python `| & - ^` and `|= &= -= ^=`; Kotlin `+ - += -=` (note: Kotlin uses `+/-` instead of `|/-` because Kotlin operator-overloading whitelist permits `+/-` but not `|/&/^`; symmetric-difference therefore has no operator in Kotlin); Swift no operator overloads (intentional — `formUnion`/`union` vocabulary is the Swift convention). See "Naming Difference Table" in Phase-6 reviewer report.

2. **Algorithmic equivalence — APPROVED (Phase 5 evidence accepted).** Phase-5 conformance run was 6/6 byte-identical against `cross_language.json` (200 mixed inserts/removes ops + 126 probes covering subscript/transitions/getRange + 20 set_op cases each with 5–10 probes + chain-op associativity probes). No discrepancies reported on insertion-order, ord renumber, eager-prune, version-bump, or boundary close-coord behavior. Manual sanity check on the three two-pointer helpers (`merge_disjoint_lists`, `intersect_disjoint_lists`, `subtract_disjoint_lists`) across all 6 languages confirms `O(M_a + M_b)` complexity (no `O(M·N)` regression); reviewer verified `i, j` advance once per loop iteration in every implementation, no nested scan over `b` per element of `a`.

3. **v1 API surface backward-compat — APPROVED.** All 6 v1 baseline test suites are reported green post-v2: Swift 38/38, Ruby 28/28, Python 117/117, Kotlin 30+/30+, Go (existing test count) all passing, JS 117/117. v1 method signatures are unchanged: `insert / getRange / [i] / transitions / version / count / isEmpty / copy / iteration` are byte-compatible. v2 fixtures are a strict superset of v1 (record without `op` field defaults to `insert`); v1 `cross_language.json` continues to round-trip on every language (Phase-5 backward-compat smoke test).

4. **Per-language deviation classification — APPROVED.** All 14 documented per-language implementation-note deviations are within RFC scope (ALLOWED) or are non-blocking polish items (SHOULD-FIX for v2.1). Zero MUST-FIX. Highlights:
   * **ALLOWED (RFC scope):** Swift `storageEquivalent` post-construction comparison; Swift `adoptStorage` whole-storage replacement; Ruby `union(self, self)` going through `swap_with`; Ruby `structurally_equal?` O(M_out) overhead; Python `_populate` direct `_entries` write (bypasses public API but documented); Kotlin `MutationResult` sealed class; Kotlin `ord: var` (RFC permits dense renumber via in-place mutation); Go `pruneElement` slice-in-place mutate (idiomatic Go, semantically equivalent); JS `unionInPlace` extra `O(M)` dirty check (semantically equivalent, no observable behavior change).
   * **SHOULD-FIX (v2.1 polish):** (a) Python `__delitem__` / `__getitem__` API ambiguity (`del r[e]` vs `r[i]` overload pun); should add docstring caveat. (b) Kotlin `structurallyEquals` does not compare `ord` numerically (relies on iteration-order match); harmless because dense-renumber is deterministic from iteration order, but for clarity should add an `ord` assert in debug builds. (c) JS `_intervalListsEqual` is called multiple times per key inside `unionInPlace`; could be cached. (d) Go `intMaxSentinel` was already left-wins before this RFC patch — keep behavior, but the new §9 case 37 sentinel-conflict-resolution rule retroactively makes it normative across all 6 languages (JS Phase-3 report noted "mutating ops don't propagate"; should align to left-wins). (e) Kotlin currently makes `intMaxSentinel` an opt-in constructor param; document whether it propagates through set ops (currently does, via `Rangeable<E>(intMaxSentinel = ...)` in `union`/`intersect`/`difference`/`symmetricDifference`).
   * **MUST-FIX:** none. All 6 implementations conform to the v2 normative spec on every observable behavior covered by the test contract and the cross-language fixture.

**RFC patches applied in-line by reviewer (4 items):**

1. **§10.E Test #65 worked example rewritten.** Original example multi-element example (`r1.Strong=[(0,10)], r1.Italic=[(5,15)]`; `r2.Strong=[(3,6)], r2.Italic=[(12,18)]`) fails the stated `difference ≡ removeRanges-loop` equivalence: per-element `difference` produces `Italic=[(5,11)]`, but `removeRanges(3,6); removeRanges(12,18)` cross-pollutes (the second `removeRanges` cuts `[12,15]` out of Italic regardless of which key in `r2` the range came from). All 6 language teams independently flagged this. Test #65 is now a single-element worked example (`r1.Strong=[(0,10)], r2.Strong=[(3,6)]`) where the equivalence holds trivially; the multi-element scenario is preserved as an explicit counter-example showing why the original claim was wrong.

2. **§6.12 informative equivalence note re-targeted.** The note now correctly states `r.difference(other)` is structurally equivalent to the per-element `for e in keys(other): for ((lo, hi) in other.getRange(of: e)) r.remove(e, lo, hi)` reduction (per-key surgical erase via §6.6 `remove`, NOT cross-element erase via §6.9 `removeRanges`). A caveat paragraph explains the cross-element-vs-per-key distinction with the same counter-example.

3. **§9 case 37 sentinel-conflict resolution rule.** Added normative resolution: result inherits `self.intMaxSentinel` (left-wins, since set ops are not algebraically commutative on configuration metadata; `lhs ∪ rhs` and `rhs ∪ lhs` may differ on insertion-order anyway). When `self.intMaxSentinel != other.intMaxSentinel` and both are set, implementations SHOULD warn but MUST NOT throw. When `other.intMaxSentinel` is set but `self.intMaxSentinel` is unset, the result inherits `nil` / unset (no auto-promotion). Go has implemented this since Phase 3; the rule is now retroactively normative for the other 5 languages.

4. **§4.7 (C6) JS/TS i64-precision informative note.** Added: JS `Number.MAX_SAFE_INTEGER` is `2^53 - 1`; coordinates beyond ±2^53 MAY be silently quantised (Phase-4 fixture used the i64-ish value `4611686018427387903`, which `JSON.parse` quantised to `4611686018427388000` in JS — no observable effect on the fixture probes because the value was outside every probe range, but flagged for future fixture authors). Cross-language fixtures SHOULD constrain test coordinates to ±2^53. Boundary tests (#23, #23.A, #30, #31) remain normative and use language-specific sentinels.

**Naming Difference Table (per-language, 8 v2 methods, abbreviated):**

| Method | Swift | Ruby | Python | Kotlin | Go | JS/TS |
|---|---|---|---|---|---|---|
| remove `(e, start, end)` | `remove(_:start:end:)` | `remove(e, start:, end:)` | `remove(e, *, start, end)` | `remove(e, start, end)` | `Remove(e, start, end)` | `remove(e, {start, end})` |
| remove element | `remove(_:)` | `remove_element` (alias `delete`) | `remove_element`, `__delitem__` | `remove(e)` | `RemoveElement` | `removeElement` |
| clear | `removeAll` | `clear` (alias `remove_all`) | `clear` | `clear` | `Clear` | `clear` |
| removeRanges | `removeRanges(start:end:)` | `remove_ranges(start:, end:)` | `remove_ranges(*, start, end)` | `removeRanges` | `RemoveRanges` | `removeRanges({start, end})` |
| union (mut) | `formUnion` | `union!` | `update`, `\|=` | `unionInPlace`, `+=` | `Union` | `unionInPlace` |
| union (non-mut) | `union` | `union`, `\|` | `union`, `\|` | `union`, `+` | `Unioned` | `union` |
| intersect (mut) | `formIntersection` | `intersect!` (alias `intersection!`) | `intersection_update`, `&=` | `intersectInPlace` | `Intersect` | `intersectInPlace` |
| intersect (non-mut) | `intersection` | `intersect` (alias `intersection`), `&` | `intersection`, `&` | `intersect` | `Intersected` | `intersection` |
| difference (mut) | `subtract` | `difference!` (alias `subtract!`) | `difference_update`, `-=` | `differenceInPlace`, `-=` | `Subtract` | `subtractInPlace` |
| difference (non-mut) | `subtracting` | `difference` (alias `subtract`), `-` | `difference`, `-` | `difference`, `-` | `Subtracted` | `difference` |
| sym-diff (mut) | `formSymmetricDifference` | `symmetric_difference!` | `symmetric_difference_update`, `^=` | `symmetricDifferenceInPlace` | `SymmetricDifference` | `symmetricDifferenceInPlace` |
| sym-diff (non-mut) | `symmetricDifference` | `symmetric_difference`, `^` | `symmetric_difference`, `^` | `symmetricDifference` | `SymmetricallyDifferenced` | `symmetricDifference` |

All names align with the language's own standard set library convention. Semantically all 12 method pairs implement RFC §6.6–§6.13 byte-identically (Phase-5 confirmed).

**Final verdict.** APPROVED for v2 merge. 4 in-line RFC patches applied (Issues A/B/C from Phase-6 reviewer report). 0 MUST-FIX blocking. SHOULD-FIX items (5) deferred to v2.1 doc-only / polish patch. The Rangeable v2 specification, the 6 reference implementations, the cross-language fixture, and the conformance harness collectively meet the "algorithmic equivalence under per-language idiom" cross-language conformance bar set in §10.G.

