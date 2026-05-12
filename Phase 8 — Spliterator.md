# Phase 8 — Spliterator

## What a Spliterator Is

`Spliterator` (Splittable Iterator) is the data source abstraction behind every Stream. It does two things a regular `Iterator` cannot:

1. **Traverse** elements one by one — like Iterator
2. **Split** itself into two halves — enabling parallel processing

```java
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);  // process one element
    Spliterator<T> trySplit();                        // split into two
    long estimateSize();                              // estimated remaining elements
    int characteristics();                            // bitmask of properties
}
```

Every time you call `.stream()` or `.parallelStream()`, the collection creates a Spliterator internally. The stream framework uses it to traverse sequentially or split for parallel execution.

---

## `tryAdvance` vs `forEachRemaining`

```java
// tryAdvance — process ONE element, return false if none left
Spliterator<String> sp = List.of("a","b","c").spliterator();

sp.tryAdvance(s -> System.out.println("got: " + s)); // got: a
sp.tryAdvance(s -> System.out.println("got: " + s)); // got: b

// forEachRemaining — process ALL remaining elements
sp.forEachRemaining(s -> System.out.println("remaining: " + s)); // remaining: c
```

`tryAdvance` is how the stream framework pulls elements lazily. `forEachRemaining` is used when sequential processing of all remaining elements is decided.

---

## `trySplit` — How Parallel Splitting Works

`trySplit()` attempts to partition the source roughly in half:
- Returns a new `Spliterator` covering the first half
- The original Spliterator now covers the second half
- Returns `null` if splitting is not possible or not worth it

```java
List<Integer> list = new ArrayList<>(List.of(1,2,3,4,5,6,7,8));
Spliterator<Integer> original = list.spliterator();

Spliterator<Integer> firstHalf = original.trySplit();
// firstHalf covers [1,2,3,4]
// original now covers [5,6,7,8]

Spliterator<Integer> quarter = firstHalf.trySplit();
// quarter covers [1,2]
// firstHalf now covers [3,4]
```

The ForkJoinPool keeps calling `trySplit()` recursively until the partitions are small enough (below a threshold) or `trySplit()` returns null.

---

## `estimateSize()`

Returns the estimated number of elements remaining. Used by the framework to decide whether further splitting is worthwhile.

```java
Spliterator<Integer> sp = List.of(1,2,3,4,5).spliterator();
System.out.println(sp.estimateSize()); // 5

sp.trySplit();
System.out.println(sp.estimateSize()); // ~2 (second half)
```

If the size is unknown (e.g., a stream over a network source), return `Long.MAX_VALUE`.

---

## Characteristics — The Bitmask

Characteristics are int flags that describe properties of the Spliterator's source. The stream framework uses them to apply optimizations.

| Characteristic | Constant | Meaning |
|---|---|---|
| `ORDERED` | `0x00000010` | Elements have a defined encounter order |
| `DISTINCT` | `0x00000001` | No duplicate elements |
| `SORTED` | `0x00000004` | Elements are in sorted order |
| `SIZED` | `0x00000040` | `estimateSize()` is exact |
| `NONNULL` | `0x00000100` | No null elements |
| `IMMUTABLE` | `0x00000400` | Source cannot be modified |
| `CONCURRENT` | `0x00001000` | Source can be modified concurrently |
| `SUBSIZED` | `0x00004000` | Splits are also `SIZED` |

```java
Spliterator<String> listSp = List.of("a","b","c").spliterator();
System.out.println(listSp.hasCharacteristic(Spliterator.ORDERED)); // true
System.out.println(listSp.hasCharacteristic(Spliterator.SIZED));   // true
System.out.println(listSp.hasCharacteristic(Spliterator.DISTINCT)); // false

Spliterator<String> setSp = Set.of("a","b","c").spliterator();
System.out.println(setSp.hasCharacteristic(Spliterator.ORDERED)); // false
System.out.println(setSp.hasCharacteristic(Spliterator.DISTINCT)); // true
```

---

## How Characteristics Enable Optimizations

The stream framework reads characteristics to skip unnecessary work:

- `SIZED` → `count()` can return `estimateSize()` directly, no traversal needed
- `DISTINCT` → `distinct()` operation is a no-op, skipped
- `SORTED` → `sorted()` is a no-op if already sorted with same comparator
- `ORDERED` absent → `findFirst()` can behave like `findAny()`, no ordering coordination needed
- `IMMUTABLE` or `CONCURRENT` → no need to check for concurrent modification

