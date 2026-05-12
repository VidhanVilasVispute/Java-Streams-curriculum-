# Phase 9 — Advanced Patterns and Real-World Usage

## 1. Lazy Evaluation Chains and Short-Circuiting

You already know streams are lazy. Here's where it pays off in real pipeline design.

### The vertical execution model revisited

```java
List.of("apple", "ant", "banana", "avocado", "cherry")
    .stream()
    .peek(s -> System.out.println("source:   " + s))
    .filter(s -> s.startsWith("a"))
    .peek(s -> System.out.println("filtered: " + s))
    .map(String::toUpperCase)
    .peek(s -> System.out.println("mapped:   " + s))
    .findFirst();
```

Output:
```
source:   apple
filtered: apple
mapped:   APPLE
← stops here, findFirst() satisfied
```

`"ant"`, `"banana"`, `"avocado"`, `"cherry"` are never touched. The pipeline processes **one element vertically** until a terminal is satisfied, then stops.

---

### Designing pipelines for early exit

```java
// Expensive operation — only run if truly needed
Optional<Order> fraudulentOrder = orders.stream()
    .filter(Order::isHighValue)           // cheap — filter first
    .filter(this::passesRiskCheck)        // moderate cost
    .filter(this::callsFraudDetectionAPI) // expensive — put last
    .findFirst();                          // stops at first match
```

**Rule:** Put cheapest, most selective filters first. Expensive operations last. Short-circuit terminals (`findFirst`, `anyMatch`, `limit`) amplify this benefit.

---

## 2. Infinite Streams — Patterns and Safety

### `Stream.iterate` patterns

```java
// Fibonacci sequence
Stream.iterate(new long[]{0, 1}, f -> new long[]{f[1], f[0] + f[1]})
    .limit(10)
    .map(f -> f[0])
    .forEach(System.out::println);
// 0 1 1 2 3 5 8 13 21 34

// Exponential backoff delays
Stream.iterate(1000L, delay -> delay * 2)
    .limit(5)
    .forEach(delay -> System.out.println("retry in " + delay + "ms"));
// 1000 2000 4000 8000 16000
```

### `Stream.generate` for polling/sampling

```java
// Poll until condition met — with limit as safety cap
Stream.generate(() -> orderService.checkStatus(orderId))
    .limit(10)                                  // max 10 polls
    .filter(status -> status == COMPLETED)
    .findFirst()
    .orElseThrow(() -> new TimeoutException("Order not completed"));
```

### `takeWhile` and `dropWhile` — Java 9

```java
// takeWhile — emit elements while predicate holds, stop at first failure
Stream.iterate(1, n -> n + 1)
    .takeWhile(n -> n < 6)
    .forEach(System.out::println);
// 1 2 3 4 5

// dropWhile — skip elements while predicate holds, emit the rest
Stream.of(1, 2, 3, 10, 4, 5)
    .dropWhile(n -> n < 5)
    .forEach(System.out::println);
// 10 4 5  ← once predicate fails, all subsequent elements pass through
```

> **Interview trap:** `dropWhile` is NOT the same as `filter(n -> n >= 5)`. It stops dropping at the first failure and passes everything after — including elements that would fail the predicate again (`4`, `5` below `10` still appear).

---

## 3. Stream vs Collection — When to Stay Lazy

```java
// Materialize early — lose laziness
List<Order> filtered = orders.stream()
    .filter(Order::isActive)
    .collect(Collectors.toList()); // materialized into memory

filtered.stream()          // new stream from materialized list
    .map(Order::getAmount)
    .sum();

// Keep lazy — one pipeline, no intermediate list
double sum = orders.stream()
    .filter(Order::isActive)
    .mapToDouble(Order::getAmount)
    .sum();
```

**When to materialize (collect early):**
- You need to reuse the result in multiple pipelines
- You need `size()` of the intermediate result
- You need random access into the filtered result
- Downstream operations require the full dataset (e.g., sorting)

**When to stay lazy:**
- Single-pass terminal operation
- Large or infinite source
- Short-circuit terminal — don't waste memory collecting what won't be used

---

## 4. Exception Handling in Streams

Checked exceptions don't fit functional interfaces. `Predicate.test()`, `Function.apply()`, etc. declare no checked exceptions. This is the single biggest pain point in production stream code.

### The problem

```java
// Compile error — readAllBytes throws IOException (checked)
List<byte[]> contents = paths.stream()
    .map(Files::readAllBytes)  // ❌ Unhandled exception: IOException
    .collect(Collectors.toList());
```

### Solution 1 — Wrap in unchecked (most common)

```java
// Utility wrapper
public static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> fn) {
    return t -> {
        try {
            return fn.apply(t);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    };
}

@FunctionalInterface
interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;
}

// Usage — clean
List<byte[]> contents = paths.stream()
    .map(wrap(Files::readAllBytes))  // ✅
    .collect(Collectors.toList());
```

