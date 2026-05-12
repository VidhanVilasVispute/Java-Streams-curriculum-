
# Phase 10 — Interview and System Design Angle

## The Questions That Separate Mid from Senior

---

## Q1: What is the difference between intermediate and terminal operations? Why are intermediate operations lazy?

**Model answer:**

Intermediate operations (`filter`, `map`, `flatMap`, `sorted`, etc.) return a new `Stream` and register a step in the pipeline. They never execute on their own. Terminal operations (`collect`, `forEach`, `reduce`, `count`, `findFirst`, etc.) trigger the entire pipeline to run and produce a result or side effect.

Laziness exists for two reasons:

**Efficiency:** Without laziness, each intermediate operation would traverse the entire source and produce an intermediate collection. With laziness, elements flow through the full pipeline one at a time — no intermediate storage, no wasted traversals.

**Short-circuiting:** Lazy evaluation allows terminals like `findFirst`, `anyMatch`, and `limit` to stop the pipeline as soon as they're satisfied. With a 10-million element source and `findFirst`, you may process only one element.

```java
// Without laziness — three full traversals, two intermediate lists
List<T> step1 = source.filter(...);   // full pass
List<T> step2 = step1.map(...);       // full pass
T result = step2.findFirst();         // still full pass

// With laziness — one element processed, pipeline stops
source.stream().filter(...).map(...).findFirst(); // processes 1 element
```

---

## Q2: What happens if you call a terminal operation twice on the same stream?

**Model answer:**

`IllegalStateException: stream has already been operated upon or closed`.

A stream is **single-use**. Once a terminal operation is called, the stream is consumed and closed. Any subsequent terminal operation on the same reference throws immediately.

```java
Stream<String> s = list.stream();
s.forEach(System.out::println); // fine
s.count();                       // IllegalStateException
```

**Fix:** Call `.stream()` again on the source, or collect to a `List` and re-stream from it.

**Why:** Streams are pipelines, not collections. They carry no data. Once the terminal drains the pipeline, there's nothing left. Restarting would require re-creating the source connection (which may be a file, network socket, or generator).

---

## Q3: What is the difference between `findFirst()` and `findAny()` in a parallel stream?

**Model answer:**

Both return an `Optional<T>` with an element satisfying the pipeline's filters.

- `findFirst()` — always returns the element that is **first in encounter order**. In a parallel stream this requires coordination across all partitions: each partition must check whether any earlier partition found a match. This is expensive — effectively a sequential constraint on a parallel pipeline.

- `findAny()` — returns **whichever element any partition finds first**. Non-deterministic. No inter-partition coordination needed. In parallel it's significantly faster.

```java
// Ordered list [1,2,3,4,5,6,7,8], filter even
list.parallelStream().filter(n -> n % 2 == 0).findFirst(); // always Optional[2]
list.parallelStream().filter(n -> n % 2 == 0).findAny();  // might be 2, 4, 6, or 8
```

**Interview rule:** In a parallel stream, use `findAny` unless you genuinely need the first element in encounter order. `findFirst` with a parallel stream is a contradiction — you're paying for parallelism but enforcing sequential ordering.

---

## Q4: Why must the identity value in parallel `reduce` be truly neutral, and why must the accumulator be associative?

**Model answer:**

In a parallel `reduce`, the stream is split into N partitions. Each partition independently starts with the **identity value** and accumulates its elements. The partial results are then merged with the combiner.

```
Source: [1, 2, 3, 4, 5, 6]
Split:  [1, 2, 3]  and  [4, 5, 6]

Thread-1: identity(0) + 1 + 2 + 3 = 6
Thread-2: identity(0) + 4 + 5 + 6 = 15
Combine:  6 + 15 = 21  ✅
```

**If identity is wrong (e.g., 10):**

```
Thread-1: 10 + 1 + 2 + 3 = 16
Thread-2: 10 + 4 + 5 + 6 = 25
Combine:  16 + 25 = 41  ← wrong (correct answer is 21)
```

