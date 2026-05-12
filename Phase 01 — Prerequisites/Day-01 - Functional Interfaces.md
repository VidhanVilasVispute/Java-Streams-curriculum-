

# Phase 1 — Functional Interfaces

## What is a Functional Interface?

A **functional interface** is any interface with **exactly one abstract method (SAM — Single Abstract Method)**. That's the only rule.

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);  // the single abstract method
}
```

The `@FunctionalInterface` annotation is **optional** — the compiler enforces SAM regardless. The annotation just makes intent explicit and causes a compile error if you accidentally add a second abstract method.

A functional interface **can** have:
- Default methods (any number)
- Static methods (any number)
- Methods inherited from `Object` (`equals`, `hashCode`, `toString`) — these don't count toward the SAM count

---

## Why They Exist

Before Java 8, to pass behavior you wrote anonymous inner classes:

```java
list.sort(new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});
```

Functional interfaces + lambdas collapse this to:

```java
list.sort((a, b) -> a.compareTo(b));
// or even:
list.sort(String::compareTo);
```

The lambda is just a **concise implementation of the SAM**.

---

## The Four Core Pillars — `java.util.function`

### 1. `Predicate<T>`

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

**Purpose:** Takes a T, returns boolean. Used for filtering/conditions.

```java
Predicate<String> isLong = s -> s.length() > 5;
System.out.println(isLong.test("hello"));     // false
System.out.println(isLong.test("shopsphere")); // true
```

**Default methods on Predicate (important for interviews):**

```java
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<String> longerThan3 = s -> s.length() > 3;

// AND
Predicate<String> combined = startsWithA.and(longerThan3);

// OR
Predicate<String> either = startsWithA.or(longerThan3);

// NEGATE
Predicate<String> notLong = longerThan3.negate();
```

---

### 2. `Function<T, R>`

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

**Purpose:** Takes a T, produces an R. Used for transformation/mapping.

```java
Function<String, Integer> length = String::length;
System.out.println(length.apply("ShopSphere")); // 10
```

**Default methods:**

```java
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;

// andThen: trim first, THEN uppercase
Function<String, String> trimThenUpper = trim.andThen(upper);

// compose: uppercase first, THEN trim (reverse of andThen)
Function<String, String> upperThenTrim = trim.compose(upper);
```

> **Interview trap:** `andThen` vs `compose` — `f.andThen(g)` means `g(f(x))`. `f.compose(g)` means `f(g(x))`.

---

### 3. `Consumer<T>`

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```

**Purpose:** Takes a T, returns nothing. Used for side effects (logging, saving, printing).

```java
Consumer<String> print = System.out::println;
print.accept("Order placed"); // Order placed
```

**Default method:**

```java
Consumer<String> log = s -> System.out.println("LOG: " + s);
Consumer<String> save = s -> System.out.println("SAVE: " + s);

Consumer<String> logThenSave = log.andThen(save);
logThenSave.accept("payment"); 
// LOG: payment
// SAVE: payment
```

---

### 4. `Supplier<T>`

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

**Purpose:** Takes nothing, returns a T. Used for lazy initialization, factories, default values.

```java
Supplier<List<String>> listFactory = ArrayList::new;
List<String> list = listFactory.get(); // fresh ArrayList each call
```

Real use in `Optional`:
```java
// orElseGet is lazy — Supplier only called if value is absent
Optional<String> name = Optional.empty();
String result = name.orElseGet(() -> "default"); // "default"
```

> **Interview point:** `orElse("default")` evaluates `"default"` **always**, even if the Optional has a value. `orElseGet(() -> expensive())` only calls `expensive()` if the Optional is empty. For costly computations, always prefer `orElseGet`.

---

## Bi-Variants

| Interface | Signature | Use case |
|---|---|---|
| `BiPredicate<T,U>` | `(T,U) → boolean` | Two-input condition |
| `BiFunction<T,U,R>` | `(T,U) → R` | Two-input transform |
| `BiConsumer<T,U>` | `(T,U) → void` | Two-input side effect |

```java
BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
System.out.println(repeat.apply("ha", 3)); // hahaha
```

No `BiSupplier` exists — a Supplier takes no input by definition, so "bi" makes no sense.

---

## Special Case: `UnaryOperator<T>` and `BinaryOperator<T>`

```java
// UnaryOperator<T> extends Function<T,T> — same type in and out
UnaryOperator<String> shout = s -> s.toUpperCase() + "!";

// BinaryOperator<T> extends BiFunction<T,T,T> — two inputs, same type out
BinaryOperator<Integer> add = (a, b) -> a + b;
```

These come up heavily in `Stream.iterate` and `reduce`.

---

## The Effectively Final Rule

Lambdas can capture local variables from the enclosing scope, but those variables must be **effectively final** — meaning they're never reassigned after initialization.

```java
String prefix = "ORDER-";              // effectively final ✅
Consumer<String> logger = id -> System.out.println(prefix + id);

prefix = "UPDATED-";                   // ❌ compile error — captured variable reassigned
```

Instance variables and static variables don't have this restriction — they're accessed via the heap, not the stack frame.

---

## Quick Cheat Sheet

| Interface | In | Out | SAM |
|---|---|---|---|
| `Predicate<T>` | T | boolean | `test(T)` |
| `Function<T,R>` | T | R | `apply(T)` |
| `Consumer<T>` | T | void | `accept(T)` |
| `Supplier<T>` | — | T | `get()` |
| `UnaryOperator<T>` | T | T | `apply(T)` |
| `BinaryOperator<T>` | T, T | T | `apply(T,T)` |

---

## Interview Questions on This Topic

**Q: Can a functional interface extend another interface?**
Yes — as long as it doesn't add a second abstract method. `UnaryOperator<T>` extends `Function<T,T>` and is still a functional interface.

**Q: Is `Comparator<T>` a functional interface even though it has `equals` declared?**
Yes. `equals` is inherited from `Object`, so it doesn't count as an abstract method. Comparator has exactly one SAM: `compare(T,T)`.

**Q: What's wrong with this code?**
```java
int count = 0;
list.forEach(s -> count++); // compile error
```
`count` is not effectively final — it's reassigned inside the lambda. Fix: use `AtomicInteger` or collect with a downstream `counting()` collector.

---

Solid on this? Say **next** and we move to **Lambda Expressions** — syntax, all four method reference forms, and the JVM-level picture of what a lambda actually is.
