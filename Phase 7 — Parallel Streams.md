# Phase 7 — Parallel Streams

## How Parallel Streams Work Internally

When you call `.parallelStream()` or `.parallel()` on an existing stream, the framework:

1. **Splits** the source into subranges using `Spliterator`
2. **Submits** each subrange as a task to `ForkJoinPool.commonPool()`
3. **Processes** each partition independently
4. **Combines** partial results back into a final result

```java
List<Integer> nums = List.of(1,2,3,4,5,6,7,8);

int sum = nums.parallelStream()
              .mapToInt(Integer::intValue)
              .sum();
```

Internally this becomes something like:

```
commonPool (4 threads on a 4-core machine):

Thread-1: [1,2]  → sum=3
Thread-2: [3,4]  → sum=7
Thread-3: [5,6]  → sum=11
Thread-4: [7,8]  → sum=15

Combine:  3+7+11+15 = 36
```

---

## `ForkJoinPool.commonPool()`

The default pool shared across the entire JVM:

```java
System.out.println(
    ForkJoinPool.commonPool().getParallelism()
);
// defaults to Runtime.getRuntime().availableProcessors() - 1
// on a 4-core machine: 3
```

**Critical implication:** Every parallel stream in your JVM shares this pool. A long-running parallel stream in one service can starve another. In a Spring Boot app with multiple parallel stream usages, this matters.

---

## Converting Sequential ↔ Parallel

```java
// Sequential → Parallel
list.stream().parallel()

// Parallel → Sequential
list.parallelStream().sequential()

// Check if parallel
stream.isParallel()
```

The last call to `parallel()` or `sequential()` in a pipeline wins — it applies to the entire pipeline.

```java
list.parallelStream()
    .filter(...)     // parallel
    .sequential()    // switches whole pipeline back to sequential
    .collect(...);
```

---

## When Parallel Wins

Parallel streams beat sequential when ALL of these hold:

### 1. Large dataset
There's a fixed overhead for splitting, task submission, and combining. For small datasets, this overhead exceeds the gain.

```
Rule of thumb: < 10,000 elements — don't bother with parallel.
Above 100,000 CPU-bound elements — parallel likely wins.
```

### 2. CPU-bound operations
Parallel streams use CPU threads. If the operation is I/O-bound (network, DB, disk), threads block and you get no parallelism benefit — you just exhaust the thread pool.

```java
// CPU-bound — good candidate
list.parallelStream()
    .map(item -> heavyCryptoHash(item))  // pure computation
    .collect(Collectors.toList());

// I/O-bound — wrong tool, use CompletableFuture instead
list.parallelStream()
    .map(item -> restClient.fetch(item)) // blocks thread on network
    .collect(Collectors.toList());
```

### 3. No shared mutable state
Each partition must operate independently. Shared mutation = race conditions.

```java
// Race condition — DO NOT DO THIS
List<String> result = new ArrayList<>();
list.parallelStream()
    .filter(s -> s.length() > 3)
    .forEach(result::add);  // ArrayList is not thread-safe
// result will be corrupt or miss elements
```

Fix: use `collect` — it creates one container per partition safely.

### 4. Encounter order doesn't matter
If order of results is irrelevant, parallel streams can skip reordering work.

---

## When Parallel Backfires — Know All Cases

### Small datasets
```java
// Sequential is faster — ForkJoin overhead dominates
List.of("a","b","c").parallelStream().map(String::toUpperCase).toList();
```

### `sorted()` in parallel
`sorted()` is a stateful operation — it must collect **all elements** from all partitions before sorting. This creates a synchronization barrier that eliminates most parallelism benefit.

```java
// sorted() forces full materialization before proceeding
list.parallelStream()
    .filter(...)
    .sorted()      // kills parallelism here — full barrier
    .limit(10)
    .collect(...);
```

### `distinct()` in parallel
Same problem — needs a shared set across partitions to track seen elements. Expensive synchronization.

### `limit()` in parallel on unordered sources
On an ordered source, parallel `limit(n)` must preserve the first `n` elements in encounter order — requires coordination between partitions.