```java
// count() on an ArrayList stream — O(1), reads size directly via SIZED
long count = new ArrayList<>(millions).stream().count();

// count() on a filtered stream — must traverse, SIZED is lost after filter
long count = list.stream().filter(...).count(); // O(n)
```

---

## Source Splittability — Why It Matters for Parallel

Not all sources split well:

```java
// ArrayList — SUBSIZED + SIZED, splits at midpoint index in O(1)
new ArrayList<>(list).spliterator().trySplit(); // fast, balanced

// LinkedList — must traverse to midpoint, O(n/2) per split
new LinkedList<>(list).spliterator().trySplit(); // slow, poor balance

// HashSet — splits by internal bucket structure, unbalanced possible
new HashSet<>(set).spliterator().trySplit(); // moderate

// IntStream.range — perfect O(1) split at midpoint
IntStream.range(0, 1_000_000).spliterator().trySplit(); // ideal
```

This is why `ArrayList` and primitive arrays are the best parallel stream sources.

---

## Implementing a Custom Spliterator

The real interview value. When would you need this?

**Scenario:** You have a paginated REST API or database query. You want to stream results lazily without loading all pages into memory first. No built-in Spliterator supports this.

```java
// Simulated paginated source
class PagedApiClient {
    List<Order> fetchPage(int page, int pageSize) { ... }
    int getTotalCount() { ... }
}
```

```java
public class PagedOrderSpliterator implements Spliterator<Order> {

    private final PagedApiClient client;
    private final int pageSize;
    private int currentPage;
    private final int totalPages;
    private List<Order> currentBatch;
    private int indexInBatch;

    public PagedOrderSpliterator(PagedApiClient client, int pageSize) {
        this.client    = client;
        this.pageSize  = pageSize;
        this.currentPage = 0;
        this.totalPages  = (int) Math.ceil(
            (double) client.getTotalCount() / pageSize
        );
        this.currentBatch = Collections.emptyList();
        this.indexInBatch = 0;
    }

    @Override
    public boolean tryAdvance(Consumer<? super Order> action) {
        // If current batch exhausted, fetch next page
        if (indexInBatch >= currentBatch.size()) {
            if (currentPage >= totalPages) return false; // done

            currentBatch  = client.fetchPage(currentPage++, pageSize);
            indexInBatch  = 0;

            if (currentBatch.isEmpty()) return false;
        }

        action.accept(currentBatch.get(indexInBatch++));
        return true;
    }

    @Override
    public Spliterator<Order> trySplit() {
        // Don't support splitting — sequential only
        // For parallel support you'd split page ranges
        return null;
    }

    @Override
    public long estimateSize() {
        return (long)(totalPages - currentPage) * pageSize;
    }

    @Override
    public int characteristics() {
        return ORDERED | NONNULL;
        // SIZED not set — estimate is approximate
        // No IMMUTABLE — source is a live API
    }
}
```

**Using the custom Spliterator:**

```java
PagedApiClient client = new PagedApiClient();
Spliterator<Order> spliterator = new PagedOrderSpliterator(client, 100);

// Wrap in a Stream — lazy, pages fetched on demand
Stream<Order> orderStream = StreamSupport.stream(spliterator, false);
// false = sequential; true = parallel

orderStream
    .filter(o -> o.getAmount() > 5000)
    .map(Order::getId)
    .forEach(System.out::println);
// Fetches pages lazily as stream consumes elements
// Never loads all orders into memory at once
```

`StreamSupport.stream(Spliterator, boolean)` is how you turn any Spliterator into a Stream. This is how all built-in collections create their streams internally.

---

## Adding Parallel Support to Custom Spliterator

To support splitting, divide the page range:

```java
@Override
public Spliterator<Order> trySplit() {
    int remainingPages = totalPages - currentPage;
    if (remainingPages <= 1) return null; // not worth splitting

    int midPage = currentPage + remainingPages / 2;

    // Create a new Spliterator for the first half
    PagedOrderSpliterator prefix =
        new PagedOrderSpliterator(client, pageSize, currentPage, midPage);

    // This Spliterator takes the second half
    this.currentPage  = midPage;
    this.currentBatch = Collections.emptyList();
    this.indexInBatch = 0;

    return prefix;
}
```

Now:
```java
Stream<Order> parallel = StreamSupport.stream(spliterator, true); // parallel=true
```

The ForkJoinPool will call `trySplit()` recursively, assigning page ranges to different threads.

