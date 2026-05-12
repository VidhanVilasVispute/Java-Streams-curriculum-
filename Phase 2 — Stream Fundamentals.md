# Phase 2 — Stream Fundamentals

## What a Stream Actually Is

A Stream is **not a data structure**. It does not store elements. It is a **pipeline of computational steps** over a source.

The formal definition: a Stream is a sequence of elements supporting sequential and parallel aggregate operations.

Three things make up every stream pipeline:

```
Source  →  Intermediate Operations (zero or more)  →  Terminal Operation (exactly one)
```

```java
List.of("apple", "banana", "cherry")     // source
    .stream()
    .filter(s -> s.length() > 5)         // intermediate
    .map(String::toUpperCase)            // intermediate
    .collect(Collectors.toList());       // terminal
```

---

## Two Critical Properties

### 1. Lazy

Intermediate operations **do not execute** when you call them. They register a step in the pipeline. Nothing runs until a terminal operation is invoked.

```java
Stream<String> pipeline = List.of("a", "bb", "ccc")
    .stream()
    .filter(s -> {
        System.out.println("filtering: " + s);
        return s.length() > 1;
    });

// Nothing printed yet — no terminal operation called
System.out.println("Pipeline built, nothing ran");

pipeline.collect(Collectors.toList()); // NOW filtering runs
```

Output:
```
Pipeline built, nothing ran
filtering: a
filtering: bb
filtering: ccc
```

**Why laziness matters:** With `limit()` + `findFirst()`, Java short-circuits — it processes only as many elements as needed, not the whole source. Critical for infinite streams and large datasets.

---

### 2. Single-Use (Non-Reusable)

Once a terminal operation is called, the stream is **consumed**. Calling another terminal operation on the same stream throws `IllegalStateException`.

```java
Stream<String> stream = List.of("a", "b").stream();
stream.forEach(System.out::println); // fine
stream.forEach(System.out::println); // IllegalStateException: stream has already been operated upon or closed
```

> **Interview question right here.** If you need to process the same data twice, call `.stream()` again on the source, or collect to a list first.

---

## Encounter Order

A stream's encounter order depends on its source:
- `List` → ordered (encounter order preserved)
- `Set` → unordered (no encounter order)
- `Map.entrySet()` → unordered

This matters for `findFirst()` vs `findAny()`, and for `forEachOrdered()` vs `forEach()` in parallel streams. We'll go deep on this in Phase 7.

---

## Creating Streams — All Forms

### From a Collection

```java
List<String> list = List.of("a", "b", "c");

Stream<String> sequential = list.stream();
Stream<String> parallel   = list.parallelStream();
```

---

### `Stream.of(...)`

```java
Stream<String> s = Stream.of("x", "y", "z");
Stream<Integer> nums = Stream.of(1, 2, 3);

// Single element
Stream<String> one = Stream.of("only");

// Empty stream
Stream<String> empty = Stream.empty();
```

---

### `Arrays.stream(array)`

```java
String[] arr = {"a", "b", "c"};
Stream<String> s = Arrays.stream(arr);

// Supports range — only this subarray
Stream<String> sub = Arrays.stream(arr, 1, 3); // "b", "c"

// For primitives — returns IntStream, not Stream<Integer>
int[] nums = {1, 2, 3};
IntStream is = Arrays.stream(nums);
```

---

### `Stream.iterate(seed, UnaryOperator)` — Java 8

Produces an **infinite sequential ordered** stream.

```java
// 0, 1, 2, 3, 4 ...
Stream<Integer> naturals = Stream.iterate(0, n -> n + 1);
naturals.limit(5).forEach(System.out::println);
// 0 1 2 3 4
```

```java
// Powers of 2
Stream.iterate(1, n -> n * 2)
      .limit(8)
      .forEach(System.out::println);
// 1 2 4 8 16 32 64 128
```

**Java 9 added a predicate overload** — acts like a for-loop with a stop condition:

```java
// iterate(seed, hasNext, next) — terminates when predicate is false
Stream.iterate(0, n -> n < 10, n -> n + 2)
      .forEach(System.out::println);
// 0 2 4 6 8
```

---

### `Stream.generate(Supplier)` — infinite, unordered

Produces an **infinite sequential unordered** stream by calling the Supplier repeatedly.

```java
// Infinite stream of "hello"
Stream.generate(() -> "hello").limit(3).forEach(System.out::println);

// Random numbers
Stream.generate(Math::random).limit(5).forEach(System.out::println);

// UUIDs
Stream.generate(UUID::randomUUID).limit(3).forEach(System.out::println);
```

