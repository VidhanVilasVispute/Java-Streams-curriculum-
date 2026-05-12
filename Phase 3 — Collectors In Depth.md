# Phase 3 — Collectors In Depth

## What a Collector Is

`collect()` is a **mutable reduction** — it folds stream elements into a mutable container (List, Map, StringBuilder, etc.) using three steps:

1. **Supplier** — create the container
2. **Accumulator** — add one element to the container
3. **Combiner** — merge two containers (used in parallel)

`Collectors` is a utility class of ready-made Collector implementations. You'll use it constantly.

---

## `toList()`, `toSet()`, `toUnmodifiableList()`

```java
List<String> list = stream.collect(Collectors.toList());
// Returns an ArrayList — mutable, allows nulls, no order guarantee contract

Set<String> set = stream.collect(Collectors.toSet());
// Returns a HashSet — no duplicates, no order

List<String> immutable = stream.collect(Collectors.toUnmodifiableList());
// Java 10 — throws UnsupportedOperationException on modification
```

**Java 16+:** `stream.toList()` — shorthand, returns an unmodifiable list, disallows nulls.

```java
List<String> result = stream.toList(); // Java 16+, unmodifiable
```

> **Interview trap:** `Collectors.toList()` does NOT guarantee which List implementation is returned. Don't rely on it being an ArrayList in code that casts or depends on ArrayList-specific behavior.

---

## `toMap(keyMapper, valueMapper)`

```java
// Map<productId, productName>
Map<Long, String> productMap = products.stream()
    .collect(Collectors.toMap(
        Product::getId,
        Product::getName
    ));
```

**Duplicate key exception** — if two elements produce the same key, it throws `IllegalStateException: Duplicate key`.

Fix with the **three-arg overload** — provide a merge function:

```java
Map<String, Integer> wordCount = words.stream()
    .collect(Collectors.toMap(
        w -> w,           // key
        w -> 1,           // value
        Integer::sum      // merge — sum values on duplicate key
    ));
```

**Four-arg overload** — control the Map implementation:

```java
Map<Long, String> linkedMap = products.stream()
    .collect(Collectors.toMap(
        Product::getId,
        Product::getName,
        (a, b) -> a,          // keep first on duplicate
        LinkedHashMap::new    // use LinkedHashMap to preserve insertion order
    ));
```

---

## `groupingBy(classifier)`

Groups elements into `Map<K, List<V>>`.

```java
// Group orders by status
Map<OrderStatus, List<Order>> byStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));
// { PENDING -> [o1, o3], SHIPPED -> [o2], DELIVERED -> [o4, o5] }
```

---

### Two-arg `groupingBy` — with downstream collector

This is where it gets powerful. The second argument is another Collector applied **to each group**.

```java
// Group by department, count employees in each
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));
// { "Engineering" -> 12, "Sales" -> 7, "HR" -> 3 }
```

---

## Downstream Collectors — The Advanced Part

These are Collectors designed to be nested inside `groupingBy`.

### `counting()`

```java
Map<String, Long> ordersByStatus = orders.stream()
    .collect(Collectors.groupingBy(
        o -> o.getStatus().name(),
        Collectors.counting()
    ));
```

---

### `summingInt / summingLong / summingDouble`

```java
// Total revenue per category
Map<String, Double> revenueByCategory = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCategory,
        Collectors.summingDouble(Order::getAmount)
    ));
```

---

### `averagingInt / averagingDouble`

```java
// Average salary per department
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));
```

---

### `mapping(mapper, downstream)`

Transform each element before collecting within the group.

```java
// Group by department, collect just the names (not the whole Employee)
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));
// { "Engineering" -> ["Alice", "Bob"], "Sales" -> ["Charlie"] }
```

---

### `joining()`

Concatenates strings. Three overloads:

```java
// No args — plain concatenation
String result = Stream.of("a", "b", "c").collect(Collectors.joining());
// "abc"

// Delimiter
String csv = Stream.of("apple", "banana", "cherry")
    .collect(Collectors.joining(", "));
// "apple, banana, cherry"

// Delimiter + prefix + suffix
String json = Stream.of("id", "name", "price")
    .collect(Collectors.joining(", ", "[", "]"));
// "[id, name, price]"
```

As a downstream:

```java
// Group by dept, join employee names with comma
Map<String, String> nameStringByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.joining(", "))
    ));
// { "Engineering" -> "Alice, Bob, Carol" }
```

---

### `summarizingInt / Long / Double`

Returns a `IntSummaryStatistics` object with count, sum, min, max, and average — **in a single pass**.

```java
IntSummaryStatistics stats = orders.stream()
    .collect(Collectors.summarizingInt(Order::getItemCount));

System.out.println(stats.getCount());   // total orders
System.out.println(stats.getSum());     // total items
System.out.println(stats.getAverage()); // avg items per order
System.out.println(stats.getMin());     // min items in an order
System.out.println(stats.getMax());     // max items in an order
```

> **Interview point:** This is more efficient than computing count, sum, and average in three separate stream passes.

---

### `minBy / maxBy`

```java
// Highest paid employee per department
Map<String, Optional<Employee>> topEarnerByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))
    ));
```

Returns `Optional` because a group could theoretically be empty.

---

## `partitioningBy(Predicate)`

Special case of groupingBy — always produces `Map<Boolean, List<T>>` with exactly two keys: `true` and `false`.

```java
Map<Boolean, List<Product>> partitioned = products.stream()
    .collect(Collectors.partitioningBy(p -> p.getPrice() > 1000));

List<Product> expensive = partitioned.get(true);
List<Product> affordable = partitioned.get(false);
```

With downstream:

```java
// Count of expensive vs affordable products
Map<Boolean, Long> countPartition = products.stream()
    .collect(Collectors.partitioningBy(
        p -> p.getPrice() > 1000,
        Collectors.counting()
    ));
// { true -> 12, false -> 38 }
```

---

## `collectingAndThen(downstream, finisher)`

Applies a finishing function to the result of another Collector.

```java
// Collect to list, then make unmodifiable
List<String> immutable = stream.collect(
    Collectors.collectingAndThen(
        Collectors.toList(),
        Collections::unmodifiableList
    )
);
```

```java
// Group, count, then get the entry with max count
Optional<Map.Entry<String, Long>> topDept = employees.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.groupingBy(Employee::getDepartment, Collectors.counting()),
        map -> map.entrySet().stream().max(Map.Entry.comparingByValue())
    ));
```

---

## `teeing(downstream1, downstream2, merger)` — Java 12

Passes each element to **two collectors simultaneously**, then merges the results. Single pass over the stream.

```java
// Get both sum and count in one pass, compute average
double average = orders.stream()
    .collect(Collectors.teeing(
        Collectors.summingDouble(Order::getAmount),  // result1: total
        Collectors.counting(),                        // result2: count
        (total, count) -> total / count               // merge
    ));
```

```java
// Partition into two lists in one pass
record Result(List<Product> expensive, List<Product> cheap) {}

Result r = products.stream()
    .collect(Collectors.teeing(
        Collectors.filtering(p -> p.getPrice() > 1000, Collectors.toList()),
        Collectors.filtering(p -> p.getPrice() <= 1000, Collectors.toList()),
        Result::new
    ));
```

---

## Multi-Level `groupingBy`

Group by department, then by seniority level within each department:

```java
Map<String, Map<String, List<Employee>>> grouped = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getSeniority)
    ));
// { "Engineering" -> { "SENIOR" -> [...], "JUNIOR" -> [...] } }
```

---

## Writing a Custom Collector — `Collector<T, A, R>`

Required for senior-level interviews. Four components:

```java
public interface Collector<T, A, R> {
    Supplier<A>          supplier();      // create empty container
    BiConsumer<A, T>     accumulator();   // fold one element in
    BinaryOperator<A>    combiner();      // merge two containers (parallel)
    Function<A, R>       finisher();      // convert container to result
    Set<Characteristics> characteristics();
}
```

**Example: Custom Collector that collects to an ImmutableList**

```java
public class ImmutableListCollector<T>
        implements Collector<T, List<T>, List<T>> {

    @Override
    public Supplier<List<T>> supplier() {
        return ArrayList::new;
    }

    @Override
    public BiConsumer<List<T>, T> accumulator() {
        return List::add;
    }

    @Override
    public BinaryOperator<List<T>> combiner() {
        return (a, b) -> { a.addAll(b); return a; };
    }

    @Override
    public Function<List<T>, List<T>> finisher() {
        return Collections::unmodifiableList;
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Set.of(); // not IDENTITY_FINISH since finisher transforms
    }
}

// Usage
List<String> result = stream.collect(new ImmutableListCollector<>());
```

**Characteristics:**
- `IDENTITY_FINISH` — finisher is identity function, can be skipped
- `CONCURRENT` — accumulator can be called concurrently (thread-safe container)
- `UNORDERED` — result doesn't depend on encounter order

---

## ShopSphere Practice Problem

```java
// Given List<Employee> with fields: name, department, salary
// Task: group by department, find average salary per dept,
// then find the department with the highest average salary.

Map<String, Double> avgByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

Optional<Map.Entry<String, Double>> topDept = avgByDept.entrySet()
    .stream()
    .max(Map.Entry.comparingByValue());

topDept.ifPresent(e ->
    System.out.println(e.getKey() + " → " + e.getValue())
);

// Or in one pipeline with collectingAndThen:
Optional<Map.Entry<String, Double>> top = employees.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.averagingDouble(Employee::getSalary)
        ),
        map -> map.entrySet().stream().max(Map.Entry.comparingByValue())
    ));
```

---

## Interview Questions on This Topic

**Q: What's the difference between `collect` and `reduce`?**
`reduce` produces an immutable result by combining elements with a BinaryOperator. `collect` uses a mutable container — it's more efficient when building collections because it avoids creating intermediate objects on every fold step. Never use `reduce` to build a List; use `collect`.

**Q: `groupingBy` vs `partitioningBy` — when do you choose each?**
`partitioningBy` is always `Map<Boolean, List<T>>` — use it when you have a binary condition (above/below threshold, pass/fail). `groupingBy` is `Map<K, List<V>>` for any number of groups. `partitioningBy` is slightly more efficient for boolean splits because it always has exactly two buckets.

**Q: What happens with `toMap` when there are duplicate keys?**
`IllegalStateException: Duplicate key <value>`. Always provide a merge function as the third argument when duplicates are possible.

**Q: Why does `maxBy` return `Optional`?**
Because when used as a downstream collector inside `groupingBy`, a group could be empty in theory. In practice when used directly on a non-empty stream it will have a value, but the return type is `Optional<T>` by contract.

**Q: What are Collector Characteristics and why do they matter?**
They are hints to the stream framework. `IDENTITY_FINISH` allows skipping the finisher call. `UNORDERED` allows parallel collectors to skip reordering. `CONCURRENT` allows a single shared container to be accumulated into concurrently without a combiner step. Misusing them (e.g., marking a non-thread-safe accumulator as `CONCURRENT`) causes data corruption.

---

Say **next** and we go to **Phase 4 — `flatMap` and the Nested Structure Problem** — the most misunderstood operation, with real ShopSphere-grounded exercises.
