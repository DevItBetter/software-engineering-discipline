# Eviction Policies

Reference for choosing what to evict when the cache is full. Sizing the cache is half the problem; eviction is the other half.

## The core trade-off

An eviction policy approximates "evict the entry least likely to be requested again." No policy is optimal for every workload — the optimal (Belady) policy requires future knowledge. Real policies are heuristics tuned to access-pattern shapes.

## LRU — Least Recently Used

Default in Memcached, Guava, `functools.lru_cache`, and most introductory caching libraries.

**The idea:** evict the entry whose last access was longest ago.
**Implementation:** doubly-linked list + hash map. O(1) lookup and eviction.
**Strength:** simple; decent on workloads with **temporal locality** (recent items are likely to be re-accessed).
**Weakness:** vulnerable to **scan workloads** — a one-time pass over many cold items evicts the entire hot set. Also weak when popularity shifts slowly (LRU adapts only via repeated re-access).

## LFU — Least Frequently Used

**The idea:** evict the entry with the lowest access count.
**Strength:** better than LRU when there's a **stable hot set** and the working set fits in cache.
**Weakness:** "old hot key" problem — yesterday's hits keep keys alive forever. Pure LFU never adapts to popularity shifts. Aged/decaying LFU mitigates by halving counters periodically.

## TinyLFU and W-TinyLFU

Einziger, Friedman, Manes. *TinyLFU: A Highly Efficient Cache Admission Policy*. ACM Transactions on Storage 13(4):35, 2017 (preprint arXiv:1512.00727).

**The idea:** maintain an **admission filter** using a count-min sketch (~4 bits per estimate, ~8 bytes per cache entry). When the cache is full, a new candidate is **admitted** only if its estimated frequency exceeds that of the eviction candidate. Otherwise the candidate is rejected.

**W-TinyLFU** prepends a small windowed LRU (default ~1% of the cache, adapted at runtime via hill climbing in Caffeine) to absorb recency bursts that pure frequency-admission would reject. New items go to the window; if accessed again there, they're promoted into the main LFU-admitted region.

**Strengths:**
- Near-optimal hit rate; competitive with ARC and LIRS across diverse workloads.
- Constant memory overhead from the sketch.
- Doesn't keep ghost entries (evicted-and-tracked items) — the sketch is the memory.

**Implementations:**
- **Caffeine** (Java) — the reference implementation. Replaces Guava's cache as the JVM default.
- **Ristretto** (Go) — Dgraph Labs' implementation; popular in Go services.
- Increasingly common in CDN and edge caches.

For new caches at any scale, W-TinyLFU is the modern default. Caffeine's wiki notes it provides "near optimal hit rate" and is "competitive with ARC and LIRS while not retaining evicted (ghost) keys."

## ARC — Adaptive Replacement Cache

Megiddo & Modha. *ARC: A Self-Tuning, Low Overhead Replacement Cache*. USENIX FAST '03 (March 2003), IBM Almaden.

**The idea:** maintain two LRU lists — one for entries seen once (recency), one for entries seen multiple times (frequency) — plus two ghost lists (recently evicted from each). A target ratio between the two adapts based on which ghost list a re-access hits. Self-tuning between recency and frequency without manual configuration.

**Strengths:** consistently outperforms LRU; adapts to workload shape; well-studied.
**Weaknesses:** **patented by IBM (US 6,996,676 / 7,058,766)**. The patent is widely reported to have expired around 2024 but verify current status before committing in regulated contexts. The patent history is why Postgres briefly used ARC in 8.0 then reverted; ZFS still uses a modified ARC.

In 2026, TinyLFU and S3-FIFO often perform comparably without the legacy patent baggage. ARC remains a strong choice and is the default in some storage systems, but it is no longer the obvious choice for new code.

## S3-FIFO