```java
// Forces encounter-order preservation — slow in parallel
list.parallelStream()
    .limit(10)
    .collect(Collectors.toList());

// Add unordered() hint — allows any 10 elements, much faster
list.parallelStream()
    .unordered()
    .limit(10)
    .collect(Collectors.toList());
```

### Spliterator can't split well
`LinkedList` splits poorly — it must traverse to find the midpoint. `ArrayList` splits at index in O(1). Source structure matters enormously.

| Source | Splits well? |
|---|---|
| `ArrayList` | ✅ |
| `int[]` array | ✅ |
| `IntStream.range` | ✅ |
| `LinkedList` | ❌ |
| `HashSet` | moderate |
| `Files.lines()` | ❌ poor |

---

## Ordering in Parallel Streams

### `forEach` vs `forEachOrdered`

```java
List<Integer> list = List.of(1, 2, 3, 4, 5);

// forEach — no order guarantee in parallel
list.parallelStream().forEach(System.out::println);
// might print: 3 1 5 2 4

// forEachOrdered — guarantees encounter order, kills parallel benefit
list.parallelStream().forEachOrdered(System.out::println);
// always prints: 1 2 3 4 5
```

### `findFirst()` vs `findAny()` in parallel

```java
// findFirst — must return the FIRST element in encounter order
// requires coordination between partitions — slower in parallel
Optional<Integer> first = list.parallelStream()
    .filter(n -> n > 2)
    .findFirst();
// always Optional[3]

// findAny — returns whichever partition finishes first
// faster in parallel, result is non-deterministic
Optional<Integer> any = list.parallelStream()
    .filter(n -> n > 2)
    .findAny();
// might return Optional[3], Optional[4], or Optional[5]
```

> **Interview rule:** In parallel streams, prefer `findAny` over `findFirst` unless you genuinely need the first element. `findFirst` is an ordered operation that requires inter-partition coordination.

---

## `unordered()` Hint

Tells the stream the caller doesn't care about encounter order, allowing optimizations:

```java
list.parallelStream()
    .unordered()          // hint: order doesn't matter
    .distinct()           // can now use partition-local sets
    .limit(100)           // can stop any partition that hits 100
    .collect(Collectors.toList());
```

`unordered()` doesn't randomize — it just removes the ordering constraint so the framework can take shortcuts.

---

## Stateless vs Stateful Operations

| Type | Operations | Parallel-safe? |
|---|---|---|
| Stateless | `filter`, `map`, `flatMap`, `peek` | ✅ — each element processed independently |
| Stateful | `distinct`, `sorted`, `limit`, `skip` | ⚠️ — require cross-partition state |

Stateless operations parallelize perfectly. Stateful ones require barriers and synchronization.

```java
// All stateless — parallelizes cleanly
list.parallelStream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// sorted() is stateful — barrier introduced
list.parallelStream()
    .map(String::toUpperCase)
    .sorted()              // full barrier here
    .collect(Collectors.toList());
```

---

## Custom Thread Pool — Critical for Production

The default `commonPool` is shared JVM-wide. In a Spring Boot microservice, if a parallel stream hogs all commonPool threads, other framework code using ForkJoin (like `CompletableFuture`) is starved.

Run parallel streams in a **dedicated pool**:

```java
ForkJoinPool customPool = new ForkJoinPool(4);

try {
    List<String> result = customPool.submit(() ->
        largeList.parallelStream()
            .filter(s -> s.startsWith("O"))
            .map(String::toUpperCase)
            .collect(Collectors.toList())
    ).get();
} catch (InterruptedException | ExecutionException e) {
    Thread.currentThread().interrupt();
    throw new RuntimeException(e);
} finally {
    customPool.shutdown();
}
```

The stream inherits the thread pool of the **submitting thread**. Submitting via `customPool.submit()` makes that pool the execution context.

---

## Parallel Reduction — Rules Recap

