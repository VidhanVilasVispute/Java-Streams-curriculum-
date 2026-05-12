
# Phase 1 — Lambda Expressions

## What a Lambda Actually Is

A lambda is a **concise implementation of a functional interface's SAM**. Nothing more. The compiler knows which functional interface you mean from the context (assignment, parameter type, return type).

```java
Predicate<String> p = s -> s.length() > 5;
//                   ^^^^^^^^^^^^^^^^^^^^
//         this IS the implementation of test(String s)
```

At the JVM level — lambdas are **not anonymous inner classes**. The compiler uses `invokedynamic` (introduced in Java 7, leveraged by Java 8 for lambdas). The actual class is generated at **runtime** by `LambdaMetafactory`, not at compile time. This means:

- No `.class` file on disk for each lambda
- First call has a small bootstrap cost; subsequent calls are fast
- Lambdas captured without state can be reused as a **singleton** by the JVM — an important optimization

---

## Syntax — All Forms

```java
// 1. No params
Supplier<String> s = () -> "hello";

// 2. One param — parentheses optional
Predicate<String> p = s -> s.isEmpty();

// 3. Multiple params
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// 4. Explicit types (rarely needed, compiler infers)
BiFunction<Integer, Integer, Integer> add = (Integer a, Integer b) -> a + b;

// 5. Single expression — no braces, no return keyword
Function<String, Integer> len = s -> s.length();

// 6. Block body — braces required, return required
Function<String, Integer> len = s -> {
    int result = s.length();
    return result;
};
```

> **Interview trap:** In a block body `{}`, `return` is mandatory. In a single-expression body, `return` is **forbidden** — the compiler adds it.

---

## Variable Capture Rules

Lambdas can access three categories of variables:

### 1. Local variables from the enclosing scope — must be effectively final

```java
String env = "PROD";          // effectively final — never reassigned
Supplier<String> tag = () -> "env=" + env;  // ✅

env = "DEV";                  // ❌ breaks effective finality
```

**Why this restriction exists:** Local variables live on the **stack** of the thread that created them. The lambda may execute on a different thread (e.g., in a parallel stream). If the stack frame is gone, the variable is gone. The JVM solves this by **copying** the value into the lambda — but a copy only makes sense if the value never changes. Mutable local capture would give you a stale copy with no visibility guarantee.

### 2. Instance variables — no restriction

```java
public class OrderService {
    private String currency = "INR";

    public void process(List<Order> orders) {
        orders.forEach(o -> System.out.println(currency + " " + o.getAmount()));
        // currency accessed via 'this' — heap, not stack, no restriction
    }
}
```

### 3. Static variables — no restriction

```java
private static final String PREFIX = "ORDER-";
Consumer<String> log = id -> System.out.println(PREFIX + id); // ✅
```

---

## Method References — All Four Forms

A method reference is just a shorter lambda when your lambda does nothing except call an existing method.

```
lambda:           s -> s.toUpperCase()
method reference: String::toUpperCase
```

### Form 1 — Static Method Reference

```
ClassName::staticMethod
```

```java
// Lambda
Function<String, Integer> parse = s -> Integer.parseInt(s);

// Method reference
Function<String, Integer> parse = Integer::parseInt;
```

```java
// In a stream
List<Integer> numbers = List.of("1", "2", "3")
    .stream()
    .map(Integer::parseInt)
    .collect(Collectors.toList());
```

---

### Form 2 — Instance Method Reference on a Specific Instance

```
instance::method
```

```java
String prefix = "SHOP-";

// Lambda
Predicate<String> check = s -> prefix.equals(s);  // specific instance 'prefix'

// Method reference
Predicate<String> check = prefix::equals;
```

```java
// Common example
PrintStream out = System.out;
Consumer<String> printer = out::println;
// same as: s -> System.out.println(s)
```

---

### Form 3 — Instance Method Reference on an Arbitrary Instance of a Type

```
ClassName::instanceMethod
```