### Solution 2 — Either/Try monad pattern

For production code where you want to collect both successes and failures without aborting the pipeline:

```java
record Result<T>(T value, Exception error) {
    static <T> Result<T> of(Supplier<T> supplier) {
        try {
            return new Result<>(supplier.get(), null);
        } catch (Exception e) {
            return new Result<>(null, e);
        }
    }
    boolean isSuccess() { return error == null; }
}

// Collect successes and failures separately
List<Result<byte[]>> results = paths.stream()
    .map(path -> Result.of(() -> Files.readAllBytes(path)))
    .collect(Collectors.toList());

List<byte[]> successes = results.stream()
    .filter(Result::isSuccess)
    .map(Result::value)
    .collect(Collectors.toList());

List<Exception> failures = results.stream()
    .filter(r -> !r.isSuccess())
    .map(Result::error)
    .collect(Collectors.toList());
```

This is the pattern used in production batch pipelines — process everything, segregate errors, report at the end.

---

## 5. Debugging Streams with `peek`

`peek` is an intermediate operation that applies a Consumer **without consuming the stream**. It's transparent — elements pass through unchanged.

```java
List<String> result = orders.stream()
    .peek(o -> log.debug("before filter: {}", o.getId()))
    .filter(o -> o.getAmount() > 1000)
    .peek(o -> log.debug("after filter: {}", o.getId()))
    .map(Order::getId)
    .peek(id -> log.debug("mapped id: {}", id))
    .collect(Collectors.toList());
```

### Production rules for `peek`

```java
// ❌ WRONG — using peek for side effects in production
stream.peek(order -> database.save(order))  // side effect — unreliable
      .collect(Collectors.toList());
// peek may not be called if the terminal short-circuits
// peek is not called on elements filtered out before it

// ✅ CORRECT — use forEach or map for side effects
stream.map(order -> { database.save(order); return order; }) // explicit
      .collect(Collectors.toList());

// ✅ CORRECT — peek only for debug logging
stream.peek(o -> log.debug("processing: " + o.getId()))
      .filter(...)
      .collect(...);
```

> **Interview point:** `peek` with a non-debug side effect is a code smell. Because of laziness and short-circuiting, there's no guarantee how many times or in what order `peek` is invoked. Never persist, modify external state, or count via `peek`.

---

## 6. Three-Arg `collect` — Custom Mutable Reduction

When no built-in Collector fits, use the three-arg form:

```java
<R> R collect(Supplier<R> supplier,
              BiConsumer<R, ? super T> accumulator,
              BiConsumer<R, R> combiner)
```

```java
// Collect orders into a custom summary object
class OrderSummary {
    double total;
    int count;
    void add(Order o) { total += o.getAmount(); count++; }
    void merge(OrderSummary other) { total += other.total; count += other.count; }
}

OrderSummary summary = orders.stream()
    .collect(
        OrderSummary::new,       // supplier
        OrderSummary::add,       // accumulator
        OrderSummary::merge      // combiner (for parallel)
    );

System.out.println("Total: " + summary.total);
System.out.println("Count: " + summary.count);
System.out.println("Avg:   " + summary.total / summary.count);
```

---

## 7. Writing a Full Custom Collector

Senior-level expectation. Here's a complete, production-quality custom Collector.

**Problem:** Collect orders into batches of N for bulk processing.

```java
public class BatchCollector<T> implements Collector<T, List<List<T>>, List<List<T>>> {

    private final int batchSize;

    public BatchCollector(int batchSize) {
        this.batchSize = batchSize;
    }

    @Override
    public Supplier<List<List<T>>> supplier() {
        return () -> {
            List<List<T>> result = new ArrayList<>();
            result.add(new ArrayList<>());  // start with one empty batch
            return result;
        };
    }

    @Override
    public BiConsumer<List<List<T>>, T> accumulator() {
        return (batches, element) -> {
            List<T> currentBatch = batches.get(batches.size() - 1);
            if (currentBatch.size() >= batchSize) {
                List<T> newBatch = new ArrayList<>();
                batches.add(newBatch);
                newBatch.add(element);
            } else {
                currentBatch.add(element);
            }
        };
    }

    @Override
    public BinaryOperator<List<List<T>>> combiner() {
        return (batches1, batches2) -> {
            // Merge last batch of batches1 with first batch of batches2
            // if last batch of batches1 isn't full
            List<T> lastOfFirst  = batches1.get(batches1.size() - 1);
            List<T> firstOfSecond = batches2.get(0);

            if (lastOfFirst.size() < batchSize) {
                int canTake = batchSize - lastOfFirst.size();
                lastOfFirst.addAll(firstOfSecond.subList(0, Math.min(canTake, firstOfSecond.size())));
                if (firstOfSecond.size() > canTake) {
                    batches2.set(0, new ArrayList<>(firstOfSecond.subList(canTake, firstOfSecond.size())));
                } else {
                    batches2.remove(0);
                }
            }
            batches1.addAll(batches2);
            return batches1;
        };
    }

    @Override
    public Function<List<List<T>>, List<List<T>>> finisher() {
        return batches -> {
            // Remove last batch if empty (edge case)
            if (!batches.isEmpty() && batches.get(batches.size()-1).isEmpty()) {
                batches.remove(batches.size()-1);
            }
            return batches;
        };
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Set.of(); // ordered, needs finisher
    }
}

// Usage
List<List<Order>> batches = orders.stream()
    .collect(new BatchCollector<>(100));

batches.forEach(batch -> bulkInsertService.insert(batch));
```

