# Phase 5 — Primitive Streams

## Why Primitive Streams Exist

Every time you use `Stream<Integer>`, Java boxes each `int` into an `Integer` object on the heap. For large datasets this is significant:

```java
// Boxing overhead — each int wrapped in Integer object
Stream<Integer> boxed = Stream.of(1, 2, 3, 4, 5);

// No boxing — raw int values
IntStream primitive = IntStream.of(1, 2, 3, 4, 5);
```

For a million integers:
- `Stream<Integer>` — 1M `Integer` objects on heap + GC pressure
- `IntStream` — contiguous int array in memory, no objects, no GC

The three primitive stream types:

| Type | Primitive | Object equivalent |
|---|---|---|
| `IntStream` | `int` | `Stream<Integer>` |
| `LongStream` | `long` | `Stream<Long>` |
| `DoubleStream` | `double` | `Stream<Double>` |

No `BooleanStream`, no `CharStream`, no `ByteStream` — only these three.

---

## Creating Primitive Streams

### `IntStream.of`, `LongStream.of`, `DoubleStream.of`

```java
IntStream    is = IntStream.of(1, 2, 3, 4, 5);
LongStream   ls = LongStream.of(100L, 200L, 300L);
DoubleStream ds = DoubleStream.of(1.1, 2.2, 3.3);
```

---

### `IntStream.range` and `IntStream.rangeClosed`

```java
IntStream.range(0, 5)        // 0, 1, 2, 3, 4       — exclusive end
IntStream.rangeClosed(1, 5)  // 1, 2, 3, 4, 5       — inclusive end
```

This is the functional replacement for the classic for loop:

```java
// Old
for (int i = 0; i < 10; i++) {
    System.out.println("Item-" + i);
}

// Stream
IntStream.range(0, 10)
    .forEach(i -> System.out.println("Item-" + i));
```

`LongStream` has the same — `LongStream.range(0L, 1_000_000_000L)` — important for large ranges where `int` overflows.

---

### `Arrays.stream(primitiveArray)`

```java
int[]    ints    = {1, 2, 3, 4, 5};
double[] doubles = {1.1, 2.2, 3.3};

IntStream    is = Arrays.stream(ints);
DoubleStream ds = Arrays.stream(doubles);
```

---

### `String.chars()` — returns `IntStream`

```java
"hello".chars()                       // IntStream of char code points
       .filter(c -> c != 'l')
       .mapToObj(c -> String.valueOf((char) c))
       .collect(Collectors.joining());
// "heo"
```

---

### `iterate` and `generate` on Primitive Streams

```java
// IntStream.iterate
IntStream.iterate(0, n -> n + 2)
    .limit(5)
    .forEach(System.out::println);
// 0 2 4 6 8

// IntStream.generate
IntStream.generate(() -> (int)(Math.random() * 100))
    .limit(5)
    .forEach(System.out::println);
```

---

## Extra Terminal Operations — Unavailable on `Stream<T>`

This is the main reason to use primitive streams. These don't exist on `Stream<T>`:

### `sum()`

```java
int total = IntStream.rangeClosed(1, 100).sum();  // 5050

double totalRevenue = orders.stream()
    .mapToDouble(Order::getAmount)
    .sum();
```

---

### `average()` — returns `OptionalDouble`

```java
OptionalDouble avg = IntStream.of(10, 20, 30, 40).average();
avg.ifPresent(System.out::println); // 25.0

// OptionalDouble, not Optional<Double> — important distinction
double result = IntStream.empty().average().orElse(0.0);
```

---

### `min()` and `max()` — return `OptionalInt / OptionalLong / OptionalDouble`

```java
OptionalInt max = IntStream.of(3, 1, 4, 1, 5, 9).max();
// OptionalInt[9]

OptionalInt min = IntStream.of(3, 1, 4, 1, 5, 9).min();
// OptionalInt[1]
```

> **Interview trap:** `Stream<T>` has `min(Comparator)` and `max(Comparator)` requiring a Comparator. Primitive streams have no-arg `min()` and `max()` — no Comparator needed because the ordering is obvious.

---

### `summaryStatistics()` — everything in one pass

```java
IntSummaryStatistics stats = IntStream.rangeClosed(1, 10)
    .summaryStatistics();

stats.getCount();    // 10
stats.getSum();      // 55
stats.getMin();      // 1
stats.getMax();      // 10
stats.getAverage();  // 5.5
```

ShopSphere use:

```java
IntSummaryStatistics orderStats = orders.stream()
    .mapToInt(Order::getItemCount)
    .summaryStatistics();

System.out.println("Orders: "     + orderStats.getCount());
System.out.println("Total items: "+ orderStats.getSum());
System.out.println("Avg items: "  + orderStats.getAverage());
System.out.println("Max items: "  + orderStats.getMax());
```

---

## Converting Between Stream Types

This is essential knowledge — conversions in all directions.

### `Stream<T>` → Primitive Stream (`mapToInt`, `mapToLong`, `mapToDouble`)

```java
// Stream<Order> → IntStream
IntStream itemCounts = orders.stream()
    .mapToInt(Order::getItemCount);

// Stream<String> → IntStream (length of each string)
IntStream lengths = words.stream()
    .mapToInt(String::length);

// Stream<Product> → DoubleStream
DoubleStream prices = products.stream()
    .mapToDouble(Product::getPrice);
```

---

### Primitive Stream → `Stream<T>` (`boxed()` or `mapToObj()`)