The identity is applied **once per partition**, not once globally. With N partitions you get N × identity folded in instead of one.

**Associativity requirement:** The framework may combine partial results in any order depending on which threads finish first. If `(a op b) op c ≠ a op (b op c)`, different combination orders give different answers.

```java
// Subtraction — not associative
(10 - 3) - 2 = 5
10 - (3 - 2) = 9   // different result
```

A parallel reduce with subtraction gives non-deterministic, wrong results.

---

## Q5: How does a SIZED Spliterator affect parallel splitting, and what happens when SIZED is lost?

**Model answer:**

A `SIZED` Spliterator has an exact element count via `estimateSize()`. This allows `trySplit()` to split at the **exact midpoint** — producing two perfectly balanced halves. Balanced splits mean even load distribution across threads, which is optimal.

```
ArrayList of 1000 elements — SIZED:
Split 1: [0..499] and [500..999]  — perfectly balanced
Split 2: [0..249], [250..499], [500..749], [750..999]  — 4 equal partitions
```

When `SIZED` is lost (after `filter`, `flatMap`, etc.), the framework can only estimate. Splits become unbalanced — one thread may finish early and sit idle while another processes most of the data. The framework falls back to a work-stealing strategy but it's less efficient.

```java
// SIZED — perfect splits
new ArrayList<>(list).spliterator();                    // SIZED + SUBSIZED

// SIZED lost — filter doesn't know how many elements survive
list.stream().filter(predicate).spliterator();          // NOT SIZED
```

**Practical implication:** If you need parallel efficiency, keep `SIZED` as long as possible. Avoid `filter` before expensive parallel operations on large datasets — consider filtering after splitting by collecting to a filtered list first.

---

## Q6: What are the performance implications of `sorted()` in a parallel stream?

**Model answer:**

`sorted()` is a **stateful, full-barrier operation**. It must:

1. Collect **all elements** from all partitions into a single buffer
2. Sort the entire buffer
3. Re-emit elements in order

Step 1 destroys all parallelism benefit for everything before `sorted()` — it forces all threads to synchronize and dump into one collection. Step 2 is sequential. Step 3 re-enables parallelism for downstream operations.

```
parallelStream()
    .filter(...)       // runs in parallel ✅
    .map(...)          // runs in parallel ✅
    .sorted()          // full barrier — all threads sync, sequential sort ❌
    .limit(10)         // runs after sorted
    .collect(...)
```

**Cost:** O(n log n) sort after O(n/threads) parallel processing. The parallel benefit is capped by the sort barrier.

**Alternative patterns:**

```java
// If you only need top-K elements — don't sort the whole stream
// Use a PriorityQueue-based custom Collector — O(n log k) not O(n log n)
orders.parallelStream()
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        list -> list.stream()
            .sorted(Comparator.comparingDouble(Order::getAmount).reversed())
            .limit(10)
            .collect(Collectors.toList())
    ));

// Or use min/max which don't sort — O(n)
Optional<Order> maxOrder = orders.parallelStream()
    .max(Comparator.comparingDouble(Order::getAmount));
```

---

## Q7: When would you choose sequential over parallel for a large dataset?

**Model answer — six cases:**

**1. I/O-bound operations**
Parallel streams use CPU threads from `ForkJoinPool`. If each element requires a network call or DB query, threads block on I/O. You get no parallelism — just thread exhaustion. Use `CompletableFuture` with a dedicated thread pool instead.

**2. Poor source splittability**
`LinkedList`, `Files.lines()`, iterator-backed streams — `trySplit()` is expensive or unbalanced. Splitting overhead exceeds gain.

**3. Stateful pipeline dominates**
If `sorted()`, `distinct()`, or `limit()` dominate, the synchronization barriers eliminate parallel benefit.

**4. Small datasets**
ForkJoin overhead — task submission, splitting, combining — exceeds computation time. For fewer than ~10,000 simple operations, sequential is faster.

**5. Shared mutable state**
Operations with side effects on shared non-thread-safe state (counters, external lists, maps) require synchronization that kills parallelism and risks data corruption.

