# Phase 6 — Reduction

## What Reduction Is

Reduction is the process of combining all stream elements into **a single result** by repeatedly applying a combining function.

```
[1, 2, 3, 4, 5]  +  (a, b) -> a + b  =  15
```

Java Streams give you two reduction mechanisms:

- **`reduce`** — immutable reduction, produces a single value
- **`collect`** — mutable reduction, accumulates into a container

This phase covers `reduce`. Understand both so you can choose correctly in interviews.

---

## Form 1 — `reduce(identity, BinaryOperator<T>)`

```java
T reduce(T identity, BinaryOperator<T> accumulator)
```

- `identity` — the starting value; also the result if stream is empty
- `BinaryOperator` — combines two values into one
- Returns `T` directly — never returns `Optional`

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

int sum = nums.stream()
    .reduce(0, (a, b) -> a + b);
// 0+1=1, 1+2=3, 3+3=6, 6+4=10, 10+5=15
// result: 15

// Same with method reference
int sum = nums.stream().reduce(0, Integer::sum);
```

```java
// Product of all elements
int product = nums.stream().reduce(1, (a, b) -> a * b);
// 1*1*2*3*4*5 = 120

// String concatenation
String joined = Stream.of("Shop", "Sphere")
    .reduce("", (a, b) -> a + b);
// "ShopSphere"
```

---

## The Identity Contract — Critical Interview Point

The identity value must satisfy this law for **every** value `v` in the stream:

```
identity op v  ==  v
```

| Operation | Identity | Why |
|---|---|---|
| sum | `0` | `0 + v = v` |
| product | `1` | `1 * v = v` |
| string concat | `""` | `"" + v = v` |
| max | `Integer.MIN_VALUE` | `max(MIN_VALUE, v) = v` |
| min | `Integer.MAX_VALUE` | `min(MAX_VALUE, v) = v` |

**What happens if you use a wrong identity:**

```java
// Wrong — identity is 10, not 0
int sum = List.of(1, 2, 3).stream()
    .reduce(10, Integer::sum);
// 10+1+2+3 = 16, not 6

// In parallel — CATASTROPHIC
// The stream splits into [1,2] and [3]
// Each partition starts with identity 10:
// partition1: 10+1+2 = 13
// partition2: 10+3   = 13
// combined:   13+13  = 26  ← completely wrong
```

This is why the identity must be truly neutral — in parallel streams, it's applied once **per partition**, not once globally.

---

## Form 2 — `reduce(BinaryOperator<T>)` — No Identity

```java
Optional<T> reduce(BinaryOperator<T> accumulator)
```

Returns `Optional<T>` because the stream might be empty — there's no identity to return as a default.

```java
Optional<Integer> max = List.of(3, 1, 4, 1, 5, 9).stream()
    .reduce((a, b) -> a > b ? a : b);
// OptionalInt[9]

max.ifPresent(System.out::println); // 9

// Empty stream
Optional<Integer> empty = Stream.<Integer>empty()
    .reduce((a, b) -> a + b);
// Optional.empty
```

Use this form whenever there's no sensible identity value — for example, finding the longest string:

```java
Optional<String> longest = Stream.of("cat", "elephant", "dog", "hippopotamus")
    .reduce((a, b) -> a.length() >= b.length() ? a : b);
// Optional["hippopotamus"]
```

---

## Form 3 — `reduce(identity, BiFunction, BinaryOperator)` — The Parallel Form

```java
<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner)
```

This three-arg form exists for one reason: **reducing to a different type than the stream element type**, especially in parallel.

- `identity` — starting value of type `U` (the result type)
- `accumulator` — `(U result, T element) → U` — fold one element in
- `combiner` — `(U partial1, U partial2) → U` — merge two partial results

```java
// Sum of string lengths — result type (int) differs from element type (String)
int totalLength = Stream.of("Shop", "Sphere", "Order")
    .reduce(
        0,
        (sum, s) -> sum + s.length(),   // accumulator: int + String → int
        Integer::sum                     // combiner: int + int → int
    );
// 4 + 6 + 5 = 15
```

---

## Why the Combiner Exists — Parallel Split Model

In a **sequential** stream, the combiner is **never called**. The accumulator handles everything left to right.

In a **parallel** stream, the stream is split into partitions. Each partition runs the accumulator independently, producing a partial result. The combiner merges those partial results.

```
Sequential:  identity → acc(+s1) → acc(+s2) → acc(+s3) → result
                         ↑ combiner never called

Parallel:    Partition A: identity → acc(+s1) → acc(+s2) → partialA
             Partition B: identity → acc(+s3)             → partialB
             Merge:       combiner(partialA, partialB)     → result
```

```java
// Parallel — combiner IS called
int totalLength = Stream.of("Shop", "Sphere", "Order")
    .parallel()
    .reduce(
        0,
        (sum, s) -> sum + s.length(),
        Integer::sum   // called to merge partition results
    );