> **`iterate` vs `generate`:** `iterate` produces values based on the previous value — deterministic, ordered. `generate` calls a Supplier independently each time — stateless preferred, but the Supplier can be stateful (though that's not thread-safe in parallel).

---

### `IntStream.range()` and `IntStream.rangeClosed()`

```java
IntStream.range(0, 5)        // 0, 1, 2, 3, 4  — exclusive end
IntStream.rangeClosed(1, 5)  // 1, 2, 3, 4, 5  — inclusive end
```

Replaces index-based for loops in pipelines:

```java
// Old
for (int i = 0; i < 10; i++) { process(i); }

// Stream
IntStream.range(0, 10).forEach(i -> process(i));

// With result
List<String> labels = IntStream.rangeClosed(1, 5)
    .mapToObj(i -> "Item-" + i)
    .collect(Collectors.toList());
// ["Item-1", "Item-2", "Item-3", "Item-4", "Item-5"]
```

---

### `Files.lines(Path)` — lazy file streaming

```java
Path path = Path.of("orders.csv");

try (Stream<String> lines = Files.lines(path)) {
    lines.filter(line -> line.startsWith("ORDER"))
         .map(String::trim)
         .forEach(System.out::println);
}
```

> **Always use try-with-resources.** `Files.lines()` opens an I/O resource. If you don't close the stream, you leak a file handle. The stream's `close()` method closes the underlying reader.

---

### `BufferedReader.lines()`

```java
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
    reader.lines()
          .filter(s -> !s.isBlank())
          .collect(Collectors.toList());
}
```

---

### From a String — `chars()` and `codePoints()`

```java
"hello".chars()                    // IntStream of char values
       .mapToObj(c -> (char) c)    // back to Character
       .forEach(System.out::println);
```

---

## Intermediate Operations — Overview

All intermediate operations are **lazy** and return a new `Stream`.

| Operation | What it does |
|---|---|
| `filter(Predicate)` | Keep elements matching predicate |
| `map(Function)` | Transform each element |
| `flatMap(Function)` | Transform + flatten nested streams |
| `distinct()` | Remove duplicates (uses `equals`) |
| `sorted()` | Natural order sort |
| `sorted(Comparator)` | Custom order sort |
| `peek(Consumer)` | Side-effect without consuming (debug) |
| `limit(long)` | Truncate to max n elements |
| `skip(long)` | Skip first n elements |

---

## Terminal Operations — Overview

Terminal operations are **eager** — they trigger the pipeline to execute.

| Operation | Returns | What it does |
|---|---|---|
| `forEach(Consumer)` | void | Consume each element |
| `forEachOrdered(Consumer)` | void | Consume in encounter order |
| `collect(Collector)` | R | Mutable reduction into container |
| `count()` | long | Count elements |
| `reduce(identity, BinaryOperator)` | T | Immutable fold |
| `findFirst()` | `Optional<T>` | First element |
| `findAny()` | `Optional<T>` | Any element (faster in parallel) |
| `anyMatch(Predicate)` | boolean | Short-circuits on first true |
| `allMatch(Predicate)` | boolean | Short-circuits on first false |
| `noneMatch(Predicate)` | boolean | Short-circuits on first true |
| `min(Comparator)` | `Optional<T>` | Minimum element |
| `max(Comparator)` | `Optional<T>` | Maximum element |
| `toArray()` | `Object[]` | Collect to array |

---

## How Laziness + Short-Circuiting Works Internally

Consider this pipeline:

```java
List.of("apple", "ant", "banana", "avocado", "cherry")
    .stream()
    .filter(s -> s.startsWith("a"))
    .map(String::toUpperCase)
    .findFirst();
```

Execution is **vertical**, not horizontal. Java doesn't run all of `filter` first, then all of `map`. It pushes **one element at a time** through the whole pipeline:

```
"apple"  → filter ✅ → map → "APPLE" → findFirst() → DONE. Stop.
```

`"ant"`, `"banana"`, `"avocado"`, `"cherry"` are **never processed**. This is short-circuit evaluation.

```
Without short-circuit:  filter all 5, map 2, return first
With short-circuit:     filter 1, map 1, return — done
```

This is why `limit()` makes infinite streams safe — it short-circuits before the stream reaches infinity.

---

## ShopSphere Practice Problem

```java
// Given this list of product names:
List<String> products = List.of(
    "Laptop", "Mouse", "Keyboard", "Monitor", "USB Hub",
    "Webcam", "Headphones", "Microphone", "Desk Lamp", "Chair"
);

// Task:
// 1. Filter products whose name length > 5
// 2. Convert to uppercase
// 3. Sort alphabetically
// 4. Collect to a new List

List<String> result = products.stream()
    .filter(p -> p.length() > 5)
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());

// Result: [HEADPHONES, KEYBOARD, MICROPHONE, MONITOR, USB HUB, WEBCAM]
```

---

## Interview Questions on This Topic

**Q: What is lazy evaluation and why does it matter?**
Intermediate operations don't execute until a terminal operation is called. This allows short-circuit optimizations — with `limit` or `findFirst`, only the minimum elements needed are processed, which is essential for performance on large or infinite sources.

**Q: What happens if you call two terminal operations on the same stream?**
`IllegalStateException: stream has already been operated upon or closed`. A stream is single-use. Re-create it from the source, or collect to a list and stream again.

**Q: What's the difference between `Stream.iterate` and `Stream.generate`?**
`iterate` builds each value from the previous one — deterministic, ordered, like a loop. `generate` calls a Supplier independently each time — no state dependency between elements, unordered. For stateful generation (sequence numbers), use `iterate`. For independent values (random, UUID), use `generate`.

**Q: Why must `Files.lines()` be used in a try-with-resources block?**
It opens a file handle that must be closed. If the stream is not closed (via `close()` or try-with-resources), the file handle leaks. The Stream interface extends `AutoCloseable` specifically for this.

**Q: Is `sorted()` a stateful or stateless intermediate operation? Why does it matter?**
Stateful — it must see all elements before it can emit any. This breaks the vertical one-element-at-a-time execution model for that stage, buffers the entire stream, and kills short-circuit benefits before it. In parallel streams, it's also a synchronization bottleneck.

---

Say **next** and we go to **Phase 3 — Collectors in Depth** — `groupingBy`, downstream collectors, `toMap`, `joining`, `teeing`, and custom Collector — the section that separates mid-level from senior.
