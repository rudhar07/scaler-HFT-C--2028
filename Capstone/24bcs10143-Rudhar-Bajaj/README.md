# Capstone: Low-Latency Order Book

email: "rudhar.24bcs10143@sst.scaler.com"

roll_no: "10143"

Name: Rudhar Bajaj (individual submission)

This is a resting order book for one instrument with three different designs. All three run behind the same API, are checked against each other by one correctness test, and are timed by one benchmark program. They are profiled with Linux `perf` and cachegrind and with the macOS hardware counters.

| | Design | Total time, 1M ops |
|---|---|---|
| **A** | `std::map` price levels + `std::unordered_map` orders | 145.3 ms |
| **B** | binary heaps with lazy deletion + hashed levels + `std::map` orders | 151.3 ms |
| **C** | flat tick array + two-level occupancy bitmap + own open-addressing hash | **14.6 ms** |

---

## 1. Build and run

You need C++17 and the standard library only. The graded build uses `-O2`.

```bash
make                      # or: g++ -std=c++17 -O2 -Iinclude src/bench.cpp -o bench

./bench --verify          # correctness test: A, B and C must agree
./bench --impl A          # benchmark one implementation (5 runs, median)
./bench --impl B
./bench --impl C
./bench --impl all        # all three, one after another
```

Other options: `--runs N`, `--ops N`, `--seed N`.
- `--total-only` runs a single pass with no per-op timers; use it under `perf`.
- `--impl none` only generates the workload, as a baseline to subtract.

To get the Linux profile (perf stat, perf record and cachegrind for all three), run:
```bash
scripts/perf_linux.sh      # on Linux; writes results/
# on a Mac, inside Docker:
docker run --rm --platform linux/arm64 --privileged -v "$PWD":/ob -w /ob ubuntu:24.04 \
  bash -c 'apt-get update -qq && apt-get install -y -qq g++ make linux-tools-generic valgrind >/dev/null && scripts/perf_linux.sh'
```

**Files**

| Path | What |
|---|---|
| `include/common.hpp` | `Side`, `Level`, `BBO`, price ↔ tick conversion, band checks |
| `include/book_a.hpp`, `book_b.hpp`, `book_c.hpp` | the three implementations |
| `include/workload.hpp` | repeatable workload generator (mt19937, seed 42) |
| `src/bench.cpp` | benchmark, `--verify`, edge-case tests |
| `scripts/perf_linux.sh` | perf stat, perf record and cachegrind for A, B and C |
| `results/` | raw output of every run quoted in this README |

## 2. Interface changes

The API is the one from the brief: `add_order(id, side, double price, qty)`, `cancel_order`, `get_bbo`, `get_bids(depth=10)`, `get_asks(depth=10)`. The differences are:

- **Integer ticks inside.** Every book converts the price once on entry, with `tick = llround(price * 100)`, so 99.99 becomes 9999 and 100.01 becomes 10001. It stores only `int`s and converts back to `double` only when returning a `Level` or `BBO`. `llround` is used rather than a cast because `100.01 * 100` is `10000.9999…`.
- **Invalid input is ignored.** `add_order` returns `void`, so it silently drops these:
  - a price outside the side's band (bids 50.00–99.99, asks 100.01–150.00)
  - a price that is not a multiple of 0.01
  - a quantity ≤ 0
  - an id that is already in use

  The correctness test checks each of these cases.
- **`get_bbo()` is `const` in all three.** B achieves this by cleaning dead ticks off the heap top during `cancel_order` (see below).
- `BookC` has an optional constructor argument, the expected number of orders (default 2^20), used to pre-size its hash table.

## 3. The three designs

The designs differ in all four of the things the brief lists:

| | A | B | C |
|---|---|---|---|
| **Orders stored in** | `std::unordered_map<id, {side, tick, qty}>`, one heap node per order | `std::map<id, {side, tick, qty}>`, a red-black tree | own open-addressing table: a flat array of 16-byte `{id, tick, qty}` slots |
| **Price levels stored in** | `std::map<tick, qty>` per side, one tree node per level | `std::unordered_map<tick, {qty, in_heap}>` per side | `qty[15001]`, one `int64` per possible tick (120 KB, contiguous) |
| **Best bid/ask found by** | `begin()` of the map (bids use `std::greater`) | top of a `std::priority_queue` (max-heap for bids, min-heap for asks) | cached `best_bid_`/`best_ask_`; when that level empties, a `clz`/`ctz` scan of a two-level bitmap finds the next one |
| **Order found on cancel by** | hash lookup (chained buckets) | tree search, O(log N) | Fibonacci hash, then linear probing in one array |