---

## `AbstractSpliterator` — Shortcut for Custom Implementation

If you only need sequential traversal (no splitting), extend `AbstractSpliterator`:

```java
public class OrderSpliterator extends Spliterators.AbstractSpliterator<Order> {

    private final Iterator<Order> source;

    public OrderSpliterator(Iterator<Order> source, long size) {
        super(size, ORDERED | SIZED | NONNULL);
        this.source = source;
    }

    @Override
    public boolean tryAdvance(Consumer<? super Order> action) {
        if (!source.hasNext()) return false;
        action.accept(source.next());
        return true;
    }
    // trySplit() provided by AbstractSpliterator with a default strategy
}
```

`AbstractSpliterator` provides a default `trySplit()` that buffers elements in batches — not ideal for parallel but functional.

---

## Real-World Use Cases

| Use case | Why custom Spliterator |
|---|---|
| Paginated REST API | Fetch pages lazily as stream demands |
| JDBC `ResultSet` | Stream rows without loading all into memory |
| File processed in chunks | Split by byte ranges for parallel processing |
| Graph traversal | BFS/DFS as a stream |
| Custom data structure | Doubly-linked ring buffer, skip list, etc. |

**ResultSet as a Spliterator** — a common interview answer for "how would you stream 10 million DB rows?":

```java
public class ResultSetSpliterator extends Spliterators.AbstractSpliterator<Map<String, Object>> {

    private final ResultSet rs;

    public ResultSetSpliterator(ResultSet rs) {
        super(Long.MAX_VALUE, ORDERED | NONNULL);
        this.rs = rs;
    }

    @Override
    public boolean tryAdvance(Consumer<? super Map<String, Object>> action) {
        try {
            if (!rs.next()) return false;
            Map<String, Object> row = new LinkedHashMap<>();
            ResultSetMetaData meta = rs.getMetaData();
            for (int i = 1; i <= meta.getColumnCount(); i++) {
                row.put(meta.getColumnName(i), rs.getObject(i));
            }
            action.accept(row);
            return true;
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
}

// Usage
Stream<Map<String, Object>> rows = StreamSupport.stream(
    new ResultSetSpliterator(resultSet), false
);
rows.filter(row -> (Double) row.get("amount") > 5000)
    .forEach(System.out::println);
```

---

## Interview Questions on This Topic

**Q: What is a Spliterator and how does it differ from an Iterator?**
Both traverse elements sequentially via `tryAdvance` / `hasNext+next`. Spliterator adds `trySplit()` — the ability to split the source into two halves for parallel processing — plus `estimateSize()` and `characteristics()` which give the stream framework hints for optimization. Iterator has none of these.

**Q: What does `trySplit()` return when splitting is not possible?**
`null`. The framework treats a null return as "this source cannot be split further" and processes the remaining elements sequentially on the current thread.

**Q: Why does a `LinkedList` parallelize poorly compared to `ArrayList`?**
`ArrayList`'s Spliterator has `SUBSIZED` + `SIZED` and splits at a midpoint index in O(1). `LinkedList` must traverse to find the midpoint — O(n/2) per split — and the resulting splits may be unbalanced. The overhead of splitting outweighs parallelism gains.

**Q: How do you create a Stream from a custom data source?**
Implement `Spliterator<T>` (or extend `AbstractSpliterator`), then call `StreamSupport.stream(spliterator, parallel)`. The boolean flag controls sequential vs parallel.

**Q: What characteristics would you set on a Spliterator for a List-backed source?**
`ORDERED` (list has defined order) + `SIZED` (exact size known) + `SUBSIZED` (splits are also sized) + `IMMUTABLE` if the list won't change. You would NOT set `DISTINCT` (lists allow duplicates) or `SORTED` (unless the list is sorted).

**Q: How would you stream 10 million rows from a database without loading them all into memory?**
Wrap a JDBC `ResultSet` in a custom `Spliterator` that calls `rs.next()` in `tryAdvance`. Pass `Long.MAX_VALUE` as size (unknown), set `ORDERED | NONNULL`. Wrap with `StreamSupport.stream(spliterator, false)` for sequential processing. Combined with `fetchSize` on the JDBC Statement, this streams rows in batches from the DB without materializing the full result set.

---

Say **next** and we move to **Phase 9 — Advanced Patterns** — lazy evaluation chains, infinite streams, exception handling in streams, the custom three-arg `collect`, debugging with `peek`, and writing a full custom Collector from scratch.