Yang, Zhang, Qiu, Yue, Vinayak, Zhang, Yang, Zheng. *FIFO Queues are All You Need for Cache Eviction*. SOSP 2023 (DOI: 10.1145/3600006.3613147).

**The idea:** three FIFO queues — small (~10%), main (~90%), and ghost (history of evictions). New items enter the small queue; if not re-accessed there, they're evicted before reaching main ("quick demotion"). Re-accessed items in small are promoted to main. Items evicted from main go to ghost; re-access from ghost re-admits to main.

**Strengths:**
- FIFO operations are simpler than LRU's linked-list reordering — much higher throughput at multi-thread scale (the paper reports ~6× over LRU at 16 threads).
- Beats LRU on miss ratio across most of 6,594 production traces tested.
- The "quick demotion" of one-shot items is the key insight: most items are accessed only once, and LRU wastes effort tracking them.

**Status:** the new entrant. Worth tracking; not yet the default in major caching libraries but adoption is growing.

## FIFO and Random

- **FIFO (First In First Out)** — usually worse than LRU on standard workloads. Notable mainly because it's simpler.
- **Random** — surprisingly competitive on some workloads. The CLOCK family of algorithms (used in OS page caches) and S3-FIFO show that FIFO-with-tweaks can outperform LRU.

For most application caches, neither pure FIFO nor pure random is the right choice. They're useful baselines for understanding why the more sophisticated policies exist.

## Sizing heuristics

Eviction policy choice depends on cache size relative to working set:

- **Cache much larger than working set:** eviction policy doesn't matter; almost nothing gets evicted. Pick LRU for simplicity.
- **Cache somewhat smaller than working set:** policy matters most. W-TinyLFU's admission filter shines. ARC and S3-FIFO are also strong.
- **Cache much smaller than working set:** every miss is bad; admission policy matters more than eviction policy. W-TinyLFU's "don't even admit a one-shot key" is exactly right here.

Diagnosing the regime: track hit ratio and eviction rate. Hit ratio approaching 100% with low eviction rate = oversized; fine. Hit ratio low with high eviction rate = undersized or wrong policy.

## When eviction is the wrong knob

- **Working set has shifted.** No eviction policy fixes a cache that's permanently mis-sized for the current workload. Resize.
- **Key cardinality is exploding.** A bug elsewhere is generating per-request unique keys. Fix the bug; eviction is treating the symptom.
- **The cache is the wrong layer.** A bigger LRU at the application level may be the wrong move when a CDN-layer cache or a database materialized view would be the right one. Step back.

## Choosing — defensible defaults

- **New code, no specific knowledge:** LRU. Predictable, well-understood, every library has it.
- **Production scale, performance-sensitive:** W-TinyLFU (Caffeine in JVM, Ristretto in Go).
- **Storage / page cache:** ARC if patent status confirmed for your jurisdiction; otherwise S3-FIFO or W-TinyLFU.
- **High-throughput multi-threaded:** S3-FIFO's lock-free FIFO operations.
- **Workload that fits the cache:** LRU is fine; the policy doesn't matter much.

## Sources

- Einziger, Friedman, Manes. *TinyLFU: A Highly Efficient Cache Admission Policy*. ACM TOS 13(4):35, 2017. `https://arxiv.org/abs/1512.00727`.
- Caffeine wiki on efficiency. `https://github.com/ben-manes/caffeine/wiki/Efficiency`.
- Megiddo & Modha. *ARC: A Self-Tuning, Low Overhead Replacement Cache*. USENIX FAST '03. `https://www.usenix.org/conference/fast-03/arc-self-tuning-low-overhead-replacement-cache`.
- Yang et al. *FIFO Queues are All You Need for Cache Eviction*. SOSP 2023. `https://dl.acm.org/doi/10.1145/3600006.3613147`.
- Wikipedia, *Adaptive replacement cache* (for the IBM patent context). `https://en.wikipedia.org/wiki/Adaptive_replacement_cache`.