### A: standard node containers
This is the textbook version. Each side is a sorted `std::map` from tick to total quantity, so the best level is `begin()` and depth is an in-order walk. Orders sit in an `unordered_map`. Every level and every order is a separately allocated node.

### B: heaps with lazy deletion
The levels sit in a hash map, which has no order, so a heap per side keeps track of the best price.
- A tick is pushed onto its heap only when it is not already there, which the `in_heap` flag tracks, so the heap never holds duplicates.
- When a level empties, its tick stays in the heap (lazy deletion).
- `cancel_order` keeps one rule true: the top of the heap is always a live level. Whenever a level empties, it pops dead ticks off the top. That keeps `get_bbo()` a plain `top()` read.
- `get_bids`/`get_asks` copy the heap and pop from the copy, skipping dead ticks. A heap cannot be walked in sorted order any other way.
- Orders are kept in an ordered `std::map`, a tree, rather than A's hash table, so finding an order to cancel works differently too.

### C: my own design (price-indexed array + bitmap + flat hash)
- **Levels:** prices are dense: there are only 10,000 valid ticks. So the level for tick *t* is simply `qty_[t]`, with no search and no allocation.
- **Finding the next level:** an occupancy bitmap has one bit per tick, and a 4-word summary marks which bitmap words are non-empty. "Next non-empty level below or above *t*" therefore takes at most two `clz`/`ctz` instructions plus a few word loads, whatever the number of levels.
- **Cached best:** `best_bid_`/`best_ask_` are updated with one comparison on add. They are recomputed only when the best level itself empties.
- **Shared bitmap:** bids and asks use one bitmap. The bands don't overlap, so a bid search walking down from ≤ 9999 can never reach an ask bit, and vice versa.
- **Orders:** my own hash table, one flat array of 16-byte slots holding `{id, tick, qty}`.
  - It uses linear probing with Fibonacci hashing (`id * 2^64/φ`, keeping the top bits).
  - The load factor stays ≤ 0.7.
  - Deletion uses backward shift, so there are no tombstones and probe chains never fill up with dead slots.
  - The side is not stored, because tick ≤ 9999 means a bid.
- **No allocation after construction**, unless the table has to grow.

### Complexity
*L* = live price levels on a side (≤ 5000), *N* = live orders, *H* = heap size including dead ticks, *d* = requested depth.

| Operation | A | B | C |
|---|---|---|---|
| `add_order` | O(log L) map + O(1) avg hash | O(log N) map + O(1) avg hash + O(log H) push (new level only) | O(1) avg hash + O(1) array + O(1) bitmap |
| `cancel_order` | O(1) avg hash + O(log L) map | O(log N) map + O(1) avg hash + amortised O(log H) pops | O(1) avg hash + O(1) array; next-best search O(1)* |
| `get_bbo` | O(1) (`begin()`) | O(1) (`top()`) | O(1) (cached) |
| `get_bids(d)` / `get_asks(d)` | O(d) | O(H + d log H) (copies the heap) | O(d)* |

\* The bitmap scan reads at most 4 summary words plus 2 bitmap words, a constant set by the 15,000-tick range.

## 4. Correctness

`./bench --verify` does two things.
1. **Edge cases** from the brief, run on each implementation:
   - an empty book gives `has_bid == has_ask == false`
   - cancelling an unknown id returns `false`
   - cancelling the same order twice returns `false` the second time
   - asking for 10 levels when only 3 exist returns 3
   - several orders at one price add up to one level quantity
   - the best level emptying moves the BBO to the next level
   - band edges (50.00, 100.01, 150.00) are accepted
   - out-of-band, off-grid, zero-quantity and duplicate-id orders are ignored
2. **Lockstep comparison.** The same workload runs through A, B and C side by side. After **every** op the three BBOs must match, and every cancel's return value must match. Top-10 bids and asks (price and quantity) are compared every 1–100 ops, and **every level** of both sides is compared periodically and at the end. Every result is compared pairwise (A–B, A–C, B–C), so a failure names the implementation that is wrong. Four workloads are used:
   - the benchmark workload itself (1M ops, seed 42)
   - a narrow-band, cancel-heavy workload (12 ticks per side), where the best level empties all the time
   - scrambled (non-sequential) order ids
   - a drain workload, where cancels outnumber adds and the book keeps running empty