This one trips people up. The **first parameter** of the lambda becomes the instance the method is called on.

```java
// Lambda
Function<String, String> upper = s -> s.toUpperCase();

// Method reference — 's' becomes the instance
Function<String, String> upper = String::toUpperCase;
```

```java
// BiFunction — first param is the instance, second is the argument
BiFunction<String, String, Boolean> starts = (s, prefix) -> s.startsWith(prefix);
BiFunction<String, String, Boolean> starts = String::startsWith;
```

> **Interview trap:** `String::toUpperCase` looks like a static method reference but it isn't — `toUpperCase` is an instance method. The compiler figures out Form 2 vs Form 3 by checking whether the method is static or instance and how many parameters are in context.

---

### Form 4 — Constructor Reference

```
ClassName::new
```

```java
// Lambda
Supplier<ArrayList<String>> factory = () -> new ArrayList<>();

// Constructor reference
Supplier<ArrayList<String>> factory = ArrayList::new;
```

```java
// With a parameter
Function<String, StringBuilder> build = s -> new StringBuilder(s);
Function<String, StringBuilder> build = StringBuilder::new;
```

```java
// In a stream — convert strings to Order objects
List<Order> orders = orderIds.stream()
    .map(Order::new)          // calls Order(String id) constructor
    .collect(Collectors.toList());
```

---

## Summary Table

| Form | Syntax | Equivalent Lambda |
|---|---|---|
| Static method | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Specific instance | `System.out::println` | `s -> System.out.println(s)` |
| Arbitrary instance | `String::toUpperCase` | `s -> s.toUpperCase()` |
| Constructor | `Order::new` | `() -> new Order()` |

---

## Lambdas vs Anonymous Inner Classes — Key Differences

| | Anonymous Inner Class | Lambda |
|---|---|---|
| `this` keyword | refers to the anonymous class instance | refers to the **enclosing class** instance |
| `.class` file | generated at compile time | generated at runtime via `invokedynamic` |
| Can have state | yes (fields) | no (only captures) |
| Shadowing variables | can shadow outer variables | cannot shadow — same scope |

```java
String name = "outer";

Runnable r = new Runnable() {
    String name = "inner";       // ✅ shadows outer 'name'
    public void run() {
        System.out.println(name); // prints "inner"
    }
};

Runnable r2 = () -> {
    String name = "inner";       // ❌ compile error — cannot redeclare 'name' in same scope
};
```

---

## Interview Questions on This Topic

**Q: What does `this` mean inside a lambda?**
It refers to the enclosing class instance, not the lambda itself. Lambdas don't introduce a new scope for `this`.

**Q: Can a lambda throw a checked exception?**
Only if the functional interface's SAM declares that checked exception in its `throws` clause. `Predicate.test()` doesn't declare any, so a lambda implementing Predicate cannot throw a checked exception without wrapping it.

```java
// Won't compile — test() doesn't declare IOException
Predicate<Path> exists = p -> Files.exists(p); // ✅ (no checked exception)
Predicate<Path> readable = p -> Files.readAllBytes(p); // ❌ — readAllBytes throws IOException
```

Fix pattern:
```java
@FunctionalInterface
interface ThrowingPredicate<T> {
    boolean test(T t) throws Exception;
}
```

**Q: Are two lambdas with the same body equal?**
No. `==` will return false. Lambdas don't override `equals`. Two separate lambda instances are two separate objects (unless the JVM caches a stateless one as a singleton — but you can't rely on that).

**Q: What is `invokedynamic` and why does it matter for lambdas?**
It's a JVM bytecode instruction that defers method binding to runtime. For lambdas, the first call bootstraps the implementation class via `LambdaMetafactory`; after that it's as fast as a direct call. The key benefit over anonymous inner classes: no `.class` file bloat, and the JVM can optimize (e.g., singleton reuse for stateless lambdas).

---

Say **next** and we go to **Optional\<T\>** — the last prerequisite before Streams proper.
