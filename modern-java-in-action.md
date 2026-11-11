# Modern Java in Action

Notes on *Modern Java in Action* (Raoul-Gabriel Urma, Mario Fusco, Alan Mycroft, Manning, 2nd edition). It covers Java 8 to 11: lambdas, streams, `Optional`, the new date/time API, default methods, modules, `CompletableFuture`, reactive programming and functional-style Java.

Each chapter below has a short explanation, key takeaways and code. Notes are in my own words and written from memory, so check details against the book.

## Chapters

| Part | Ch | Title | Notes |
|------|----|-------|-------|
| 1 Fundamentals | 1 | Java 8, 9, 10, 11: what's happening? | todo |
| | 2 | Passing code with behavior parameterization | todo |
| | 3 | Lambda expressions | todo |
| 2 Streams | 4 | Introducing streams | todo |
| | 5 | Working with streams | todo |
| | 6 | Collecting data with streams | todo |
| | 7 | Parallel data processing and performance | todo |
| 3 Streams + lambdas in practice | 8 | Collection API enhancements | todo |
| | 9 | Refactoring, testing, debugging | todo |
| | 10 | Domain-specific languages using lambdas | todo |
| 4 Everyday Java | 11 | `Optional` as a better alternative to null | todo |
| | 12 | New Date and Time API | todo |
| | 13 | Default methods | todo |
| | 14 | The Java Module System | todo |
| 5 Concurrency | 15 | Concepts behind `CompletableFuture` and reactive programming | todo |
| | 16 | `CompletableFuture`: composable async programming | todo |
| | 17 | Reactive programming | todo |
| 6 Functional + future | 18 | Thinking functionally | todo |
| | 19 | Functional programming techniques | todo |
| | 20 | Blending OOP and FP: Java vs Scala | todo |
| | 21 | Conclusions and where next for Java | todo |

(Set a row to `done` as notes are filled in. The code below is a first draft: refine and run it as you go through each chapter.)

---

## Part 1: Fundamentals

### Chapter 1: Java 8, 9, 10, 11: what's happening?

**Explanation**
- Hardware changed: CPUs got more cores instead of higher clock speeds, so programs must be concurrent to use the machine fully.
- Data changed too: applications process large collections, and the old way (external loops with mutable state) is verbose and hard to parallelize.
- Java 8 answered with three linked ideas: passing behavior as values (lambdas and method references), the Streams API for declarative data processing, and default methods so existing interfaces could gain methods without breaking implementors.
- Streams use *internal iteration*: you say what to do and the library decides how (sequential or parallel, fused loops, short-circuiting).
- Functional ideas imported into Java: functions as first-class values, immutability, no shared mutable state, and `Optional` instead of `null`.
- Later releases built on this: Java 9 added modules, `List.of`-style factories and the `Flow` API; Java 10 added `var`; Java 11 polished the platform (e.g. `String` and `Files` helpers).
- Java stays object-oriented. The goal is to mix OOP and FP where each is stronger, not to turn Java into a pure functional language.

**Why it matters:** hardware moved to multicore and data to big collections, so Java needed concise ways to pass behavior around and process data in parallel without hand-written threads.

Three main ideas:
1. **Behavior parameterization**: pass code (lambdas, method references) to methods.
2. **Streams**: declarative, pipeline-style data processing that parallelizes easily.
3. **Default methods**: evolve interfaces without breaking implementors.

```java
// Before
Collections.sort(inventory, new Comparator<Apple>() {
    public int compare(Apple a, Apple b) { return a.getWeight() - b.getWeight(); }
});
// After
inventory.sort(comparing(Apple::getWeight));
```

**Takeaways**
- Java borrowed from functional languages (lambdas, `Optional`, immutability) but remains object-oriented.
- Streams: *what* to do, not *how* (internal iteration).
- Java 9 added modules and the `Flow` API; Java 10 added `var` (local type inference).

---

### Chapter 2: Passing code with behavior parameterization

**Explanation**
- Requirements change constantly (filter green apples, then heavy apples, then green and heavy). Writing one method per requirement leads to duplication and unmaintainable code.
- *Behavior parameterization* means a method accepts code as an argument, so its behavior is decided by the caller. The method body stays generic; the varying part is the parameter.
- The steps of evolution: hard-coded method, then a method with value parameters (color, weight), then a flag-style filter (bad: unreadable), then a strategy interface (`Predicate`) with one class per rule, then anonymous classes, then lambdas.
- Anonymous classes reduce class count but are bulky and confusing (`this`, shadowing). Lambdas keep the same idea in one line.
- Abstracting over the *type* with generics (`Predicate<T>`) makes the same `filter` usable for apples, integers, strings, and so on.
- The JDK already used this idea before Java 8: `Comparator` for sorting, `Runnable` for threads, `Callable` for executors, and GUI event handlers.
- Costs: the interface must be defined once, and the call site must be readable. Lambdas minimize both.

**Idea:** instead of writing a method per requirement (`filterGreenApples`, `filterHeavyApples`), take the *behavior* as a parameter. It evolves from: copy-paste, then parameters, then strategy interface, then anonymous class, then lambda.

```java
interface Predicate<T> { boolean test(T t); }

static <T> List<T> filter(List<T> list, Predicate<T> p) {
    List<T> result = new ArrayList<>();
    for (T e : list) if (p.test(e)) result.add(e);
    return result;
}

filter(inventory, a -> a.getColor() == Color.GREEN);
filter(inventory, a -> a.getWeight() > 150);
```

**Takeaways**
- Behavior parameterization = pass a strategy to a method, so it copes with changing requirements.
- Anonymous classes are verbose; lambdas fix this.
- Real examples: `Comparator` (sort), `Runnable` (threads), `Callable` (executors).
- Abstract over *types* too with generics (`Predicate<T>`).

---

### Chapter 3: Lambda expressions