I checked that the test can fail. Two deliberate bugs in C were caught: not moving `best_bid_` when its level empties, and a wrong backward-shift condition in the hash table. The verifier blamed only C for both. A bug in B's heap cleanup was caught too. The test also runs clean under `-fsanitize=address,undefined`.

Output ([results/verify.txt](results/verify.txt), macOS; [results/verify_linux.txt](results/verify_linux.txt) is identical on Linux):
```
Running correctness test...

Edge cases (empty book, unknown/double cancel, insufficient depth, invalid input):
  A: ok   B: ok   C: ok

Lockstep comparison of A, B and C on identical random workloads:
  benchmark workload (seed 42)        1010000 ops,  1010000 BBO +   20000 top-10 compares, 5000 bid levels left: agree
  narrow band, cancel-heavy (seed 7)   303000 ops,   303000 BBO +  303000 top-10 compares,   12 bid levels left: agree
  scrambled ids (seed 2026)            303000 ops,   303000 BBO +   33000 top-10 compares, 5000 bid levels left: agree
  drain to empty (seed 99)             202000 ops,   202000 BBO +  202000 top-10 compares,    2 bid levels left: agree

Implementation A: PASS
Implementation B: PASS
Implementation C: PASS

All implementations produce identical results.
```
The benchmark also prints a checksum of every value each book returned. All three print `fb65302f2675591e`, on macOS and on Linux.

## 5. Benchmark

**The workload follows the brief:**
- 1,000,000 ops generated with `std::mt19937(42)`: 70% add, 20% cancel, 10% `get_bbo`
- plus `get_bids(10)` + `get_asks(10)` every 100 ops
- side 50/50, quantity 1–100, price a uniform random valid tick in the side's band
- cancels always pick a currently live order: the generator keeps a list of live ids and removes each cancelled one by swapping it with the last element
- the realised mix is 700,471 add, 199,305 cancel, 100,224 BBO and 10,000 depth queries

All ops are generated before timing starts. The benchmark is templated on the book type, so there are no virtual calls in the timed loop. I wrote my own small range function on top of mt19937 instead of `std::uniform_int_distribution`, because libstdc++ and libc++ implement that differently, and the same seed would otherwise give a different workload on Linux and macOS.

**How it measures (each implementation, 1 warm-up and then 5 runs, median reported):**
1. **Total time:** one pass with only the book calls inside the loop.
2. **Per-op time:** a second pass on a fresh book, with each op wrapped in its own pair of `steady_clock::now()` calls and the time added to that op type's total. Its checksum must equal pass 1's.
3. **BBO block time:** `get_bbo()` is cheaper than the clock itself, so it is also timed as 1M back-to-back calls on the final book.

About the clock: on Apple Silicon, `steady_clock` advances in steps of about 41.7 ns (a 24 MHz counter), and a pair of `now()` calls costs about 29 ns. So a per-op figure near 15–40 ns is mostly clock cost. Only averages over many calls are meaningful, and the BBO block figure is the real cost.

**Machine:** Apple M5 Pro (15 cores), macOS, Apple clang 21, `-O2`. These are the numbers in the table. The same benchmark inside a Linux arm64 Docker VM is in [results/bench_linux.txt](results/bench_linux.txt).

[results/bench_macos.txt](results/bench_macos.txt):
```
Workload: 1000000 ops, seed 42 -> 700471 add, 199305 cancel, 100224 get_bbo, 10000 get_bids+get_asks(10)
steady_clock: resolution 41.0 ns, now()+now() pair 28.6 ns (included in every per-op number)

Implementation A (std::map + unordered_map)  (median of 5 runs)
  total time            :   145.26 ms
  ns / add_order        :    112.0
  ns / cancel_order     :    338.8
  ns / get_bbo          :     16.5   (block of 1M calls: 1.92 ns)
  ns / get_bids+asks(10):    147.0
  final BBO             : bid 99.99 x 2722 | ask 100.01 x 2515
  checksum              : fb65302f2675591e

Implementation B (heaps + lazy delete)  (median of 5 runs)
  total time            :   151.27 ms
  ns / add_order        :     41.4
  ns / cancel_order     :    495.8
  ns / get_bbo          :     15.3   (block of 1M calls: 2.91 ns)
  ns / get_bids+asks(10):   1108.3
  ...
Implementation C (tick array + bitmap + own hash)  (median of 5 runs)
  total time            :     14.59 ms
  ns / add_order        :     38.7
  ns / cancel_order     :     32.9
  ns / get_bbo          :     15.7   (block of 1M calls: 1.92 ns)
  ns / get_bids+asks(10):    106.6
  ...
```