**6. commonPool under load**
In a Spring Boot service, the commonPool is shared with `CompletableFuture` and other framework code. A parallel stream that saturates it can starve the rest of the application. Use a custom `ForkJoinPool` or switch to sequential.

---

## Q8: How would you process a 10-million record dataset from a database using Streams without loading everything into memory?

**Model answer — three approaches, know all three:**

### Approach 1 — JDBC ResultSet as Spliterator (pure Java)

```java
// Set fetchSize to stream in chunks from DB
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM orders WHERE status = 'ACTIVE'",
    ResultSet.TYPE_FORWARD_ONLY,
    ResultSet.CONCUR_READ_ONLY
);
stmt.setFetchSize(1000); // fetch 1000 rows at a time from DB
ResultSet rs = stmt.executeQuery();

Stream<Order> orderStream = StreamSupport.stream(
    new ResultSetSpliterator(rs), false
);

orderStream
    .filter(o -> o.getAmount() > 5000)
    .mapToDouble(Order::getAmount)
    .sum();
// DB sends 1000 rows at a time, stream processes lazily
// Never more than 1000 rows in JVM memory at once
```

### Approach 2 — Spring Data `@Query` with `Stream<T>` return type

```java
// Repository
@Query("SELECT o FROM Order o WHERE o.status = :status")
@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "1000"))
Stream<Order> streamByStatus(@Param("status") String status);

// Service — must be in a transaction, must close the stream
@Transactional(readOnly = true)
public double calculateRevenue(String status) {
    try (Stream<Order> orders = orderRepo.streamByStatus(status)) {
        return orders
            .filter(o -> o.getAmount() > 1000)
            .mapToDouble(Order::getAmount)
            .sum();
    } // stream closed here — releases DB cursor
}
```

Spring Data uses Hibernate ScrollableResults under the hood — streams rows lazily with `fetchSize` batching.

### Approach 3 — Pagination with `Stream.iterate`

```java
// When cursor-based streaming isn't available
int pageSize = 1000;

Stream.iterate(0, page -> page + 1)
    .map(page -> orderRepo.findAll(PageRequest.of(page, pageSize)))
    .takeWhile(pageResult -> pageResult.hasContent())
    .flatMap(pageResult -> pageResult.getContent().stream())
    .filter(o -> o.getAmount() > 5000)
    .mapToDouble(Order::getAmount)
    .sum();
```

**Interview follow-up:** Which approach is best?

- **Approach 1** — most control, no framework dependency, works for any JDBC
- **Approach 2** — cleanest for Spring apps, Hibernate handles cursor management
- **Approach 3** — works without cursor support, but each page is a separate DB query — N queries vs 1

---

## Q9: `collect` vs `reduce` — the definitive answer

**Model answer:**

`reduce` is an **immutable reduction** — each step produces a new immutable result by combining the accumulator with the next element. Appropriate for: summing numbers, finding max/min, folding into a value of a different type.

`collect` is a **mutable reduction** — it creates a mutable container once (Supplier) and accumulates elements into it (BiConsumer). No intermediate objects created per step. Appropriate for: building collections, maps, strings, custom containers.

```java
// reduce to build a list — WRONG
// Creates new ArrayList every step — O(n) allocations, O(n²) total copies
List<Integer> wrong = stream.reduce(
    new ArrayList<>(),
    (list, e) -> { list.add(e); return list; }, // mutates identity — broken in parallel
    (a, b) -> { a.addAll(b); return a; }
);

// collect — CORRECT
// One ArrayList, elements added in O(1), merge in parallel O(m)
List<Integer> right = stream.collect(Collectors.toList());
```

**The golden rule:** If you're building a container, always use `collect`. If you're folding into a single scalar value, `reduce` is appropriate — though for numeric aggregation, prefer primitive stream terminals (`sum`, `average`) over `reduce`.

---

## Q10: Full Pipeline Design Question

**"Given a list of 5 million orders, group by category, find the top 3 categories by total revenue, and for each return the category name, total revenue, and the most frequently ordered product. Do this efficiently."**