---

## 8. Collecting to Custom Data Structures

```java
// Collect to a TreeMap (sorted by key)
TreeMap<String, List<Order>> sortedByCategory = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCategory,
        TreeMap::new,      // map factory
        Collectors.toList()
    ));

// Collect to LinkedHashMap (insertion order preserved)
Map<String, Long> orderedCount = orders.stream()
    .sorted(Comparator.comparing(Order::getCreatedAt))
    .collect(Collectors.groupingBy(
        Order::getStatus,
        LinkedHashMap::new,
        Collectors.counting()
    ));
```

---

## 9. `Stream.ofNullable` — Java 9

```java
// Before Java 9 — verbose null guard before streaming
Stream<String> s = value != null ? Stream.of(value) : Stream.empty();

// Java 9
Stream<String> s = Stream.ofNullable(value);
// Empty stream if null, single-element stream if non-null
```

Useful in flatMap:
```java
// Flatten optional values in a list without NPE
List<String> values = List.of("a", null, "b", null, "c");
List<String> nonNull = values.stream()
    .flatMap(Stream::ofNullable)
    .collect(Collectors.toList());
// [a, b, c]
```

---

## 10. Collecting Statistics — Full ShopSphere Analytics Example

Pulling together everything from Phase 3–9 into one realistic pipeline:

```java
// Given orders: orderId, category, amount, status, createdAt, items

// Report: per category — count, revenue, avg order value, top item
Map<String, CategoryReport> report = orders.stream()
    .filter(o -> o.getStatus() != CANCELLED)
    .collect(Collectors.groupingBy(
        Order::getCategory,
        Collectors.collectingAndThen(
            Collectors.toList(),
            categoryOrders -> {
                long   count   = categoryOrders.size();
                double revenue = categoryOrders.stream()
                    .mapToDouble(Order::getAmount).sum();
                String topItem = categoryOrders.stream()
                    .flatMap(o -> o.getItems().stream())
                    .collect(Collectors.groupingBy(
                        Item::getName, Collectors.counting()
                    ))
                    .entrySet().stream()
                    .max(Map.Entry.comparingByValue())
                    .map(Map.Entry::getKey)
                    .orElse("N/A");

                return new CategoryReport(count, revenue, revenue/count, topItem);
            }
        )
    ));
```

This is a realistic senior-level stream exercise — nested collectors, flatMap, groupingBy inside collectingAndThen, max on a downstream map.

---

## Interview Questions on This Topic

**Q: Why should `peek` never be used for production side effects?**
Laziness and short-circuiting mean `peek` may not be called for every element — elements filtered before the `peek` never reach it, and short-circuit terminals stop the pipeline early. Side effects via `peek` are unreliable and order-dependent. Use `map` (returning the element after the side effect) or `forEach` instead.

**Q: How do you handle checked exceptions inside a stream pipeline?**
Two main patterns: wrap the throwing method in a utility that converts the checked exception to unchecked (RuntimeException), or use a Result/Either type that captures both success and failure, allowing the pipeline to continue and segregating errors at collection time.

**Q: What is the difference between `filter` and `takeWhile`?**
`filter` tests every element and includes those that match regardless of position. `takeWhile` (Java 9) emits elements while the predicate holds and **stops permanently** at the first failure — remaining elements are never tested. For ordered streams with a monotonic condition, `takeWhile` is more efficient.

**Q: When would you use the three-arg `collect` over a custom Collector?**
The three-arg `collect` (Supplier, BiConsumer accumulator, BiConsumer combiner) is for simple ad-hoc mutable reductions where writing a full Collector class is overkill. A custom Collector is preferred when you need `Characteristics` hints, a finisher transformation, or the Collector is reused across the codebase.

**Q: How would you process a stream in batches of N?**
Write a custom `Collector<T, List<List<T>>, List<List<T>>>` with a supplier that initializes a list containing one empty batch, an accumulator that starts a new batch when the current one reaches N, and a combiner that merges the last batch of the first container with the first batch of the second. This is the cleanest stream-native solution — no external libraries needed.

---

Say **next** and we finish with **Phase 10 — Interview and System Design Angle** — every high-signal question with full model answers, parallel stream correctness proofs, and the 10M record DB problem.