**Explanation**
- A lambda is an anonymous function: parameters, an arrow, a body. It has no name, but it has a parameter list, a body, a return type (inferred) and possibly thrown exceptions.
- It can be used only where a *functional interface* is expected: an interface with exactly one abstract method (`@FunctionalInterface` documents and enforces this). Default and static methods don't count.
- The lambda's type comes from context (*target typing*), so the compiler infers parameter types and you can usually omit them.
- `java.util.function` provides ready-made interfaces; the primitive specializations (`IntPredicate`, `ToIntFunction`, ...) avoid boxing cost.
- *Method references* (`Class::method`) are shorthand for a lambda that just calls one method. There are four kinds: static, instance on arbitrary object, instance on existing object, and constructor.
- Local variables used inside a lambda must be effectively final. This avoids visible races, because captured locals live on the stack and are copied.
- Inside a lambda `this` refers to the enclosing instance, whereas in an anonymous class it refers to the anonymous object itself.
- Functional interfaces provide composition helpers (`Comparator.reversed/thenComparing`, `Predicate.and/or/negate`, `Function.andThen/compose`) so small pieces build bigger behaviors.
- Checked exceptions are awkward inside lambdas: wrap them, or define your own functional interface that declares the exception.

**A lambda** is an anonymous function: `(params) -> expression` or `(params) -> { statements; }`. It can be used wherever a **functional interface** (exactly one abstract method) is expected.

```java
Runnable r = () -> System.out.println("hi");
Comparator<Apple> c = (a1, a2) -> a1.getWeight() - a2.getWeight();
```

Core functional interfaces (`java.util.function`):

| Interface | Method | Use |
|-----------|--------|-----|
| `Predicate<T>` | `boolean test(T)` | filter |
| `Consumer<T>` | `void accept(T)` | do something with it |
| `Function<T,R>` | `R apply(T)` | map / transform |
| `Supplier<T>` | `T get()` | produce a value |
| `BiFunction<T,U,R>` | `R apply(T,U)` | two inputs |
| `UnaryOperator<T>` / `BinaryOperator<T>` | | `T` to `T` |

Primitive versions (`IntPredicate`, `ToIntFunction`) avoid boxing.

**Method references** shorten lambdas:
```java
Function<String, Integer> f1 = s -> Integer.parseInt(s);
Function<String, Integer> f2 = Integer::parseInt;       // static
Supplier<List<String>> s = ArrayList::new;              // constructor
BiFunction<String, Integer, Character> charAt = String::charAt; // instance, arbitrary object
```

**Composing**
```java
Comparator<Apple> byWeightThenColor = comparing(Apple::getWeight).reversed().thenComparing(Apple::getColor);
Predicate<Apple> notRed = redApple.negate();
Function<Integer,Integer> h = f.andThen(g);   // g(f(x))
Function<Integer,Integer> k = f.compose(g);   // f(g(x))
```

**Takeaways**
- Local variables captured by a lambda must be effectively final (no reassignment).
- `this` inside a lambda is the enclosing class (unlike anonymous classes).
- The type of a lambda is inferred from context (target typing).

---

## Part 2: Functional-style data processing with streams

### Chapter 4: Introducing streams

**Explanation**
- A stream is a sequence of elements from a source (collection, array, file, generator) that supports aggregate operations such as `filter`, `map` and `reduce`. It does not store data.
- Compared with collections: a collection holds all its elements in memory and is eagerly built; a stream computes elements on demand, so it can even be infinite.
- A stream pipeline has three parts: a source, a chain of intermediate operations, and one terminal operation. Nothing runs until the terminal operation is called.
- Intermediate operations are lazy and return a new stream, which lets the library fuse them into a single pass (loop fusion) and stop early (short-circuiting, e.g. `limit`).
- Terminal operations (`collect`, `forEach`, `count`, `reduce`, `anyMatch`) consume the stream and produce a result or side effect.
- A stream can be traversed only once; reusing it throws `IllegalStateException`. Create a new stream from the source instead.
- Internal vs external iteration: with `for` loops you manage iteration; with streams the library does, which enables optimizations and parallelism without changing your code.
- Streams are declarative and read like the problem statement, which makes code shorter and easier to change than the equivalent loops.

**A stream** is a sequence of elements from a source that supports aggregate operations. It is not a data structure.

```java
List<String> names = dishes.stream()
    .filter(d -> d.getCalories() > 300)   // intermediate (lazy)
    .map(Dish::getName)
    .limit(3)
    .collect(toList());                   // terminal (triggers execution)
```

**Takeaways**
- **Internal iteration**: the library controls the loop (lets it optimize/parallelize).
- **Intermediate ops** (`filter`, `map`, `sorted`) are lazy and return a stream; **terminal ops** (`collect`, `forEach`, `count`) produce a result.
- **A stream can be consumed only once**: reusing throws `IllegalStateException`.
- Fusion: filter + map get merged in one pass, and `limit` short-circuits.

---

### Chapter 5: Working with streams