For `reduce` to be correct in parallel:
- Identity must be truly neutral
- Accumulator must be associative and stateless
- Combiner must correctly merge two partial results

```java
// Correct parallel reduce
int sum = list.parallelStream()
    .reduce(0, Integer::sum, Integer::sum);
//          ↑         ↑              ↑
//       identity  accumulator   combiner

// collect is always parallel-safe via Collector's combiner
List<String> result = list.parallelStream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList()); // framework handles combining
```

---

## Benchmark Mindset — Always Measure

Never assume parallel is faster. The only truth is a benchmark.

```java
// JMH (Java Microbenchmark Harness) — proper benchmarking
// Quick sanity check without JMH:

long start = System.nanoTime();
long seqResult = LongStream.rangeClosed(1, 10_000_000).sum();
long seqTime = System.nanoTime() - start;

start = System.nanoTime();
long parResult = LongStream.rangeClosed(1, 10_000_000).parallel().sum();
long parTime = System.nanoTime() - start;

System.out.println("Sequential: " + seqTime / 1_000_000 + "ms");
System.out.println("Parallel:   " + parTime / 1_000_000 + "ms");
```

On a real multi-core machine with a large range, parallel wins decisively here because `LongStream.range` splits perfectly and `sum` is stateless and associative.

---

## ShopSphere Context

```java
// Processing 1M orders for a daily analytics job — good parallel candidate
List<Order> millionOrders = // large dataset

// Revenue by category — parallel + groupingBy (concurrent collector needed)
Map<String, Double> revenueByCategory = millionOrders.parallelStream()
    .collect(Collectors.groupingByConcurrent(
        Order::getCategory,
        Collectors.summingDouble(Order::getAmount)
    ));
// groupingByConcurrent — parallel-safe version of groupingBy
// uses ConcurrentHashMap internally
```

> **`groupingBy` vs `groupingByConcurrent`:** `groupingBy` in parallel creates a map per partition and merges — safe but more memory. `groupingByConcurrent` uses a single `ConcurrentHashMap` with concurrent accumulation — less memory, but loses encounter order. Use `groupingByConcurrent` for large parallel aggregations where order doesn't matter.

---

## Interview Questions on This Topic

**Q: What thread pool do parallel streams use by default?**
`ForkJoinPool.commonPool()`, with parallelism level of `availableProcessors() - 1`. It's shared across the entire JVM — meaning parallel streams in different parts of your app compete for the same threads.

**Q: How do you run a parallel stream in a custom thread pool?**
Submit the stream operation as a task to a `ForkJoinPool` instance via `pool.submit(() -> stream...).get()`. The stream inherits the pool from the submitting thread.

**Q: Why is `sorted()` bad in a parallel stream?**
It's a stateful operation requiring all elements to be collected before any can be emitted. It introduces a full synchronization barrier, forcing all partitions to complete before sorting begins — eliminating most parallelism benefit.

**Q: `findFirst()` vs `findAny()` in parallel — which is faster and why?**
`findAny()` is faster. It returns the result from whichever partition finishes first. `findFirst()` must return the element that is first in encounter order, requiring coordination across partitions to verify no earlier element exists.

**Q: What is `groupingByConcurrent` and how does it differ from `groupingBy`?**
`groupingByConcurrent` uses a `ConcurrentHashMap` as the shared container, allowing concurrent accumulation across all parallel partitions without merging separate maps. `groupingBy` in parallel creates one map per partition and merges at the end. `groupingByConcurrent` is more memory-efficient but unordered — the resulting map doesn't preserve encounter order of elements within groups.

**Q: When would you choose sequential over parallel for a large dataset?**
When the operation is I/O-bound (threads would just block), when the operation has shared mutable state, when the source splits poorly (LinkedList), when stateful operations like `sorted` or `distinct` dominate the pipeline, or when the common pool is already under load from other framework code and you can't afford a custom pool.

---

Say **next** and we go to **Phase 8 — Spliterator** — the iterator behind streams, characteristics, splitting for parallel, and implementing a custom Spliterator for a paginated API source.