## 6. Profiling

**Where the counters come from.** My only machine is a Mac. Linux `perf` was run in an Ubuntu 24.04 arm64 Docker container (kernel 6.12 linuxkit, perf 6.8). The macOS hypervisor gives the VM no hardware performance counters, so these `perf stat` counters show `<not supported>` there:
- cycles
- instructions
- branches
- branch-misses
- cache-misses
- L1-dcache-load-misses
- dTLB-load-misses

Only task-clock and page-faults work. To fill the gaps:
- **IPC** comes from the real M5 Pro counters, read natively with macOS `/usr/bin/time -l` ("instructions retired" and "cycles elapsed"); see [results/macos_time_l.txt](results/macos_time_l.txt).
- **Branch and cache misses** come from `valgrind --tool=cachegrind --cache-sim=yes --branch-sim=yes`, with the caches set to an Apple P-core's sizes (128 KB 8-way L1D, 16 MB last level). These are *simulated*, but the instruction counts are exact; see [results/cachegrind.txt](results/cachegrind.txt).
- **dTLB misses** are not available from either source.

Every counter run uses `--runs 1 --total-only`: exactly one pass over the workload, with no per-op timers that would distort the counts. Each run is repeated as `--impl none`, which only generates the workload, and that baseline is subtracted. So every "net" number below is the order book alone.

### perf stat ([results/perf_stat.txt](results/perf_stat.txt))
```
perf stat -r 5 -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults ./bench --impl X --runs 1 --total-only

impl none : 24.02 msec task-clock (+-14.6%)   802 page-faults    (other 7 counters: <not supported>)
impl A    : 164.46 msec task-clock (+-4.1%)  7211 page-faults    (other 7 counters: <not supported>)
impl B    : 239.54 msec task-clock (+-12.0%) 9024 page-faults    (other 7 counters: <not supported>)
impl C    : 43.93 msec task-clock (+-6.3%)    935 page-faults    (other 7 counters: <not supported>)
```

### perf record -g (flat self-time; [results/perf_report_A.txt](results/perf_report_A.txt), [B](results/perf_report_B.txt), [C](results/perf_report_C.txt))
This was sampled on `cpu-clock` at 20 kHz, with frame pointers (`make bench_prof`). Most container code is inlined into the `Book::` functions.

| A | B | C |
|---|---|---|
| 33.0% `BookA::add_order` (map + hash insert, inlined) | **50.0% `BookB::cancel_order`** | 32.5% `BookC::add_order` |
| 28.2% `BookA::cancel_order` | 10.5% `BookB::add_order` | **16.3% kernel `clear_page`** (zeroing fresh pages) |
| 7.5% `unordered_map::_M_rehash` | 4.5% + 4.5% + 3.6% red-black-tree insert/erase/rebalance | 16.2% `run_once` (the loop, with `get_bbo` inlined) |
| **≈ 20% `malloc`/`free`/`malloc_consolidate`** | ≈ 11% `malloc`/`free` | 10.5% + 6% workload generation, mt19937 |
| 2.1% workload generation | 2.7% `memcpy` (heap copies in `get_bids`) | 5.6% `BookC::cancel_order`, 2.9% `llround` |

## 7. Results table

Numbers marked "net" have the `--impl none` baseline subtracted. "Per op" divides by the 1,010,000 calls in the workload.