```java
// IntStream → Stream<Integer>
Stream<Integer> boxed = IntStream.range(1, 6).boxed();

// IntStream → Stream<String>
Stream<String> labels = IntStream.range(1, 6)
    .mapToObj(i -> "Item-" + i);
// ["Item-1", "Item-2", "Item-3", "Item-4", "Item-5"]
```

`boxed()` is shorthand for `mapToObj(Integer::valueOf)`.

---

### Primitive Stream → Different Primitive Stream

```java
// IntStream → LongStream
LongStream ls = IntStream.range(0, 5).asLongStream();

// IntStream → DoubleStream
DoubleStream ds = IntStream.range(0, 5).asDoubleStream();

// No direct LongStream → IntStream — must use mapToInt
IntStream is = LongStream.range(0L, 5L).mapToInt(Math::toIntExact);
```

---

## Practical Patterns

### Index-based access with `IntStream`

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Dave");

// Print with 1-based index
IntStream.range(0, names.size())
    .forEach(i -> System.out.println((i + 1) + ". " + names.get(i)));
// 1. Alice
// 2. Bob
// 3. Charlie
// 4. Dave
```

---

### Generating a sequence for IDs

```java
List<String> orderIds = IntStream.rangeClosed(1001, 1005)
    .mapToObj(id -> "ORD-" + id)
    .collect(Collectors.toList());
// [ORD-1001, ORD-1002, ORD-1003, ORD-1004, ORD-1005]
```

---

### Computing with mixed types

```java
// Total discount amount across all orders
double totalDiscount = orders.stream()
    .mapToDouble(o -> o.getAmount() * o.getDiscountPercent() / 100.0)
    .sum();
```

---

### Parallel sum on large IntStream

```java
long sum = LongStream.rangeClosed(1, 1_000_000)
    .parallel()
    .sum();
// 500000500000
```

This is where primitive + parallel gives maximum benefit — no boxing, no heap pressure, ForkJoin splits the range cleanly.

---

## Optional Variants for Primitive Streams

Primitive streams return primitive Optional types — not `Optional<Integer>`:

| Stream | Optional type |
|---|---|
| `IntStream` | `OptionalInt` |
| `LongStream` | `OptionalLong` |
| `DoubleStream` | `OptionalDouble` |

```java
OptionalInt  oi = IntStream.of(1, 2, 3).max();   // OptionalInt[3]
OptionalDouble od = DoubleStream.empty().average(); // OptionalDouble.empty

// These are NOT Optional<Integer> — they have orElse but not map/flatMap
oi.orElse(0);
oi.orElseThrow(RuntimeException::new);
oi.isPresent();
// but NO: oi.map(...) — primitive Optionals don't have map/filter/flatMap
```

> **Interview trap:** `OptionalInt` is NOT a subtype of `Optional<Integer>`. It has `getAsInt()`, not `get()`. If you need `map`/`flatMap`, call `.boxed()` first to get `Optional<Integer>`.

---

## ShopSphere Practice Problem

```java
List<Order> orders = // ... list of orders with items

// 1. Total revenue
double revenue = orders.stream()
    .mapToDouble(Order::getAmount)
    .sum();

// 2. Average order value
OptionalDouble avgOrder = orders.stream()
    .mapToDouble(Order::getAmount)
    .average();

// 3. Order value statistics
DoubleSummaryStatistics stats = orders.stream()
    .mapToDouble(Order::getAmount)
    .summaryStatistics();

// 4. Generate order IDs for a batch of 10 new orders
List<String> newIds = IntStream.rangeClosed(1, 10)
    .mapToObj(i -> "ORD-" + System.currentTimeMillis() + "-" + i)
    .collect(Collectors.toList());

// 5. Find orders with more than 3 items
long bulkOrders = orders.stream()
    .mapToInt(Order::getItemCount)
    .filter(count -> count > 3)
    .count();
```

---

## Interview Questions on This Topic

**Q: Why should you use `IntStream` instead of `Stream<Integer>` for numeric operations?**
`Stream<Integer>` boxes each `int` into an `Integer` heap object — adds GC pressure and memory overhead. `IntStream` stores raw `int` values, avoids boxing entirely, and provides specialized terminals (`sum`, `average`, `summaryStatistics`) that aren't available on `Stream<T>`.

**Q: What does `mapToInt` return, and how is it different from `map`?**
`mapToInt` returns an `IntStream` — a specialized primitive stream. `map` returns a `Stream<R>`. Use `mapToInt` when your mapper extracts an int property and you plan to aggregate numerically.

**Q: What is `summaryStatistics()` and when is it better than separate operations?**
It computes count, sum, min, max, and average in a **single pass** over the stream. If you need more than one of these, calling five separate terminal operations would iterate the source five times. `summaryStatistics()` does it once — always prefer it when multiple stats are needed.

**Q: What is the difference between `OptionalInt` and `Optional<Integer>`?**
`OptionalInt` is a primitive-specialized container — it has `getAsInt()`, `orElse(int)`, `isPresent()`, but **no** `map`, `flatMap`, or `filter`. `Optional<Integer>` is the full generic Optional with all functional methods. To convert: `optionalInt.boxed()` returns `Optional<Integer>`.

**Q: How do you convert an `IntStream` back to a `Stream<String>`?**
Use `mapToObj`:
```java
IntStream.range(1, 6).mapToObj(i -> "Item-" + i)
```
Or `boxed()` if you just need `Stream<Integer>`.

---

Say **next** and we move to **Phase 6 — Reduction** — `reduce` internals, identity rules, the three-arg parallel form, and when to use `reduce` vs `collect`.
