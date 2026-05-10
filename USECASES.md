# Rangeable Use Cases

`Rangeable<Element>` is a generic, integer-coordinate, closed-interval set container described in [`RFC.md`](RFC.md). It pairs hashable elements with their merged disjoint integer ranges and answers three queries efficiently: by-element (`get_range`), by-position (`r[i]`), and by-range (`transitions`). Same-element overlapping or integer-adjacent ranges merge automatically.

This document collects **seven scenarios**: six new ones that extend the high-level list in [RFC §1.1](RFC.md), plus a self-contained Ruby distillation of the [§1.3 ZMediumToMarkdown markup render](RFC.md) reference consumer (so the canonical example also runs end-to-end here). The §1.3.1 AVPlayer non-contiguous byte-range cache is already detailed in the spec and is not repeated here. Each scenario describes the problem, explains why `Rangeable` is the natural fit, and includes a runnable Ruby example using [`RubyRangeable`](https://github.com/ZhgChgLi/RubyRangeable) **v2.0+** (which adds the Removal and Set-Operations APIs introduced in [RFC §6.6 – §6.13](RFC.md)).

```sh
gem install rangeable    # 2.0.0+
```

## Scenarios

1. [Calendar Free-Time Finder & Conflict Detection](#1-calendar-free-time-finder--conflict-detection) — extends RFC §1.1 *calendar / resource scheduling*
2. [Genomic Region Annotation Overlap & Diff](#2-genomic-region-annotation-overlap--diff) — extends RFC §1.1 *genome annotation*
3. [Game Buff/Debuff Stack with Dispel + Combat Reset](#3-game-buffdebuff-stack-with-dispel--combat-reset) — extends RFC §1.1 *game engine status effects*
4. [Tax / Contract Effective-Date Composition](#4-tax--contract-effective-date-composition) — extends RFC §1.1 *tax / contract / regulatory effective dates*
5. [CIDR / IP Block Tenant Allocation](#5-cidr--ip-block-tenant-allocation) — new (network engineering)
6. [Syntax Highlighter Multi-Pass Combination](#6-syntax-highlighter-multi-pass-combination) — new (developer tools)
7. [ZMediumToMarkdown Markdown Render](#7-zmediumtomarkdown-markdown-render-reference-consumer) — distilled from RFC §1.3 (reference consumer)

---

## 1. Calendar Free-Time Finder & Conflict Detection

### Problem

You run a meeting-room booking system, an on-call rotation, or a personal calendar. Each day you need three answers:

1. **Free-time finder**: given a person's busy slots, what slots are free for a new meeting?
2. **Conflict detection**: given two people's busy slots, when do they overlap?
3. **Mutual availability**: given any number of people, when are *all* of them free at the same time?

The naive solution sorts intervals and runs a nested O(N²) sweep per pairwise comparison. Worse, it does not answer the "free time" question directly — that takes a separate pass that subtracts every busy interval from the day boundary.

### Why Rangeable

Calendars map cleanly onto `Rangeable<Element>`:

- **Element** = the calendar owner (person, room, on-call shift), or `:any` if you only care about the time axis.
- **Range** = `[start_minute, end_minute]` (or epoch second).
- **`difference(busy)`** computes free time in `O(M_total + M_busy)` instead of a per-day pass over every minute.
- **`intersect(alice, bob)`** computes their conflicting time, also `O(M_alice + M_bob)`.
- **`union(alice, bob, …).difference(total_day)`** gives mutual availability across N people, still linear in the total interval count.

A naive nested loop is `O(N²)`; `Rangeable`'s sweep-line implementation is `O(N)` per pairwise op (RFC §7.7).

### Ruby

```ruby
require 'rangeable'

# 24-hour day in minutes (0 = 00:00, 1439 = 23:59).
total_day = Rangeable.new.insert(:any, start: 0, end: 1439)

# Alice's busy slots.
busy_alice = Rangeable.new
busy_alice.insert(:any, start: 540, end: 600)   # 09:00–10:00
busy_alice.insert(:any, start: 780, end: 840)   # 13:00–14:00

# Free time = total day − busy.
free_alice = total_day.difference(busy_alice)
puts "Alice free slots: #{free_alice.get_range(:any).inspect}"

# Bob's busy.
busy_bob = Rangeable.new.insert(:any, start: 570, end: 720)  # 09:30–12:00

# Conflict detection = Alice ∩ Bob.
conflict = busy_alice.intersect(busy_bob)
puts "Alice/Bob conflict: #{conflict.get_range(:any).inspect}"

# Mutual free slots = total − (Alice ∪ Bob).
both_busy = busy_alice.union(busy_bob)
both_free = total_day.difference(both_busy)
puts "Mutual free slots: #{both_free.get_range(:any).inspect}"
```

### Expected output

```text
Alice free slots: [[0, 539], [601, 779], [841, 1439]]
Alice/Bob conflict: [[570, 600]]
Mutual free slots: [[0, 539], [721, 779], [841, 1439]]
```

---

## 2. Genomic Region Annotation Overlap & Diff

### Problem

Bioinformatics annotators (Ensembl, RefSeq, GENCODE) each publish their own version of where every gene, exon, and regulatory region sits on a chromosome. A research pipeline typically wants:

1. **Point query**: "Which genes / features cover base pair 41,200,000?"
2. **Agreement**: "Where do Ensembl and RefSeq agree on each gene's extent?"
3. **Diff**: "Where exactly do they disagree (different left edge, missing gene, …)?"

Naive solutions reach for `bedtools` or build interval trees per source plus custom diff logic. For a per-feature comparison the diff logic is non-trivial — it has to iterate features, look each one up in both sources, and split intervals at the disagreement boundaries.

### Why Rangeable

Each gene / feature is an **element**; its base-pair extent is a **range**. The query and diff operations map directly:

- **`subscript[bp_position]`** → the feature set covering that base pair (RFC §3.3).
- **`intersect(ensembl, refseq)`** → per-feature agreed-upon core in one call.
- **`symmetric_difference(ensembl, refseq)`** → per-feature edge disagreement, with empty-result features eagerly pruned (RFC §4.10), so you only see the genes that actually disagree.

The per-feature framing matters: a flat "all intervals" diff would conflate disagreements across genes; the element-keyed split keeps them separate.

### Ruby

```ruby
require 'rangeable'

# Two annotators provide gene region annotations on chromosome 17.
# Coordinates are base-pair positions on GRCh38.
ensembl = Rangeable.new
ensembl.insert("BRCA1", start: 41_196_311, end: 41_277_500)
ensembl.insert("TP53",  start: 7_661_779,  end: 7_687_550)

refseq = Rangeable.new
refseq.insert("BRCA1", start: 41_196_100, end: 41_277_500)  # earlier left edge
refseq.insert("TP53",  start: 7_661_779,  end: 7_687_550)   # exact agreement
refseq.insert("RAD51", start: 40_695_180, end: 40_732_376)  # extra gene

# Agreed core: regions where BOTH annotators agree.
agreed = ensembl.intersect(refseq)
agreed.each do |gene, ranges|
  puts "Agreed #{gene}: #{ranges.inspect}"
end

# Edge disagreement: where exactly the two sources differ.
disagree = ensembl.symmetric_difference(refseq)
disagree.each do |gene, ranges|
  puts "Disagree #{gene}: #{ranges.inspect}"
end

# Point query: which genes cover bp 41,200,000 in ensembl?
puts "Ensembl genes at bp 41,200,000: #{ensembl[41_200_000].objs.inspect}"
```

### Expected output

```text
Agreed BRCA1: [[41196311, 41277500]]
Agreed TP53: [[7661779, 7687550]]
Disagree BRCA1: [[41196100, 41196310]]
Disagree RAD51: [[40695180, 40732376]]
Ensembl genes at bp 41,200,000: ["BRCA1"]
```

Notice that `TP53` is absent from the disagreement output because the two sources agree exactly — the empty result is pruned automatically (RFC §4.10).

---

## 3. Game Buff/Debuff Stack with Dispel + Combat Reset

### Problem

A real-time game tracks dozens of buffs and debuffs on each player. The engine needs:

1. **Active query**: "Which effects are on the player at the current frame?" (UI icons, damage calc, AI input)
2. **Full dispel**: cleansing potion removes one effect entirely.
3. **Partial dispel**: a partial cleanse cancels the back half of a buff (e.g., halve the remaining duration).
4. **Combat reset**: when combat ends, drop everything in one operation.

A common implementation is `Hash<EffectId, [start_frame, end_frame]>` plus per-frame iteration; partial dispel becomes "compute the new range and overwrite", which is brittle when the same effect was applied twice and merged into a single span.

### Why Rangeable

- **Element** = effect id (`:haste`, `:poison`, …).
- **Range** = `[start_frame, end_frame]`.
- **`subscript[current_frame].objs`** → active effects ordered by first-application (RFC §4.5), useful for stable UI rendering.
- **`remove_element(:poison)`** → full dispel, one O(E) call.
- **`remove(:haste, start:, end:)`** → partial dispel; if the dispel range falls in the middle of an existing span the entry is automatically split (RFC §6.6).
- **`clear`** → combat reset.

Re-applying the same effect (`insert(:haste, …)` twice) automatically unions and the active-set query never sees duplicates.

### Ruby

```ruby
require 'rangeable'

# Buff durations measured in game frames (60 frames/sec).
player = Rangeable.new
player.insert(:haste,    start: 100, end: 600)   # 10 s of haste
player.insert(:poison,   start: 150, end: 300)   # 2.5 s of poison
player.insert(:shield,   start: 200, end: 500)   # 5 s of shield

puts "Frame 250 active: #{player[250].objs.inspect}"

# Cleansing potion: full dispel of poison.
player.remove_element(:poison)
puts "After full dispel of poison @ frame 250: #{player[250].objs.inspect}"

# Partial dispel: cancel the back half of haste (frames 400–600).
player.remove(:haste, start: 400, end: 600)
puts "After partial haste removal @ frame 450: #{player[450].objs.inspect}"
puts "Haste survives at: #{player.get_range(:haste).inspect}"

# Combat ends — reset all effects.
player.clear
puts "Post-combat active: #{player[250].objs.inspect}"
```

### Expected output

```text
Frame 250 active: [:haste, :poison, :shield]
After full dispel of poison @ frame 250: [:haste, :shield]
After partial haste removal @ frame 450: [:shield]
Haste survives at: [[100, 399]]
Post-combat active: []
```

The "active" list is in first-insert order (`:haste` before `:shield` because it was inserted first), which makes UI ordering stable across saves and restores (RFC §4.5).

---

## 4. Tax / Contract Effective-Date Composition

### Problem

Tax law, employment contracts, and regulatory rulebooks accumulate amendments over time. Each clause has an effective-date range; some are sunset, replaced, or restored. A compliance engine needs:

1. **Point-in-time query**: "Which clauses are in force on 2026-05-09?"
2. **Multi-jurisdiction composition**: "Combine federal + state + city clauses into one rulebook."
3. **Sunset**: "This clause expires 2027-01-01; remove its post-sunset coverage but keep its history."

A typical implementation uses a `WHERE start_date <= ? AND end_date >= ?` SQL filter, which works for a single jurisdiction but does not naturally compose across sources — you either UNION ALL into one query (which loses the per-clause structure) or run N queries and merge in code.

### Why Rangeable

- **Element** = clause id (`:fed_corp_tax_15pct`, `:ca_climate_levy`, …).
- **Range** = `[start_julian_day, end_julian_day]` (or `Date#to_jd` — any monotonic integer coordinate works; the RFC §4.7 axis is just `Int`).
- **`union(federal, state)`** → composite rulebook in one call.
- **`subscript[date_jd].objs`** → applicable clauses on that date, in the same first-insert order across runs.
- **`remove(clause, start: sunset_jd, end:)`** → sunset a clause's future without forgetting its past coverage.

### Ruby

```ruby
require 'rangeable'
require 'date'

# Convert YYYY-MM-DD to Julian day (a monotonic integer coordinate).
def jd(s) = Date.parse(s).jd

# Federal tax clauses with effective date ranges.
federal = Rangeable.new
federal.insert(:fed_corp_tax_15pct, start: jd("2020-01-01"), end: jd("2099-12-31"))
federal.insert(:fed_amt_repeal,     start: jd("2024-07-01"), end: jd("2099-12-31"))

# State-level overlay (e.g., California climate levy).
state = Rangeable.new
state.insert(:ca_climate_levy, start: jd("2023-01-01"), end: jd("2099-12-31"))

# Composite rulebook = federal ∪ state.
applicable = federal.union(state)
puts "Applicable on 2026-05-09: #{applicable[jd('2026-05-09')].objs.inspect}"

# CA legislature sunsets the climate levy starting 2027-01-01.
applicable.remove(:ca_climate_levy, start: jd("2027-01-01"), end: jd("2099-12-31"))
puts "Applicable on 2027-06-01: #{applicable[jd('2027-06-01')].objs.inspect}"

surviving = applicable.get_range(:ca_climate_levy)
puts "Climate levy survives (jd range): #{surviving.inspect}"
puts "  = #{Date.jd(surviving[0][0])} .. #{Date.jd(surviving[0][1])}"
```

### Expected output

```text
Applicable on 2026-05-09: [:fed_corp_tax_15pct, :fed_amt_repeal, :ca_climate_levy]
Applicable on 2027-06-01: [:fed_corp_tax_15pct, :fed_amt_repeal]
Climate levy survives (jd range): [[2459946, 2461406]]
  = 2023-01-01 .. 2026-12-31
```

The Julian day numbers (`2459946`, `2461406`) are the integer coordinates Rangeable stores; `Date.jd` converts back for display. Any monotonic integer encoding (epoch day, Unix second, …) works equally well.

---

## 5. CIDR / IP Block Tenant Allocation

### Problem

A multi-tenant cloud or corporate network divides one large IP pool (`10.0.0.0/16` = 65 536 addresses) into per-tenant blocks. Operators frequently need:

1. **Collision detection**: "Did anyone allocate overlapping CIDR blocks to two tenants?" (a serious misconfiguration).
2. **Free pool**: "Which IP runs are still unallocated?" (for new tenant requests).
3. **Tenant footprint**: "How many addresses does Alice currently hold?"

The standard approach is a hand-rolled interval tree or a brute-force `O(N²)` pairwise CIDR overlap check per tenant pair.

### Why Rangeable

Convert each CIDR block to its `[low_ip, high_ip]` integer range using `IPAddr#to_range`. Then:

- **Element** = tenant id (`:alice`, `:bob`, `:carol`).
- **Range** = `[low_ip_int, high_ip_int]`.
- **Collision detection**: walk `transitions` over the whole pool — whenever an `:open` event raises the active-set size above 1, two tenants share that coordinate.
- **Free pool**: `total_pool.difference(union_of_all_tenant_blocks)`; `Rangeable.difference` handles the per-tenant subtraction in `O(M_total + M_occupied)`.

### Ruby

```ruby
require 'rangeable'
require 'ipaddr'
require 'socket'

def cidr(s)
  block = IPAddr.new(s)
  range = block.to_range
  [range.first.to_i, range.last.to_i]
end

# Total network pool (e.g., a corporate /16).
pool_lo, pool_hi = cidr("10.0.0.0/16")
total_pool = Rangeable.new.insert(:pool, start: pool_lo, end: pool_hi)

# Tenant allocations.
a_lo, a_hi = cidr("10.0.0.0/24")
b_lo, b_hi = cidr("10.0.0.128/25")  # collides with alice
c_lo, c_hi = cidr("10.0.5.0/24")

allocs = Rangeable.new
allocs.insert(:alice, start: a_lo, end: a_hi)
allocs.insert(:bob,   start: b_lo, end: b_hi)
allocs.insert(:carol, start: c_lo, end: c_hi)

# Collision report: any coordinate where active set size > 1 is a conflict.
puts "--- Collision report ---"
active = []
allocs.transitions(over: pool_lo..pool_hi).each do |ev|
  if ev.kind == :open
    active << ev.element
    if active.size > 1
      ip = IPAddr.new(ev.coordinate, Socket::AF_INET).to_s
      puts "COLLISION at #{ip}: tenants #{active.inspect}"
    end
  else
    active.delete(ev.element)
  end
end

# Free pool = total − union(all tenants), keyed under :pool for difference symmetry.
occupied = Rangeable.new
allocs.each do |_tenant, ranges|
  ranges.each { |lo, hi| occupied.insert(:pool, start: lo, end: hi) }
end
free = total_pool.difference(occupied)
puts "--- Free IP runs ---"
free.get_range(:pool).each do |lo, hi|
  puts "  #{IPAddr.new(lo, Socket::AF_INET).to_s} .. #{IPAddr.new(hi, Socket::AF_INET).to_s}"
end
```

### Expected output

```text
--- Collision report ---
COLLISION at 10.0.0.128: tenants [:alice, :bob]
--- Free IP runs ---
  10.0.1.0 .. 10.0.4.255
  10.0.6.0 .. 10.0.255.255
```

Note the "different element keys for collision detection vs. same key for free-pool computation" idiom: collision detection requires distinct keys (so transitions surfaces both tenants at the overlap), while the free-pool subtraction merges all allocations under one key (so `difference` does set-arithmetic, not per-tenant accounting).

---

## 6. Syntax Highlighter Multi-Pass Combination

### Problem

A modern editor or CI lint runner executes several independent analysis passes on the same source:

- **Syntax pass**: parser warnings (missing semicolon, unused variable).
- **Type pass**: type-check info (inferred type hints, type mismatches).
- **Spell pass**: identifier spell-check (typos in variable names).

Each pass produces highlight ranges with a severity (`:info` / `:warn` / `:error`). The renderer needs to merge them, decide what color to paint each character, and emit a render loop that toggles ANSI / DOM color on every boundary event.

### Why Rangeable

This is the *generalised cousin* of the markdown markup render in [RFC §1.3](RFC.md). Same shape — multiple overlapping spans, rendered in nesting order — but the elements are severities instead of markdown tags:

- **Element** = severity (`:error`, `:warn`, `:info`).
- **Range** = `[start_col, end_col]`.
- **`union(syntax_pass, type_pass, spell_pass)`** → one combined highlight set.
- **`subscript[char_idx].objs`** → severities active at this character; the renderer picks the highest priority.
- **`transitions(over: visible_range)`** → boundary events for an event-driven render loop (toggle color on `:open`, restore on `:close`).

The first-insert ordering (RFC §4.5) means severities consistently sort the same way across runs, so a "highest-severity wins" tie-break is deterministic.

### Ruby

```ruby
require 'rangeable'

SOURCE = "let user_naem = getUserId() // typo and unused var"
#         0         1         2         3         4
#         0123456789012345678901234567890123456789012345678901

# Three independent lint passes on the same line.
syntax = Rangeable.new
syntax.insert(:warn, start: 4, end: 12)   # `user_naem` unused (warning)

types = Rangeable.new
types.insert(:info, start: 16, end: 26)   # inferred type hint on `getUserId()`

spell = Rangeable.new
spell.insert(:warn,  start: 9, end: 12)   # `naem` typo (also warn)
spell.insert(:error, start: 9, end: 12)   # spell-checker upgrades to error

# Combine all three passes via repeated union.
highlights = syntax.union(types).union(spell)

# What severities apply at column 10 (inside 'naem')?
puts "Col 10 severities: #{highlights[10].objs.inspect}"
puts "Col 20 severities: #{highlights[20].objs.inspect}"

# Render with ANSI colors using subscript per character.
ansi = { warn: "\e[33m", info: "\e[34m", error: "\e[31m" }
reset = "\e[0m"
priority = [:error, :warn, :info]

print "Rendered: "
(0...SOURCE.length).each do |i|
  active = highlights[i].objs
  if active.empty?
    print SOURCE[i]
  else
    severity = priority.find { |s| active.include?(s) }
    print ansi[severity] + SOURCE[i] + reset
  end
end
puts

# Boundary event log via transitions (the alternative render-loop approach).
puts "--- Boundary events ---"
highlights.transitions(over: 0..(SOURCE.length - 1)).each do |ev|
  puts "  coord #{ev.coordinate}: #{ev.kind} #{ev.element} (ord #{ev.ord})"
end
```

### Expected output

```text
Col 10 severities: [:warn, :error]
Col 20 severities: [:info]
Rendered: let <ansi-yellow>user_</ansi-yellow><ansi-red>naem</ansi-red> = <ansi-blue>getUserId()</ansi-blue> // typo and unused var
--- Boundary events ---
  coord 4: open warn (ord 1)
  coord 9: open error (ord 3)
  coord 13: close error (ord 3)
  coord 13: close warn (ord 1)
  coord 16: open info (ord 2)
  coord 27: close info (ord 2)
```

(Run the snippet in a terminal to see the actual ANSI-coloured `Rendered:` line.) Note the close-event ordering at coordinate 13: `error` closes before `warn` because RFC §4.5 sorts close events by `ord` *descending* (LIFO), matching markdown / HTML nesting expectations.

---

## 7. ZMediumToMarkdown Markdown Render (Reference Consumer)

### Problem

[`ZMediumToMarkdown`](https://github.com/ZhgChgLi/ZMediumToMarkdown) converts Medium articles' GraphQL JSON into portable Markdown. The hard part is the per-paragraph markup render:

1. **Same-type overlap**: Medium's API returns each user-applied STRONG (or EM, A, CODE, …) as an *independent* entry. Two overlapping bold actions — e.g. STRONG `[2, 5]` followed by STRONG `[3, 7]` — arrive as two separate API entries; the renderer must union them into one `[2, 7]` span before emitting markdown, otherwise the output gets two adjacent `**` markers that confuse the parser.
2. **Nested formatting**: `**bold _italic_ bold**` requires the renderer to know which markup opens first, which closes first, and to obey the LIFO close order.
3. **Boundary-driven render loop**: at each character the renderer needs to know which tags open or close so it can emit the right markdown delimiters.

The pre-`Rangeable` ZMediumToMarkdown implementation walked the markup list per character (`O(L · m)`); the v3.6.0 rewrite via `Rangeable.transitions` collapsed the inner loop and yielded a **2.23× end-to-end speedup** (per RFC §1.3 and ZMediumToMarkdown release notes).

### Why Rangeable

This is exactly the workload `Rangeable` was distilled from (RFC §1.3). The mapping is direct:

- **Element** = markup type (e.g., `:strong`, `:em`, `:code`).
- **Range** = `[start_char, end_char]` (inclusive — RFC §4.1).
- **`insert(:strong, …)` twice** — same-type overlapping (or integer-adjacent) ranges automatically union (RFC §4.3 adjacency rule).
- **`transitions(over: paragraph_range)`** → boundary event stream that drives the render loop. RFC §4.5 guarantees opens precede closes at the same coordinate, and closes are emitted in LIFO order by `ord` — exactly the markdown nesting rule.

### Ruby

```ruby
require 'rangeable'

# === Demo A: same-type overlap merging ===
# Medium's GraphQL returns user-applied STRONG markups as independent entries.
# Two overlapping STRONG ranges must render as a single bold span.
text_a = "the quick brown fox"
r = Rangeable.new
r.insert(:strong, start: 4,  end: 12)   # covers "quick bro"
r.insert(:strong, start: 10, end: 14)   # covers "brown"

puts "Merged STRONG range: #{r.get_range(:strong).inspect}"

# === Demo B: nested STRONG containing EM, driven by transitions ===
text_b = "The quick brown fox jumps"
r2 = Rangeable.new
r2.insert(:strong, start: 4,  end: 18)   # "quick brown fox"
r2.insert(:em,     start: 10, end: 14)   # "brown"

emit = { strong: ["**", "**"], em: ["_", "_"] }
output = +""
position = 0
r2.transitions(over: 0..(text_b.length - 1)).each do |ev|
  while position < ev.coordinate && position < text_b.length
    output << text_b[position]
    position += 1
  end
  output << emit[ev.element][ev.kind == :open ? 0 : 1]
end
output << text_b[position..] if position < text_b.length

puts "Rendered: #{output}"
```

### Expected output

```text
Merged STRONG range: [[4, 14]]
Rendered: The **quick _brown_ fox** jumps
```

The same render loop in production handles ESCAPE markups, link URLs, code spans, and Jekyll HTML escaping — see [`MarkupStyleRender.rb`](https://github.com/ZhgChgLi/ZMediumToMarkdown/blob/main/lib/Parsers/MarkupStyleRender.rb) for the full implementation.

---

## See Also

- [`RFC.md`](RFC.md) §1.1 — the generic problem class this document extends
- [`RFC.md`](RFC.md) §1.3 — ZMediumToMarkdown markup render (detailed reference consumer)
- [`RFC.md`](RFC.md) §1.3.1 — AVPlayer non-contiguous byte-range cache (secondary consumer)
- [`RFC.md`](RFC.md) §6.6 – §6.13 — normative pseudocode for the v2 Removal and Set-Operations APIs used above
- Six reference implementations: [Ruby](https://github.com/ZhgChgLi/RubyRangeable) · [Swift](https://github.com/ZhgChgLi/SwiftRangeable) · [Python](https://github.com/ZhgChgLi/PythonRangeable) · [TypeScript / JS](https://github.com/ZhgChgLi/JSRangeable) · [Kotlin](https://github.com/ZhgChgLi/KotlinRangeable) · [Go](https://github.com/ZhgChgLi/GoRangeable)
