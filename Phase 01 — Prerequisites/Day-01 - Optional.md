
# Phase 1 — `Optional<T>`

## What Problem It Solves

Before Java 8, absent values were represented by `null`. This caused:

```java
String city = user.getAddress().getCity().toUpperCase();
// NullPointerException if any step returns null
```

You'd defensively write:
```java
if (user != null && user.getAddress() != null && user.getAddress().getCity() != null) {
    city = user.getAddress().getCity().toUpperCase();
}
```

`Optional<T>` is a **container object** that either holds a value or holds nothing. It forces the caller to explicitly handle the absent case instead of forgetting and hitting an NPE.

> **Important framing for interviews:** Optional is **not** a replacement for null everywhere. It's designed specifically as a **return type for methods that might not return a value**. Never use it as a field type, constructor parameter, or method parameter.

---

## Creating an Optional

```java
// 1. Definitely has a value — throws NullPointerException if you pass null
Optional<String> opt = Optional.of("ShopSphere");

// 2. Might be null — wraps if present, empty if null
Optional<String> opt = Optional.ofNullable(possiblyNullString);

// 3. Explicitly empty
Optional<String> opt = Optional.empty();
```

> **Interview trap:** `Optional.of(null)` throws NPE immediately. Use `ofNullable` when the value might be null.

---

## Consuming — The Right and Wrong Ways

### The wrong way (defeats the purpose)

```java
if (opt.isPresent()) {
    System.out.println(opt.get());
}
```

This is just null-check with extra steps. You traded `!= null` for `isPresent()`. Also — calling `get()` on an empty Optional throws `NoSuchElementException`. Don't use `get()` without `isPresent()`, and avoid `isPresent()` whenever a functional alternative exists.

---

### `ifPresent(Consumer)` — execute only if value exists

```java
Optional<String> name = Optional.of("Vidhan");
name.ifPresent(n -> System.out.println("Hello, " + n));
// Hello, Vidhan

Optional<String> empty = Optional.empty();
empty.ifPresent(n -> System.out.println("won't print"));
// nothing
```

---

### `orElse(T)` — default value if absent

```java
String result = Optional.ofNullable(nullableName).orElse("Guest");
```

**Critical interview point:** `orElse` **always evaluates its argument**, even if the Optional has a value.

```java
String result = opt.orElse(expensiveMethod()); 
// expensiveMethod() is called regardless of whether opt is empty
```

---

### `orElseGet(Supplier)` — lazy default

```java
String result = opt.orElseGet(() -> expensiveMethod());
// expensiveMethod() only called if opt is empty
```

Use `orElseGet` whenever the default involves a method call, DB lookup, or object construction.

---

### `orElseThrow(Supplier)` — throw if absent

```java
String result = opt.orElseThrow(() -> new OrderNotFoundException("Order not found"));
```

Java 10+ added a no-arg `orElseThrow()` that throws `NoSuchElementException` — but the Supplier form is cleaner for production code.

---

### `ifPresentOrElse(Consumer, Runnable)` — Java 9

```java
opt.ifPresentOrElse(
    value -> System.out.println("Found: " + value),
    ()    -> System.out.println("Not found")
);
```

---

## Transforming — `map` and `flatMap`

### `map(Function)` — transform the value if present

```java
Optional<String> name = Optional.of("shopsphere");
Optional<String> upper = name.map(String::toUpperCase);
// Optional["SHOPSPHERE"]

Optional<String> empty = Optional.empty();
Optional<String> result = empty.map(String::toUpperCase);
// Optional.empty — map is a no-op on empty
```

Real-world example:
```java
Optional<User> user = userRepo.findById(userId);
Optional<String> city = user.map(User::getAddress)
                            .map(Address::getCity);
```

---

### `flatMap(Function)` — when your mapper returns an Optional

```java
// map would give Optional<Optional<String>> — wrong
Optional<Optional<String>> wrong = user.map(User::getEmailOptional);

// flatMap flattens it to Optional<String> — correct
Optional<String> email = user.flatMap(User::getEmailOptional);
```

Rule: Use `map` when the mapper returns a plain value. Use `flatMap` when the mapper returns an `Optional`.

---

### `filter(Predicate)` — conditional presence

```java
Optional<String> code = Optional.of("DISCOUNT10");
Optional<String> valid = code.filter(c -> c.startsWith("DISCOUNT"));
// Optional["DISCOUNT10"] — passes filter

Optional<String> invalid = code.filter(c -> c.startsWith("PROMO"));
// Optional.empty — fails filter
```