| Metric | A (map + hash) | B (heaps) | C (array + bitmap + own hash) |
|---|---|---|---|
| Total time (ms), macOS median of 5 | 145.3 | 151.3 | **14.6** |
| ns / add (per-op timer) | 112.0 | 41.4 | **38.7** |
| ns / cancel (per-op timer) | 338.8 | 495.8 | **32.9** |
| ns / BBO, per-op timer (block of 1M) | 16.5 (1.92) | 15.3 (2.91) | 15.7 (**1.92**) |
| ns / get_bids+get_asks(10) | 147.0 | 1108.3 | **106.6** |
| Instructions, net (cachegrind, exact) | 502.8 M (498 / op) | 710.1 M (703 / op) | **119.8 M (119 / op)** |
| IPC, whole process (M5 Pro hardware counters) | 1.28 | 1.44 | **1.76** |
| IPC, net of baseline (M5 Pro hardware counters) | 1.09 | 1.29 | 1.09 |
| Branch misses, net (cachegrind, simulated) | **8.90 M of 87.3 M = 10.2%** | 6.44 M of 133.4 M = 4.8% | 0.80 M of 15.8 M = 5.1% |
| L1-dcache-load misses, net (cachegrind, simulated) | 10.19 M (10.1 / op) | 9.83 M (9.7 / op) | **2.17 M (2.2 / op)** |
| Cache misses = last-level D misses, net (cachegrind) | 2.22 M | 1.87 M | **1.40 M** |
| dTLB-load-misses | not supported | not supported | not supported |
| Page faults, net (Linux `perf stat`) | 6,409 | **8,222** | 133 |
| Minor page faults, net (macOS `time -l`) | 2,078 | 2,045 | 1,037 |
| Linux task-clock, net (ms) | 140.4 | 215.5 | **19.9** |

## 8. Performance analysis