**Explanation**
- Filtering: `filter` by predicate, `distinct` to remove duplicates, `limit`/`skip` for slicing, and Java 9's `takeWhile`/`dropWhile` for slicing sorted data by a condition (better than `filter` as it can stop early).
- Mapping: `map` transforms each element one-to-one. `flatMap` maps each element to a stream and merges all of them, which turns a stream of arrays or lists into a flat stream.
- Finding and matching: `anyMatch`, `allMatch`, `noneMatch` return booleans and short-circuit; `findFirst` and `findAny` return an `Optional` (use `findAny` when order doesn't matter, as it parallelizes better).
- Reducing: `reduce(identity, op)` folds elements into one value; without identity it returns an `Optional` since the stream may be empty. The operation should be associative so it can run in parallel.
- Stateless operations (`map`, `filter`) need no memory of previous elements; stateful ones (`sorted`, `distinct`, `reduce`) must buffer all or many elements.
- Primitive streams (`IntStream`, `LongStream`, `DoubleStream`) avoid boxing and offer `sum`, `average`, `max` and ranges (`range`, `rangeClosed`). Convert with `mapToInt` and `boxed`.
- Ways to build streams: `Stream.of`, `Arrays.stream`, `Files.lines`, `Stream.iterate` and `Stream.generate` (infinite, so combine with `limit`).

```java
// filtering
.filter(Dish::isVegetarian)
.distinct()
.takeWhile(d -> d.getCalories() < 320)   // Java 9, on sorted data stops early
.dropWhile(...)                          // Java 9
.skip(2).limit(3)

// mapping
.map(String::length)
.flatMap(w -> Arrays.stream(w.split("")))   // flattens Stream<Stream<String>> to Stream<String>

// matching and finding
.anyMatch(Dish::isVegetarian); .allMatch(...); .noneMatch(...);
.filter(...).findAny();   // returns Optional

// reducing
int sum = nums.stream().reduce(0, Integer::sum);
Optional<Integer> max = nums.stream().reduce(Integer::max);
```

**Numeric streams** avoid boxing: `IntStream`, `LongStream`, `DoubleStream`.
```java
int total = dishes.stream().mapToInt(Dish::getCalories).sum();
IntStream.rangeClosed(1, 100).filter(n -> n % 2 == 0).count();
```

**Building streams:** `Stream.of(...)`, `Arrays.stream(arr)`, `Files.lines(path)`, `Stream.iterate(0, n -> n + 2).limit(5)`, `Stream.generate(Math::random)`.

**Takeaways**
- `flatMap` for one-to-many mapping, `mapToInt` for performance.
- `reduce` is the general fold, but use `sum()`/`max()` where possible.
- `sorted`, `distinct`, `reduce` are stateful and need to see all/many elements.

---

### Chapter 6: Collecting data with streams

**Explanation**
- `collect` takes a `Collector` that describes how to accumulate stream elements into a result, such as a list, a map or a number. `Collectors` provides the common ones.
- Reduction collectors: `counting`, `summingInt`, `averagingInt`, `minBy/maxBy`, `summarizingInt` (count, sum, min, avg, max in one pass), and `joining` for strings.
- Grouping: `groupingBy(classifier)` builds a `Map<K, List<T>>`, comparable to SQL `GROUP BY`. A second argument is a *downstream collector* applied to each group (`counting`, `mapping`, `averagingInt`, another `groupingBy` for multi-level grouping).
- Partitioning: `partitioningBy(predicate)` is a special grouping with a boolean key; the result always contains both `true` and `false` entries, even if empty.
- `collectingAndThen` applies a final transformation (e.g. making the result unmodifiable); `teeing` (Java 12) runs two collectors on the same pass and merges results.
- A custom `Collector<T, A, R>` defines `supplier` (new container), `accumulator` (add element), `combiner` (merge two containers for parallel use), `finisher` (convert to result) and `characteristics`.
- Prefer the built-in collectors: they are tested, composable and parallel-friendly. Custom ones are for special cases only.

`collect(Collector)` is the powerful terminal op. `Collectors` has built-ins:

```java
import static java.util.stream.Collectors.*;

long n = dishes.stream().collect(counting());
Map<Dish.Type, List<Dish>> byType = dishes.stream().collect(groupingBy(Dish::getType));

// multi-level and downstream collectors
Map<Dish.Type, Map<CaloricLevel, List<Dish>>> nested = dishes.stream()
    .collect(groupingBy(Dish::getType, groupingBy(this::level)));
Map<Dish.Type, Double> avg = dishes.stream()
    .collect(groupingBy(Dish::getType, averagingInt(Dish::getCalories)));

// partitioning: split in two by a predicate (Map<Boolean, List>)
Map<Boolean, List<Dish>> veg = dishes.stream().collect(partitioningBy(Dish::isVegetarian));

// others
String menu = dishes.stream().map(Dish::getName).collect(joining(", "));
IntSummaryStatistics stats = dishes.stream().collect(summarizingInt(Dish::getCalories));
```

Java 12 added `teeing` (two collectors combined). Custom collectors implement `Collector<T,A,R>` with `supplier`, `accumulator`, `combiner`, `finisher`, `characteristics`.

**Takeaways**
- `groupingBy` is the SQL `GROUP BY` of streams, and the second argument is the downstream collector.
- `partitioningBy` always has both keys (true/false).
- Most needs are met by the built-ins; write a custom `Collector` only for special cases.

---

### Chapter 7: Parallel data processing and performance

**Explanation**
- `parallel()` turns a sequential stream into a parallel one: the data is split into chunks, processed on different threads, and the partial results are combined. `sequential()` switches back (the last call wins).
- By default it uses the common fork/join pool, sized to the number of cores. All parallel streams in the JVM share that pool.
- Fork/join framework: a `RecursiveTask` splits itself until chunks are small enough (the threshold), computes them, then joins results. *Work stealing* lets idle threads take tasks from busy ones to keep cores balanced.
- A `Spliterator` is the splitting mechanism of a stream source; collections with cheap, even splitting (`ArrayList`, arrays, `IntStream.range`) parallelize well, while `LinkedList` and `Stream.iterate` do not.
- Parallel is not automatically faster. Overhead of splitting and merging, boxing, and ordered operations (`limit`, `findFirst`) can make it slower; always measure.
- Correctness: functions passed to parallel streams must be stateless and must not mutate shared variables, otherwise results are wrong (race conditions).
- Rule of thumb: use it for big data sets or expensive per-element work, with primitive streams where possible, and when the order of results doesn't matter.

```java
long sum = LongStream.rangeClosed(1, 10_000_000).parallel().sum();
```
`.parallel()` splits the stream and runs it on the common **fork/join pool** (default threads = CPU cores - 1). The `Spliterator` defines how a source is split.

**Pitfalls**
- Shared mutable state breaks:
```java
long[] total = {0};
LongStream.rangeClosed(1, n).parallel().forEach(i -> total[0] += i); // race, wrong result
```
- Boxing is slow: use primitive streams.
- Order-dependent ops (`limit`, `findFirst`, `forEachOrdered`) are costly in parallel.
- Small data or cheap per-element work makes parallel slower (splitting/merging overhead).
- `LinkedList`/`iterate` split poorly; `ArrayList`, arrays, `IntStream.range` split well.

**Fork/join basics:** extend `RecursiveTask<R>`, implement `compute()`, `fork()` the subtask and `join()`. It uses work stealing to balance threads.

**Takeaways**
- Measure before going parallel; it isn't automatically faster.
- Only use with stateless, non-interfering, associative operations.

---

## Part 3: Effective programming with streams and lambdas

### Chapter 8: Collection API enhancements

**Explanation**
- Java 9 factory methods `List.of`, `Set.of`, `Map.of`, `Map.ofEntries` create compact *immutable* collections: they throw `UnsupportedOperationException` on modification and reject nulls (`Set.of` also rejects duplicates).
- New `List`/`Set` methods that mutate in place: `removeIf` and `replaceAll`. Streams, by contrast, produce new collections rather than changing the source.
- New `Map` methods remove common boilerplate: `getOrDefault`, `putIfAbsent`, `computeIfAbsent` (lazily create a value, e.g. a multi-map), `computeIfPresent`, `compute`, `remove(key, value)`, `replace`, and `merge` (combine with an existing value, ideal for counters).
- `Map.forEach` and `Entry.comparingByKey/comparingByValue` make iteration and sorting concise.
- `HashMap` improved performance by converting long collision chains into balanced trees (needs keys to be `Comparable` to be effective).
- `ConcurrentHashMap` adds bulk operations (`forEach`, `reduce`, `search`) that take a parallelism threshold, plus `mappingCount` and `keySet(default)`.

```java
// factories (Java 9): immutable
List<String> l = List.of("a", "b");
Set<String> s = Set.of("a", "b");
Map<String, Integer> m = Map.of("a", 1, "b", 2);
Map<String, Integer> big = Map.ofEntries(Map.entry("a", 1), Map.entry("b", 2));

// List / Set (Java 8): in-place
list.removeIf(x -> x < 0);
list.replaceAll(String::toUpperCase);

// Map
map.getOrDefault("k", 0);
map.computeIfAbsent("k", k -> new ArrayList<>()).add(v);   // multi-map idiom
map.merge("word", 1, Integer::sum);                        // counting idiom
map.forEach((k, v) -> ...);
map.entrySet().stream().sorted(Map.Entry.comparingByValue());
```

**Takeaways**
- `List.of`/`Set.of`/`Map.of` are immutable (modifying throws `UnsupportedOperationException`, nulls rejected).
- `removeIf` and `replaceAll` modify in place; stream ops create new collections.
- `ConcurrentHashMap` gained bulk ops (`forEach`, `reduce`, `search`) with a parallelism threshold.

---

### Chapter 9: Refactoring, testing, debugging

**Explanation**
- Refactoring toward lambdas improves readability: convert anonymous classes to lambdas (careful: `this` and shadowed variables change meaning), then lambdas to method references, then loops to stream pipelines.
- Lambdas simplify classic design patterns by removing boilerplate classes: Strategy, Template Method, Observer, Chain of Responsibility and Factory can often be a function or a map of suppliers.
- Be careful with conditional deferred execution (logging): pass a `Supplier` so the message is computed only when needed.
- Testing: a lambda has no name, so you can't call it directly. Test the behavior of the method using it, or move complex logic into a named method and use a method reference; static fields holding lambdas can be tested directly.
- Test functions that take or return lambdas by checking their outputs for given inputs (they are just values).
- Debugging: stack traces for lambdas are unhelpful (`lambda$main$0`) so named methods and method references give better traces.
- `peek` lets you log values flowing through a pipeline stage by stage without changing the result; remove it afterwards.

**Refactor**
- Anonymous class to lambda (watch `this` and variable shadowing) to method reference.
- Loops to stream pipelines.
- Replace patterns with lambdas:

| Pattern | With lambdas |
|---------|--------------|
| Strategy | pass a `Predicate`/`Function` instead of a class per strategy |
| Template method | pass the varying step as a `Consumer`/`Function` |
| Observer | `feed.register(tweet -> ...)` |
| Chain of responsibility | `UnaryOperator<T>` composed with `andThen` |
| Factory | `Map<String, Supplier<Product>>` |

```java
// Strategy without extra classes
Validator numeric = new Validator(s -> s.matches("\\d+"));
// Factory
Map<String, Supplier<Product>> map = Map.of("loan", Loan::new, "stock", Stock::new);
Product p = map.get("loan").get();
```

**Test:** lambdas have no name, so test the method that uses them, or extract the logic into a named method / static `Comparator` field and test that.

**Debug:** stack traces for lambdas look like `lambda$main$0`, so prefer method references (named). Use `peek` to see values inside a pipeline:
```java
nums.stream().peek(x -> System.out.println("from: " + x)).map(x -> x + 17).peek(x -> System.out.println("mapped: " + x)).collect(toList());
```

---

### Chapter 10: Domain-specific languages using lambdas

**Explanation**
- A domain-specific language (DSL) is a small API focused on one problem, so that code reads like the domain (e.g. an order-taking DSL) and non-programmers can follow it.
- Benefits: readability, maintainability, reuse. Costs: effort to design, an extra layer, and harder debugging.
- Java's DSL options are *internal* (written in Java using its syntax) rather than external (a separate parser). Examples: the Stream API, `Collectors`, jOOQ, Spring Integration, Cucumber.
- Method chaining: each method returns the builder so calls read as a sentence; good flow but needs builder classes and hides ordering rules.
- Nested functions: functions as arguments (`order(customer(...), buy(...))`) keep hierarchy visible but add noise from static imports and parentheses.
- Lambda sequencing: a lambda receives a builder and configures it, which gives scoped, nested structure with little syntax overhead.
- Real DSLs mix these styles. Use static imports and a clear naming convention to keep them clean.

A **DSL** is a small API that reads like the problem domain. Java's own examples: the `Stream` API, `Collectors`; external ones: jOOQ (SQL), Cucumber, Spring Integration.

Patterns for building one:
- **Method chaining** (fluent / builder): `order.forCustomer("X").buy(100).stocks("IBM")`
- **Nested functions**: `order(customer("X"), buy(stock("IBM", 100)))`
- **Lambda sequencing**: `order(o -> { o.forCustomer("X"); o.buy(t -> t.quantity(80).stock("IBM")); })`
- Mixed: combine them.

**Takeaways**
- Lambdas let you pass a configured *scope* (a builder consumer), so nesting looks natural.
- Trade-off: fluent APIs are readable to use but take a lot of work to write and are harder to debug.

---

## Part 4: Everyday Java

### Chapter 11: `Optional` as a better alternative to null

**Explanation**
- `null` causes `NullPointerException`, forces defensive checks, hides the meaning of "no value", and breaks the typing of Java (every reference may be null).
- `Optional<T>` is a container that holds either a value or nothing, so possible absence is visible in the method signature.
- Create with `of` (non-null, throws NPE otherwise), `ofNullable`, or `empty`.
- Use `map` to transform the contained value and `flatMap` when the function itself returns an `Optional` (avoids `Optional<Optional<T>>` and nested null checks).
- Unwrap safely: `orElse`, `orElseGet` (lazy, use for expensive defaults), `orElseThrow`, `ifPresent`, `ifPresentOrElse` (9); `or` (9) chains alternative sources; `stream` (9) turns it into a 0-or-1 element stream.
- Avoid `get()` without a check: it throws when empty and defeats the purpose.
- Where to use: as a return type. Not as a field or method parameter (it is not `Serializable` and adds indirection), not wrapping collections (return an empty collection instead).
- Primitive variants (`OptionalInt`, ...) exist, but they lack `map`/`flatMap`.

`null` is the "billion dollar mistake": it causes `NullPointerException`, clutters code with checks, and carries no meaning. `Optional<T>` makes absence explicit in the type.

```java
Optional<String> o = Optional.of("x");         // NPE if null
Optional<String> e = Optional.empty();
Optional<String> n = Optional.ofNullable(maybeNull);

// chaining without nested null checks
String name = person.flatMap(Person::getCar)
                    .flatMap(Car::getInsurance)
                    .map(Insurance::getName)
                    .orElse("Unknown");
```

Useful methods: `map`, `flatMap`, `filter`, `ifPresent`, `ifPresentOrElse` (9), `or` (9), `orElse`, `orElseGet`, `orElseThrow`, `stream` (9).

**Takeaways**
- Use as a **return type** to say "may have no value". Don't use it for fields, parameters or collections elements (not `Serializable`, extra overhead).
- Prefer `orElseGet(() -> expensive())` over `orElse(expensive())` (the latter always evaluates).
- Don't call `get()` without checking: it just moves the NPE.
- Primitive variants: `OptionalInt`, etc.

---

### Chapter 12: New Date and Time API (`java.time`)

**Explanation**
- `java.util.Date` and `Calendar` have well-known problems: mutability, 0-based months, awkward time zones and non-thread-safe formatters.
- The `java.time` API is immutable and thread-safe: every modification returns a new object.
- Human time types: `LocalDate`, `LocalTime`, `LocalDateTime` (no time zone); machine time type: `Instant` (nanoseconds since the epoch); zone-aware types: `ZonedDateTime`, `OffsetDateTime`, with `ZoneId`.
- Amounts of time: `Duration` for time-based (seconds, nanos) and `Period` for date-based (years, months, days); `ChronoUnit.X.between` gives a single unit.
- Manipulation: `plusX`, `minusX`, `withX`, and `TemporalAdjusters` for rules such as "next Monday" or "last day of month". You can write custom `TemporalAdjuster`s.
- Formatting and parsing use `DateTimeFormatter` (predefined ISO formats or `ofPattern`), and unlike `SimpleDateFormat` it is thread-safe.
- Choose the type by need: dates and times without zone for local business concepts, `Instant` for timestamps, `ZonedDateTime` when daylight saving or zones matter. Convert from legacy types with `toInstant()` and `Date.from`.

The old `Date`/`Calendar` are mutable, confusing (months 0-based) and not thread-safe. `java.time` is immutable and clear.

```java
LocalDate d = LocalDate.of(2024, Month.MARCH, 18);
LocalTime t = LocalTime.of(13, 45);
LocalDateTime dt = LocalDateTime.of(d, t);
Instant now = Instant.now();                         // machine time, since epoch
ZonedDateTime z = dt.atZone(ZoneId.of("Europe/Rome"));

// manipulation returns a new object
LocalDate next = d.plusWeeks(1).withDayOfMonth(1);
LocalDate nextMonday = d.with(TemporalAdjusters.next(DayOfWeek.MONDAY));

// amounts
Duration dur = Duration.between(t1, t2);            // time-based
Period p = Period.between(d1, d2);                  // date-based
long days = ChronoUnit.DAYS.between(d1, d2);

// format / parse
DateTimeFormatter f = DateTimeFormatter.ofPattern("dd/MM/yyyy");
LocalDate parsed = LocalDate.parse("18/03/2024", f);
```

**Takeaways**
- Immutable, thread-safe, so no defensive copies.
- Pick the right type: `LocalDate` (no zone), `Instant` (timestamps), `ZonedDateTime` (zone-aware).
- Use `DateTimeFormatter`, not the non-thread-safe `SimpleDateFormat`.

---

### Chapter 13: Default methods

**Explanation**
- Default methods let interfaces contain implemented methods (`default` keyword) so that libraries can add methods (`List.sort`, `Collection.stream`) without breaking every existing implementation.
- Static methods in interfaces replace companion utility classes (`Comparator.comparing`, `List.of`).
- Two kinds of compatibility: binary (old binaries still run), source (old code still compiles), and behavioral. Adding a default method preserves all three in most cases; adding an abstract method breaks them.
- This gives a restricted form of multiple inheritance of behavior (never of state).
- Conflict resolution rules, applied in order: (1) a method declared in a class or superclass wins over any default; (2) otherwise the most specific default interface wins (the sub-interface); (3) otherwise the class must resolve it explicitly with `Interface.super.method()`.
- Diamond problem: if two interfaces provide a default with the same signature and are unrelated, the compiler forces you to choose.
- Design tip: prefer small, focused interfaces with default methods you can combine (mixin style) to giant base classes.

Interfaces can now have method bodies (`default`) and static methods, so libraries can add methods without breaking implementors (this is how `List.sort` and `Collection.stream` arrived).

```java
interface Sized {
    int size();
    default boolean isEmpty() { return size() == 0; }
}
```

**Multiple inheritance of behavior** and conflicts, resolved in order:
1. A class (or superclass) method always wins over interface defaults.
2. Otherwise the most specific interface wins (sub-interface overrides).
3. Otherwise ambiguous: the class must override and pick explicitly:
```java
class C implements A, B {
    public void hello() { A.super.hello(); }
}
```

**Takeaways**
- Not abstract classes: interfaces still have no state.
- Use for backward-compatible API evolution and optional methods/utilities.
- Design them as small composable pieces (rather than inheriting a big class).

---

### Chapter 14: The Java Module System

**Explanation**
- Problem before Java 9: the classpath is a flat list, so any public class in any JAR is visible to everyone ("JAR hell", accidental use of JDK internals, no explicit dependencies).
- A module is a named group of packages plus a descriptor, `module-info.java`, that declares dependencies and the exported API. Everything not exported is hidden, even if `public`.
- Descriptor directives: `requires` (dependency), `requires transitive` (pass dependency on to users), `exports` (make a package accessible), `exports ... to` (limit it to named modules), `opens` (allow deep reflection), `provides ... with` and `uses` (service loading).
- Benefits: strong encapsulation, reliable configuration (missing or conflicting modules are found at startup), and smaller custom runtimes via `jlink`.
- The JDK is modularized (`java.base` is implicitly required). Tools: `jdeps` for analysing dependencies, `jlink` for building runtimes.
- Migration: libraries without descriptors work as *automatic modules* (name from the JAR); the unnamed module holds classpath code. Many applications still use the classpath, so modules matter most for library authors and custom runtimes.

Since Java 9, a **module** groups packages and declares what it needs and what it exposes. This gives strong encapsulation (public no longer means visible to everyone) and reliable configuration.

```java
// module-info.java at the root of the source
module com.example.shop {
    requires java.sql;
    requires transitive com.example.core;
    exports com.example.shop.api;                       // only this package is visible
    opens com.example.shop.model to com.fasterxml.jackson.databind; // reflection access
    provides com.example.spi.Payment with com.example.shop.CardPayment;
    uses com.example.spi.Payment;
}
```

**Takeaways**
- Why: no more "JAR hell" and the classpath's accidental access to internal APIs; smaller runtimes (`jlink`).
- `exports` = visible API, `requires` = dependencies, `opens` = reflective access, `provides/uses` = services.
- The JDK itself is modular (`java.base` is always required).
- Many apps still use the classpath; modules matter most for libraries and custom runtimes.

---

## Part 5: Enhanced Java concurrency

### Chapter 15: Concepts behind `CompletableFuture` and reactive programming

**Explanation**
- Concurrency is about structure (many tasks in progress); parallelism is about executing at the same time on several cores. Both matter on multicore hardware.
- Evolution of Java support: raw threads, then the executor framework (`ExecutorService`), then `Future`, then `CompletableFuture`, then reactive `Flow`.
- `Future` represents an async result but only offers blocking `get()` and polling, so composing several async results is clumsy.
- Many programs mostly *wait* (network, disk, remote services). Blocking a thread while waiting wastes it, and thread-per-request does not scale.
- Pool sizing: for CPU-bound work use about the number of cores; for tasks that wait, a bigger pool is needed (rule of thumb: cores x (1 + wait time / compute time)).
- Non-blocking, callback or future-composition based styles let few threads serve many tasks.
- The Reactive Manifesto: systems should be responsive, resilient, elastic and message-driven. *Backpressure* prevents fast producers from overwhelming slow consumers.

**Why:** multicore needs concurrency and parallelism, and apps spend a lot of time *waiting* (I/O, remote calls). Blocking a thread while waiting wastes it.

- Threads, then the executor framework (`ExecutorService`), then `Future` (blocking `get()`, hard to compose), then `CompletableFuture` (composable, callback-based), then reactive streams (`Flow`).
- **Reactive Manifesto:** responsive, resilient, elastic, message-driven.
- **Backpressure**: a slow consumer tells the producer how much it can handle.

**Takeaways**
- Avoid blocking in code that runs on a shared small pool (can starve it); use async APIs.
- Pick pool sizes for the workload: roughly `cores * (1 + wait/compute)`.
- `Future` can't express "when A and B finish, do C" cleanly; that is what `CompletableFuture` is for.

---

### Chapter 16: `CompletableFuture`: composable asynchronous programming

**Explanation**
- `CompletableFuture` implements `Future` and `CompletionStage`: it can be completed manually (`complete`) and supports chaining stages without blocking.
- Start async work with `supplyAsync` (returns a value) or `runAsync`. By default it runs on the common fork/join pool; pass your own `Executor` for blocking tasks.
- `thenApply` transforms the result (like `map`); `thenCompose` chains a dependent async call (like `flatMap`); `thenCombine` merges two independent results; `thenAccept` consumes it.
- Variants ending in `Async` run the next stage on another pool thread; the plain ones may run on the completing thread.
- Handling many futures: first create all of them in a list, then `join` them, because joining inside the same stream makes work sequential. `allOf` waits for all, `anyOf` for the first.
- Error handling: `exceptionally`, `handle` (result or exception), `whenComplete`. Timeouts with `orTimeout` and `completeOnTimeout` (Java 9).
- `join()` is like `get()` but throws an unchecked `CompletionException`.
- It enables writing the example flow (find price, then apply discount, then convert currency) as a non-blocking pipeline.

```java
CompletableFuture<Double> price = CompletableFuture.supplyAsync(() -> shop.getPrice("phone"));

// transform (like map) / chain dependent async step (like flatMap) / combine two independent
price.thenApply(p -> p * 1.2);
price.thenCompose(p -> CompletableFuture.supplyAsync(() -> discount(p)));
price.thenCombine(otherPrice, (a, b) -> Math.min(a, b));

// many at once
List<CompletableFuture<String>> futures = shops.stream()
    .map(s -> CompletableFuture.supplyAsync(() -> s.getPrice(product), executor))
    .collect(toList());                       // collect first, join after, so they run in parallel
List<String> prices = futures.stream().map(CompletableFuture::join).collect(toList());

CompletableFuture.allOf(array).join();        // wait for all
CompletableFuture.anyOf(array).join();        // first one wins

// errors
price.exceptionally(ex -> 0.0)
     .orTimeout(2, TimeUnit.SECONDS);         // Java 9
```

**Takeaways**
- Use a custom `Executor` for blocking tasks (the default is the common fork/join pool).
- Create all futures first and join afterwards. Joining within a single stream pipeline makes it sequential.
- `thenCompose` vs `thenApply` is the `flatMap` vs `map` distinction.
- `*Async` variants run the next stage on another thread.

---

### Chapter 17: Reactive programming

**Explanation**
- Reactive programming deals with asynchronous streams of events, where producers push items to consumers and consumers control the rate.
- Java 9 standardized the interfaces in `java.util.concurrent.Flow`: `Publisher`, `Subscriber`, `Subscription`, `Processor`. They define the contract, not a full implementation.
- Protocol: subscriber calls `subscribe`, publisher calls `onSubscribe` with a `Subscription`, subscriber requests `n` items with `request(n)`, publisher delivers with `onNext` up to `n`, and ends with `onComplete` or `onError`. `request` is how backpressure is expressed.
- `SubmissionPublisher` is a simple JDK implementation of `Publisher` useful for learning.
- A `Processor` is both subscriber and publisher: a transformation stage in the middle of a pipeline.
- For real projects use libraries built on this model: Project Reactor, RxJava, Akka Streams. They provide many operators (`map`, `filter`, `merge`, `zip`, `buffer`) similar to the Stream API.
- Difference from `Stream`: reactive is async, push with backpressure, long-lived and possibly infinite, whereas `Stream` is pull-based and single-use.
- Reactive is a good fit for I/O heavy, high-concurrency systems, but adds complexity, so use it where it pays off.

Reactive = asynchronous stream of events with backpressure. Java 9 standardised the interfaces in `java.util.concurrent.Flow`:

```java
Publisher<T>     { void subscribe(Subscriber<? super T> s); }
Subscriber<T>    { onSubscribe(Subscription s); onNext(T item); onError(Throwable t); onComplete(); }
Subscription     { void request(long n); void cancel(); }   // request(n) = backpressure
Processor<T,R>   // both Subscriber and Publisher (a transformation stage)
```

```java
// Simple use with the JDK publisher
try (SubmissionPublisher<String> pub = new SubmissionPublisher<>()) {
    pub.subscribe(new Flow.Subscriber<>() {
        Flow.Subscription sub;
        public void onSubscribe(Flow.Subscription s) { sub = s; s.request(1); }
        public void onNext(String item) { System.out.println(item); sub.request(1); }
        public void onError(Throwable t) { t.printStackTrace(); }
        public void onComplete() { System.out.println("done"); }
    });
    pub.submit("hello");
}
```

**Takeaways**
- `Flow` is only the contract. In practice you use a library: **Project Reactor** (Spring WebFlux), **RxJava**, Akka Streams.
- Differs from `Stream`: reactive is push-based, async, long-lived and can be infinite; `Stream` is pull-based and single use.
- Operators like `map`, `filter`, `merge`, `zip` look like the Stream API.

---

## Part 6: Functional programming and future Java evolution

### Chapter 18: Thinking functionally

**Explanation**
- Functional-style programming means building programs from functions without side effects and with immutable data.
- Side effects include mutating a data structure or variable, throwing exceptions, and doing I/O. A *pure* function has none: same inputs always give the same output.
- *Referential transparency*: a call can be replaced by its result without changing program behavior. This enables caching (memoization), reordering and safe parallelism.
- Java rarely enforces purity, so a practical guideline is "functionally pure to the outside": a method may use local mutation internally as long as callers see no side effects.
- Immutability: return new objects (new list, new tree) instead of changing arguments. It eliminates a large class of concurrency bugs.
- Recursion is the functional replacement for loops with mutable counters. Java does not optimize tail calls, so deep recursion can cause `StackOverflowError`; use streams or iteration for large inputs.
- Declarative vs imperative: describe what you want (streams) rather than step-by-step mutation.

**Functional-style:** programs built from **pure functions** (no side effects) and **immutable data**.

- **Pure function:** same input gives same output and no mutation of shared state, no I/O, no exceptions as control flow. This is "referential transparency": a call can be replaced by its result.
- **Immutability** makes code easier to reason about and safe to parallelize.
- **Recursion** replaces loops with mutable counters. The JVM has **no tail-call optimization**, so deep recursion overflows the stack; Java's answer is to use streams or iteration for such cases.

```java
// iterative with mutation
static int factorialIterative(int n) { int r = 1; for (int i = 1; i <= n; i++) r *= i; return r; }
// declarative
static long factorialStreams(long n) { return LongStream.rangeClosed(1, n).reduce(1, (a, b) -> a * b); }
// tail-recursive style (no Java TCO, but shows the idea)
static long factorialTail(long n, long acc) { return n == 1 ? acc : factorialTail(n - 1, n * acc); }
```

**Takeaways**
- Side effects are the main source of bugs: push them to the edges.
- Don't mutate arguments: return new values (e.g. a new list/tree).
- Java is "functional enough" in the small; truly pure code is a discipline, not enforced.

---

### Chapter 19: Functional programming techniques

**Explanation**
- Higher-order functions take functions as arguments or return them. Lambdas make this practical in Java.
- Currying turns a multi-argument function into a chain of single-argument functions, enabling partial application (fix some arguments and get a new function).
- Persistent data structures: operations return a new version and share unchanged parts with the old one, so the old version remains valid and nothing is mutated (e.g. updating a tree copies only the path to the changed node).
- Lazy evaluation: compute values only when required. Streams do this; you can model it with `Supplier` or a custom lazy list (head plus a supplier for the tail), useful for infinite sequences such as primes.
- Pattern matching is not built into Java (as of the book), so it is simulated with the visitor pattern, `instanceof` chains, or helper functions that take lambdas per case.
- Combinators are functions that combine other functions (`compose`, `andThen`, custom helpers), a way to build complex behavior from small, reusable parts.
- Miscellaneous: memoization, using `Optional`/`Either`-style types for errors instead of exceptions.

- **Higher-order functions:** take or return functions (`map`, `filter`, `comparing`).
- **Currying:** turn `f(x, y)` into `f(x)(y)` and partially apply.
```java
static Function<Double, Double> converter(double factor, double base) {
    return x -> x * factor + base;
}
Function<Double, Double> cToF = converter(9.0 / 5, 32);
```
- **Persistent data structures:** "updates" return a new version and share unchanged parts, so the old version stays valid (e.g. a tree `update` copying only the path).
- **Lazy evaluation:** compute only when needed. Streams are lazy; for custom laziness use `Supplier<T>` or a lazy list.
- **Pattern matching:** Java had none at the time, only simulated with visitors or `if/instanceof` chains. (Now available: see below.)
- **Combinators:** small functions composed (`compose`, `andThen`).

**Takeaways**
- Persistent structures give cheap versioning and thread safety.
- Prefer composing small functions to big flows with state.

---

### Chapter 20: Blending OOP and FP: comparing Java and Scala

**Explanation**
- Scala runs on the JVM and combines OOP and FP, so it is a good lens to see where Java is heading.
- Features: type inference everywhere, `val` (immutable) vs `var`, concise classes and `case class`es (data + equals/hashCode/toString/copy for free).
- Pattern matching with `match` (and sealed traits) gives exhaustive, expressive decomposition of data, something Java later got via records, sealed types and `switch` patterns.
- Traits are like interfaces with state and implementation, usable as mixins.
- Immutable collections are the default, and collection operations (`map`, `filter`, `fold`) are available directly on collections, without `stream()`.
- Higher-kinded types and richer type system enable constructs like `Option`, `Try`, `Either`.
- Trade-off: Scala is more concise and powerful but has a larger learning curve, a slower compile, and several ways to do the same thing; Java prioritizes simplicity and backward compatibility.

A tour of Scala features next to Java: concise syntax, type inference, immutable collections by default (`val`, `List`), `case class` + pattern matching, traits (like interfaces with state/mixins), higher-kinded functional constructs (`Option`, `Try`).

```scala
// Scala: case class and pattern matching
sealed trait Shape
case class Circle(r: Double) extends Shape
case class Rect(w: Double, h: Double) extends Shape
def area(s: Shape) = s match {
  case Circle(r)  => math.Pi * r * r
  case Rect(w, h) => w * h
}
```

**Takeaways**
- Scala is more functional and more concise but has a steeper learning curve and more ways to do things.
- Java adopts good ideas gradually and keeps compatibility.

---

### Chapter 21: Conclusions and where next for Java

**Explanation**
- The book's theme: Java 8 added functional features without giving up OOP and compatibility; the result is a multi-paradigm language.
- Key recap: behavior parameterization, lambdas, streams, `Optional`, `CompletableFuture`, the date/time API, default methods and modules.
- Planned or discussed future work at the time: improved generics (reified/specialised for primitives), declaration-site variance, *value types* (Project Valhalla), more pattern matching, and better concurrency (Project Loom).
- Many of these arrived afterwards: records (16), sealed classes (17), pattern matching for `instanceof` (16) and `switch` (21), virtual threads (21).
- Take away: learn the core ideas (immutability, composition, declarative style, async composition) rather than syntax, since they carry over to newer Java versions and other languages.
- Keep the pragmatic view: use functional style where it clarifies code, and stay with plain OOP where it is simpler.

Recap of the book's themes and where Java was heading. Areas mentioned: generics improvements, **value types** (Project Valhalla), more pattern matching and **records/sealed types** (later added), improved concurrency (**Project Loom** virtual threads).

**What arrived after the book** (all in Java 14+):
```java
// record: concise immutable data carrier
record Point(int x, int y) {}

// sealed types + pattern matching in switch (Java 21)
sealed interface Shape permits Circle, Square {}
record Circle(double r) implements Shape {}
record Square(double s) implements Shape {}

double area(Shape s) {
    return switch (s) {
        case Circle c -> Math.PI * c.r() * c.r();
        case Square q -> q.s() * q.s();
    };
}

// virtual threads (Java 21)
try (var ex = Executors.newVirtualThreadPerTaskExecutor()) {
    ex.submit(() -> fetch(url));
}
```

**Takeaways**
- The ideas in this book (lambdas, streams, `Optional`, immutability, async composition) are the basis for modern Java.
- Newer features (records, sealed classes, pattern matching, virtual threads) fill gaps the book pointed at.