**Model answer — think out loud:**

```java
// Step 1: group by category, collect to CategoryStats in one pass
record CategoryStats(String category, double revenue, Map<String, Long> productCounts) {}

Map<String, CategoryStats> statsByCategory = orders.parallelStream()
    .collect(Collectors.groupingByConcurrent(  // parallel-safe
        Order::getCategory,
        Collectors.collectingAndThen(
            Collectors.toList(),
            categoryOrders -> {
                double revenue = categoryOrders.stream()
                    .mapToDouble(Order::getAmount).sum();

                Map<String, Long> productCounts = categoryOrders.stream()
                    .flatMap(o -> o.getItems().stream())
                    .collect(Collectors.groupingBy(
                        Item::getName, Collectors.counting()
                    ));

                return new CategoryStats(
                    categoryOrders.get(0).getCategory(),
                    revenue,
                    productCounts
                );
            }
        )
    ));

// Step 2: top 3 categories by revenue
List<CategoryReport> top3 = statsByCategory.values().stream()
    .sorted(Comparator.comparingDouble(CategoryStats::revenue).reversed())
    .limit(3)
    .map(stats -> {
        String topProduct = stats.productCounts().entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElse("N/A");

        return new CategoryReport(stats.category(), stats.revenue(), topProduct);
    })
    .collect(Collectors.toList());
```

**Talking points for the interview:**
- Used `groupingByConcurrent` for parallel safety on 5M records
- Single-pass grouping with nested stats computation via `collectingAndThen`
- `flatMap` to flatten items across orders within each category group
- Deferred `sorted().limit(3)` to step 2 — only sort the small category map (say 20 categories), not the 5M orders
- `max()` instead of `sorted().findFirst()` for top product — O(n) not O(n log n)

---

## Final Revision Map

```
Phase 1 — Prerequisites
  ├── Functional Interfaces (Predicate, Function, Consumer, Supplier)
  ├── Lambda syntax, method references (4 forms), effectively final
  └── Optional — creation, map, flatMap, orElseGet vs orElse

Phase 2 — Fundamentals
  ├── Source → Intermediate → Terminal
  ├── Lazy + single-use
  └── All creation methods

Phase 3 — Collectors
  ├── groupingBy + all downstream collectors
  ├── toMap duplicate handling
  ├── collectingAndThen, teeing
  └── Custom Collector (supplier, accumulator, combiner, finisher, characteristics)

Phase 4 — flatMap
  ├── map vs flatMap — the nested structure problem
  ├── flatMap on Optional
  └── Conditional inclusion with Stream.of / Stream.empty

Phase 5 — Primitive Streams
  ├── IntStream, LongStream, DoubleStream — no boxing
  ├── sum, average, summaryStatistics
  └── Conversion: mapToInt, boxed, mapToObj, asLongStream

Phase 6 — Reduction
  ├── reduce — identity contract, associativity
  ├── 3-arg form — combiner only runs in parallel
  └── reduce vs collect — when to use each

Phase 7 — Parallel Streams
  ├── ForkJoinPool.commonPool internals
  ├── When parallel wins / backfires
  ├── findFirst vs findAny, forEach vs forEachOrdered
  └── Custom ForkJoinPool

Phase 8 — Spliterator
  ├── tryAdvance, trySplit, estimateSize, characteristics
  ├── Source splittability (ArrayList vs LinkedList)
  └── Custom Spliterator — paginated API, ResultSet streaming

Phase 9 — Advanced Patterns
  ├── takeWhile / dropWhile (Java 9)
  ├── Exception handling — wrap + Try/Result monad
  ├── peek — debug only
  └── Full custom Collector (BatchCollector)

Phase 10 — Interview Angle
  ├── All Q&A above
  └── 10M record DB streaming — 3 approaches
```

---

That's the complete Java 8 Streams roadmap — all 10 phases, interview-ready, grounded in ShopSphere patterns throughout. You're well-equipped for mid to senior Java interviews on this topic.