---

### `or(Supplier<Optional>)` — Java 9, fallback Optional

```java
Optional<String> primary   = Optional.empty();
Optional<String> secondary = Optional.of("fallback");

Optional<String> result = primary.or(() -> secondary);
// Optional["fallback"]
```

Different from `orElseGet` — `or` returns an `Optional`, `orElseGet` unwraps to a value.

---

### `stream()` — Java 9, bridge to Stream API

```java
Optional<String> opt = Optional.of("hello");
opt.stream().map(String::toUpperCase).forEach(System.out::println);
// HELLO
```

Most useful when you have a `List<Optional<T>>` and want to filter empties and unwrap:

```java
List<Optional<String>> list = List.of(Optional.of("a"), Optional.empty(), Optional.of("b"));

List<String> values = list.stream()
    .flatMap(Optional::stream)   // Java 9
    .collect(Collectors.toList());
// ["a", "b"]
```

---

## The Chaining Pattern — Replacing Null-Guard Chains

```java
// Old way
String city = null;
if (user != null) {
    Address addr = user.getAddress();
    if (addr != null) {
        city = addr.getCity();
    }
}

// Optional way
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");
```

Clean, linear, no nested ifs, explicit default.

---

## What Optional is NOT for

```java
// ❌ Field type
public class User {
    private Optional<String> middleName; // wrong — use nullable String field
}

// ❌ Method parameter
public void process(Optional<Order> order) { } // wrong — use overloading or nullable

// ❌ Collection element
List<Optional<String>> list = ...; // wrong — filter nulls from the list instead

// ❌ Return type for collections
public Optional<List<Order>> getOrders() { } // wrong — return empty list instead
```

The rule: Optional is a **return type** for a single value that might not exist. It's a communication contract on the method signature — it tells callers "this might not return anything, handle it."

---

## Summary — Method Quick Reference

| Method | Arg | Returns | When to use |
|---|---|---|---|
| `of(value)` | T | `Optional<T>` | Value is definitely non-null |
| `ofNullable(value)` | T | `Optional<T>` | Value might be null |
| `empty()` | — | `Optional<T>` | Explicitly absent |
| `isPresent()` | — | boolean | Avoid — use functional methods |
| `get()` | — | T | Avoid — use orElse/orElseThrow |
| `ifPresent(consumer)` | Consumer | void | Side effect if present |
| `orElse(default)` | T | T | Cheap literal default |
| `orElseGet(supplier)` | Supplier | T | Expensive default |
| `orElseThrow(supplier)` | Supplier | T | Throw if absent |
| `map(fn)` | Function | `Optional<R>` | Transform value |
| `flatMap(fn)` | Function→Optional | `Optional<R>` | Mapper returns Optional |
| `filter(pred)` | Predicate | `Optional<T>` | Conditional keep |
| `or(supplier)` | Supplier→Optional | `Optional<T>` | Fallback Optional (Java 9) |
| `stream()` | — | `Stream<T>` | Bridge to streams (Java 9) |

---

## Interview Questions on This Topic

**Q: What's the difference between `orElse` and `orElseGet`?**
`orElse` evaluates its argument eagerly — always. `orElseGet` takes a Supplier and only invokes it if the Optional is empty. For any non-trivial default (method call, object creation), `orElseGet` is correct.

**Q: Why shouldn't Optional be used as a method parameter?**
It forces callers to wrap their values in Optional before passing, which is awkward. It also doesn't actually prevent passing `null` — you can still do `method(null)` and get an NPE inside. Overloading or `@Nullable` annotations are cleaner solutions.

**Q: What does `map` return on an empty Optional?**
`Optional.empty()` — `map` short-circuits without calling the function.

**Q: `Optional.of(null)` vs `Optional.ofNullable(null)` — what happens?**
`Optional.of(null)` throws `NullPointerException` immediately. `Optional.ofNullable(null)` returns `Optional.empty()`.

**Q: In a Stream pipeline, how do you go from `List<Optional<T>>` to `List<T>` with empties removed?**
```java
list.stream()
    .flatMap(Optional::stream)   // Java 9
    .collect(Collectors.toList());

// Java 8 equivalent
list.stream()
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());
```

---

That's all three prerequisites done. Say **next** and we enter **Phase 2 — Stream Fundamentals** — what a Stream actually is at the JVM level, the source → intermediate → terminal model, laziness, and all creation methods.