```

> **Interview trap:** In sequential streams you can pass `null` as the combiner and it works — but don't. In parallel streams, a null combiner throws NPE. Always provide a valid combiner.

---

## Associativity Requirement

For `reduce` to produce correct results in parallel, the accumulator must be **associative**:

```
(a op b) op c  ==  a op (b op c)
```

| Operation | Associative? |
|---|---|
| `+` (sum) | ✅ |
| `*` (product) | ✅ |
| `max`, `min` | ✅ |
| String concat | ✅ (but order-sensitive) |
| subtraction | ❌ `(10-3)-2 ≠ 10-(3-2)` |
| division | ❌ |
| average | ❌ (not directly reducible) |

Non-associative operations give different results in parallel vs sequential — a bug that only shows up under parallelism.

---

## `reduce` vs `collect` — When to Use Which

This is a guaranteed interview question.

| | `reduce` | `collect` |
|---|---|---|
| Result type | Immutable value | Mutable container |
| How it combines | Creates new object each step | Mutates existing container |
| Building collections | ❌ wrong tool | ✅ correct tool |
| Numeric aggregation | ✅ sum, product, max | Use primitive stream terminals |
| String joining | ❌ O(n²) — new String each step | ✅ `Collectors.joining()` uses StringBuilder |

**Why `reduce` is wrong for building collections:**

```java
// This is technically correct but WRONG approach
List<Integer> list = Stream.of(1, 2, 3)
    .reduce(
        new ArrayList<>(),
        (acc, e) -> { acc.add(e); return acc; },  // mutates identity object — broken in parallel
        (a, b) -> { a.addAll(b); return a; }
    );

// Correct
List<Integer> list = Stream.of(1, 2, 3)
    .collect(Collectors.toList());
```

The issue: `reduce` expects the identity object to be **immutable and reused** across partitions. Mutating it (via `add`) means all partitions share the same mutated list — broken.

**Why `reduce` for string join is O(n²):**

```java
// Each step creates a NEW String — O(n²) total
Stream.of("a", "b", "c", "d")
    .reduce("", (a, b) -> a + b);
// "" + "a" = "a" (new object)
// "a" + "b" = "ab" (new object)
// "ab" + "c" = "abc" (new object)
// "abc" + "d" = "abcd" (new object)

// Correct — StringBuilder internally, O(n)
Stream.of("a", "b", "c", "d")
    .collect(Collectors.joining());
```

---

## Practical `reduce` Patterns

### Finding max/min without Comparator (on primitives)

```java
// Max of ints
OptionalInt max = IntStream.of(3, 1, 4, 1, 5, 9)
    .reduce(Integer::max);

// Or with identity
int max = IntStream.of(3, 1, 4, 1, 5, 9)
    .reduce(Integer.MIN_VALUE, Integer::max);
```

---

### Composing Functions with `reduce`

```java
List<Function<Integer, Integer>> transforms = List.of(
    x -> x + 1,
    x -> x * 2,
    x -> x - 3
);

// Compose all functions into one pipeline
Function<Integer, Integer> combined = transforms.stream()
    .reduce(
        Function.identity(),
        Function::andThen
    );

System.out.println(combined.apply(5));
// (5+1)*2-3 = 9
```

---

### Running total (scan-like) — use `iterate` instead

`reduce` collapses to one value. For running totals, use `iterate` or collect with a custom Collector. This is a common trap — don't try to do it with `reduce`.

---

## ShopSphere Practice

```java
List<Order> orders = // orders with getAmount() and getItemCount()

// 1. Total revenue — prefer mapToDouble + sum
double revenue = orders.stream()
    .mapToDouble(Order::getAmount)
    .sum();

// 2. Same with reduce (shows the concept)
double revenue2 = orders.stream()
    .map(Order::getAmount)
    .reduce(0.0, Double::sum);

// 3. Most expensive order
Optional<Order> priciest = orders.stream()
    .reduce((a, b) -> a.getAmount() > b.getAmount() ? a : b);

// 4. Concatenate order IDs for a receipt
String receipt = orders.stream()
    .map(Order::getId)
    .collect(Collectors.joining(", ", "Orders: [", "]"));
// "Orders: [ORD-001, ORD-002, ORD-003]"
// NOT reduce — joining uses StringBuilder internally
```

---

## Interview Questions on This Topic

**Q: What is the identity contract in `reduce`, and what breaks if you violate it?**
The identity value must satisfy `identity op v == v` for every `v`. In sequential streams a wrong identity gives a wrong result (off by identity). In parallel it's multiplied — applied once per partition, so the error is amplified proportionally to the number of partitions.

**Q: When does the three-arg `reduce` combiner get called?**
Only in parallel streams. In sequential execution it is never invoked — the accumulator processes everything left to right. The combiner exists solely to merge partial results from parallel partitions.

**Q: Why is `reduce` the wrong tool for building a List?**
`reduce` expects the identity to be immutable and reusable across parallel partitions. Mutating an `ArrayList` identity object means all partitions share the same list — thread-unsafe and wrong. Use `collect` — it creates a fresh container per partition via the Supplier, mutates independently, then merges with the combiner.

**Q: Is string concatenation with `reduce` correct? Is it efficient?**
Correct for sequential — produces the right string. Not efficient — every step creates a new `String` object, making it O(n²). `Collectors.joining()` uses a `StringBuilder` internally, O(n). Always use `joining` for string accumulation.

**Q: What's the difference between `reduce` and primitive stream `sum()`?**
Both produce a sum. `sum()` on `IntStream`/`DoubleStream` is equivalent to `reduce(0, Integer::sum)` but is cleaner, more readable, and avoids boxing. For numeric aggregation, always prefer primitive stream terminals over `reduce`.

---

Say **next** and we go to **Phase 7 — Parallel Streams** — ForkJoin internals, when parallel wins and when it backfires, ordering semantics, stateful vs stateless, and custom thread pools.