### 8.1 Which is fastest?
**C is the fastest at every operation.** Its total time is 10× lower than A's and B's: 14.6 ms against 145–151 ms. That is about 14 ns per op averaged over the whole workload.
- **add:** C at 38.7 ns and B at 41.4 ns are close. Both are mostly clock cost: the per-op timer adds about 29 ns and has 41 ns resolution. A is clearly slower at 112 ns, because each add can allocate a tree node and always allocates a hash node, and its `unordered_map` rehashes as it grows (7.5% of A's samples).
- **cancel:** this is where the designs separate most. C takes 33 ns, A 339 ns and B 496 ns. A cancel hits a *random* old order, so its data is almost never in cache.
  - **C:** one hash probe into a flat array, one array decrement and, rarely, a bitmap scan.
  - **A:** a hash-bucket chain walk, then a ~12-level tree search for the price, then two `free`s.
  - **B:** a ~19-level red-black-tree search over about 500k order nodes, then rebalancing, then a hash lookup for the level, then heap pops when a level empties.
- **get_bbo:** all three are O(1). Timed as a block of 1M calls, A and C take 1.9 ns and B 2.9 ns; the per-call ~15 ns is just the clock. A reads `begin()` of the map, which is a cached pointer. C reads two cached ints and two array cells. B's `top()` is cheap too, but it then needs a hash lookup to find the level's quantity.
- **depth:** C (107 ns for both sides) walks the bitmap, about 1 `ctz` per level. A (147 ns) walks 10 tree nodes. B (1.1 µs) has to **copy its whole heap**, which holds thousands of entries including dead ticks, and then pop 10 times. That is the cost of using a structure with no sorted iteration.

C wins because it does much less work per op: **119 instructions per op, against ~500 for A and ~700 for B (cachegrind)**. Those instructions also touch far fewer cache lines: 2.2 L1 misses per op, against ~10.

### 8.2 Branch prediction
**A has the most branch misses: 10.2% of its branches, 8.9 M in total, about 9 per op.** Most of them come from the tree searches. Prices are uniformly random over 5000 ticks, so at each node of the red-black tree "go left or right?" is close to a coin flip. A 12-level search therefore costs several mispredictions, and there's another one at the end of the hash-bucket chain walk. B's *rate* is lower (4.8%) only because it executes many more easy, predictable branches, such as heap and loop bookkeeping. In absolute terms B still mispredicts 6.4 M times, for the same reason: its 19-level order tree is searched with random ids. C has just 0.8 M mispredicts, under 1 per op. It has almost no data-dependent branches: an array index replaces the price search, a single hash probe usually hits on the first slot at load ≤ 0.7, and `clz`/`ctz` replace the loops that search for the next level.

### 8.3 Cache behaviour
**A and B have the most cache misses: about 10 L1D misses per op, against 2.2 for C.**
- **A and B follow pointers and allocate many small objects:**
  - A tree node or hash node for every level and every order, about 48–64 bytes each.
  - Hundreds of thousands of separate `malloc` blocks scattered over the heap.
  - Each step of a tree search is a dependent load from an unpredictable address, so the CPU cannot prefetch it and each step can miss.
  - `malloc`/`free` themselves account for ~20% of A's profile and ~11% of B's.
- **C accesses contiguous memory:**
  - The level array is 120 KB and the bitmap 2 KB. Both fit in the M5's 128 KB L1D plus L2, and stay hot.
  - The only cold access is the hash slot for the order, roughly one miss per add or cancel.
  - That one miss is C's main remaining cost. Fibonacci hashing deliberately scatters ids across the 16 MB table, so consecutive adds write to random cache lines.
  - This shows in C's last-level misses: 1.4 M, which is lower than A and B but not 5× lower.

IPC fits this picture. Whole-process IPC is highest for C (1.76), because its book work is short and the rest is the cheap generator. Net of the generator, C's IPC is about 1.1, the same as A's. C executes 4–6× fewer instructions, but the ones left are dominated by the cache-missing hash access. C is faster because it does far less work, not because the CPU runs its instructions faster.

### 8.4 Page faults
**B causes the most page faults (8,222 net on Linux), with A close behind (6,409).** Both grow by allocating nodes:
- B's `std::map` of orders allocates a node per order.
- Its hashed levels and its two heaps grow by reallocating, and `get_bids` makes a fresh copy of the heap on every call.
- A allocates a node per order and per level, and rehashes its order table several times as it grows.

Every new page that `malloc` takes from the kernel is a page fault. **C has only 133 net faults.** It allocates its 16 MB table and 120 KB array once, in the constructor, and never again. Linux most likely backs that large allocation with transparent huge pages, so 16 MB costs only a handful of faults, and the kernel zeroes it up front, which is the 16% `clear_page` in C's profile. In a real system this cost would be paid at start-up, not on the trading path. On macOS, which doesn't use huge pages here, C's 1,037 faults are about 16 MB / 16 KB pages: exactly the table being touched once. The lesson is that C's memory use is fixed and allocated up front, while A's and B's grows with the number of orders.

### 8.5 Production choice: C
I would deploy **C**, and not only because its average is lowest:
- **Latency:** it is fastest on every operation, with cancel 10–15× faster than A or B.
- **Predictability:** it never allocates after start-up, so `malloc` can't stall, and its operations are O(1) with tiny constants. A's and B's latency depends on tree depth and allocator state. B has an especially bad outlier: `get_bids` copies the whole heap, so its cost grows with how many dead ticks have built up.
- **Memory:** fixed and known: 16 MB of order slots for 1M orders plus 120 KB of levels. A and B use a little less at this size but grow without bound, and their nodes are fragmented.
- **Cache behaviour:** all level data is hot and contiguous, with about one cold miss per order operation.
- **Complexity and maintainability:** this is C's real cost. It has about 250 lines of custom code (the bitmap scans and the backward-shift deletion) where A has about 80 lines of standard containers. It also hard-codes the tick range, so a new price band means re-sizing it. The `--verify` lockstep test, with A as the simple reference, is what makes that acceptable: A is easy to trust, and C is checked against it on every op.

### 8.6 What I would try next
- **Index orders directly by id.** Exchange ids are usually sequential, so a slab `orders[id - base]`, with a hash only as a fallback, would turn C's one random hash miss into a mostly sequential write.
- **Smaller order slots:** a 16-bit tick offset and a 32-bit quantity with a 64-bit id give 12-byte slots, or 8 bytes with 32-bit ids. That means more orders per cache line.
- **Pre-fault the table at start-up** (`MAP_POPULATE`, or touch every page) and use huge pages explicitly, to remove page faults and TLB misses from the hot path.
- **An integer-tick API:** `llround` alone is 2.9% of C's profile.
- **Measure tail latency** (p99 and p99.9) with a cycle counter instead of `steady_clock`, which is too coarse at 41 ns on this machine. Tails, not averages, are where A's and B's allocations would show up most.
- **Fix B's depth query** by keeping a small sorted cache of the top levels instead of copying the heap. B's add is already good.
- **A per-level FIFO** (an intrusive list in the order slab) if this book is ever extended to matching.
