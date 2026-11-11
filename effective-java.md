# Effective Java

Notes on *Effective Java* (Joshua Bloch, 3rd edition): 90 practical items on writing clear, correct, maintainable Java. Each item is a rule with the reasoning behind it. The notes below are in my own words with short examples.

## Chapters

| Ch | Title | Notes |
|----|-------|-------|
| 2 | Creating and Destroying Objects | done |
| 3 | Methods Common to All Objects | done |
| 4 | Classes and Interfaces | todo |
| 5 | Generics | todo |
| 6 | Enums and Annotations | todo |
| 7 | Lambdas and Streams | todo |
| 8 | Methods | todo |
| 9 | General Programming | todo |
| 10 | Exceptions | todo |
| 11 | Concurrency | todo |
| 12 | Serialization | todo |

(Chapter 1 is the introduction, so there are no notes for it.)


## Chapter 2: Creating and Destroying Objects

### Item 1: Static factory methods > constructors

A static method that returns an instance of the class, offered in addition to (or instead of) a public constructor.

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

**Advantages**
- They have names. `BigInteger.probablePrime(...)` says more than a constructor, and you can have several with the same parameter types (constructors can't).
- They don't have to create a new object on each call. Allows immutable/cached/preconstructed instances (`Boolean.valueOf`, `Integer.valueOf`).
- They can return any subtype of the return type, so the implementation class can stay hidden.
```java
// EnumSet has no public constructors; it picks the implementation by size
EnumSet<Day> days = EnumSet.of(Day.MON, Day.TUE); // RegularEnumSet or JumboEnumSet inside
List<String> list = List.of("a", "b");            // caller never sees the concrete class
```
- The returned class can vary between calls (based on input args), and can change in later releases without breaking callers.
- Less verbose with generics inference: `new HashMap<String, List<String>>()` vs `Map<String, List<String>> m = new HashMap<>()` (the diamond now covers most of this).

**Disadvantages**
- A class with only non-public constructors can't be subclassed (arguably a plus: it pushes you to composition over inheritance).
- Hard to find in Javadoc, since they don't stand out from other static methods. Use common names:
  - `from` (conversion): `Date d = Date.from(instant);`
  - `of` (aggregation): `Set<Rank> faces = EnumSet.of(JACK, QUEEN, KING);`
  - `valueOf` (verbose `from`/`of`): `BigInteger.valueOf(Integer.MAX_VALUE)`
  - `getInstance` / `newInstance` (`newInstance` guarantees a new object)

---

### Item 2: Use a builder when facing many constructor parameters

Problem: with many (especially optional) params, telescoping constructors are hard to read and easy to get wrong (`new Pizza(12, 1, 0, 3, true)` - which is which?). JavaBeans setters fix readability but leave the object in an inconsistent state mid-construction and can't be immutable.

Builder: readable like named params, object is immutable and built in one step.

```java
public class NutritionFacts {
    private final int servingSize;  // required
    private final int calories;     // optional
    private final int sodium;       // optional

    public static class Builder {
        private final int servingSize;      // required
        private int calories = 0;           // optional, with defaults
        private int sodium = 0;

        public Builder(int servingSize) { this.servingSize = servingSize; }
        public Builder calories(int v)  { calories = v; return this; }
        public Builder sodium(int v)    { sodium = v; return this; }
        public NutritionFacts build()   { return new NutritionFacts(this); }
    }

    private NutritionFacts(Builder b) {
        servingSize = b.servingSize;
        calories = b.calories;
        sodium = b.sodium;
        // validate invariants here
    }
}

NutritionFacts cola = new NutritionFacts.Builder(240).calories(100).sodium(35).build();
```

Rule of thumb: use it when there are ~4+ params, especially if many are optional. Cost: extra object and more code (Lombok's `@Builder` removes the boilerplate).

---

### Item 3: Enforce the singleton property with a private constructor or an enum

Three ways:
1. **public static final field**, private constructor: `public static final Elvis INSTANCE = new Elvis();`
2. **static factory** returning a private static instance: `Elvis.getInstance()` (lets you change to non-singleton later without changing the API).
3. **single-element enum** (preferred): simple, serialization-safe, and immune to reflection attacks.

```java
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { ... }
}
```

Caveats for 1 and 2: serializable singletons need `transient` fields plus a `readResolve` method, otherwise deserializing creates a new instance. Reflection can call the private constructor. An enum can't extend a class (it can implement interfaces).

Note: singletons are hard to test (can't swap in a mock). See Item 5.

---

### Item 4: Enforce noninstantiability with a private constructor

For utility classes (only static methods, like `Math`, `Collections`). Without an explicit constructor, the compiler adds a public default one, and `abstract` doesn't help (it can be subclassed).

```java
public class MyUtils {
    private MyUtils() {
        throw new AssertionError(); // guards against calls from inside the class
    }
}
```

Side effect: the class can't be subclassed (no accessible super constructor).

---

### Item 5: Prefer dependency injection to hardwiring resources

Don't let a class create or look up its own dependencies via static utilities or singletons. Pass them in through the constructor.

```java
// Bad: hardwired, can't test with a different dictionary
class SpellChecker {
    private final Lexicon dictionary = new EnglishLexicon();
}

// Good: injected
class SpellChecker {
    private final Lexicon dictionary;
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }
}
```

Benefits: flexibility, reusability, testability (pass a fake). Frameworks like Spring/Guice just automate this wiring.

---

### Item 6: Avoid creating unnecessary objects

Reuse an immutable object instead of creating an equivalent one every time.

```java
String s = new String("bikini"); // bad: new object every time
String s = "bikini";             // good: reuses the pooled literal
```

- Prefer static factories to constructors for immutable classes: `Boolean.valueOf("true")` over `new Boolean("true")`.
- **Expensive objects: cache them.** `String.matches` compiles a `Pattern` on every call; compile once:
```java
private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})...$");
static boolean isRomanNumeral(String s) { return ROMAN.matcher(s).matches(); }
```
- The `Person` example: the original creates `Calendar`/`Date` objects on every `isBabyBoomer()` call. Build them once in a `static` block:

```java
class Person {
    private final Date birthDate;
    private static final Date BOOM_START;
    private static final Date BOOM_END;

    static {
        Calendar gmtCal = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
        gmtCal.set(1946, Calendar.JANUARY, 1, 0, 0, 0);
        BOOM_START = gmtCal.getTime();
        gmtCal.set(1965, Calendar.JANUARY, 1, 0, 0, 0);
        BOOM_END = gmtCal.getTime();
    }

    public boolean isBabyBoomer() {
        return birthDate.compareTo(BOOM_START) >= 0 &&
               birthDate.compareTo(BOOM_END) < 0;
    }
}
```
- **Autoboxing trap:** `Long sum = 0L; for (...) sum += i;` creates ~2^31 `Long` objects. Prefer primitives (`long sum`).
- Don't overdo it: small, cheap objects are fine. Object pools are usually a bad idea (except for really heavy objects like DB connections). Contrast with *defensive copying* (Item 50), where reusing is the bug.

---

### Item 7: Eliminate obsolete object references

Java has GC, but memory leaks still happen when you keep a reference that will never be used again. Classic case: a class that manages its own memory, e.g. a stack backed by an array.

```java
public Object pop() {
    if (size == 0) throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // eliminate obsolete reference
    return result;
}
```
Without the null-out, popped elements stay referenced by the array and are never collected.

Other common sources of leaks:
- **Caches**: entries stay after they're useful. Use `WeakHashMap` (when entries live as long as the key is referenced elsewhere) or `LinkedHashMap` with eviction.
- **Listeners/callbacks** that are registered but never deregistered. Store them as weak references.

Nulling is the exception, not the norm. The best way is to let the variable fall out of scope (keep scopes small).

---

### Item 8: Avoid finalizers and cleaners

- Unpredictable, often dangerous, and unnecessary. No guarantee they run promptly, or at all.
- Slow: finalizers made object creation/destruction ~50x slower in the book's test.
- Security risk: a finalizer attack can resurrect a half-constructed object when its constructor throws.
- Don't use them to close files or release scarce resources (file handles, connections); GC timing is arbitrary and you can run out.

Instead: implement `AutoCloseable` and use try-with-resources (Item 9). Legit uses of cleaners are limited to a safety net if the caller forgets `close`, or freeing native peers.

---

### Item 9: Prefer try-with-resources to try-finally

```java
// Bad: nested try-finally, and an exception in close() hides the real one
static String firstLine(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}

// Good: shorter, closes automatically, keeps the first exception (others are "suppressed")
static String firstLine(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```
Works with anything implementing `AutoCloseable`. Multiple resources: `try (var in = ...; var out = ...) { ... }` - closed in reverse order.

---

## Chapter 3: Methods Common to All Objects

`equals`, `hashCode`, `toString`, `clone`, `compareTo`. They're non-final methods of `Object` designed to be overridden, and they have general contracts you must obey.

### Item 10: Obey the general contract when overriding `equals`

Don't override it if the default (identity) is right: each instance is unique (e.g. `Thread`), logical equality isn't needed, or a superclass already does it properly.

Override when the class has *value semantics* (`String`, `Integer`, `LocalDate`).

**Contract** (for non-null references):
- **Reflexive**: `x.equals(x)` is true
- **Symmetric**: `x.equals(y)` <=> `y.equals(x)`
- **Transitive**: x=y and y=z => x=z
- **Consistent**: repeated calls give the same result (don't depend on unreliable resources like network/IP)
- `x.equals(null)` is false

**Recipe**
1. `==` check for same reference (fast path)
2. `instanceof` check (also handles null)
3. Cast
4. Compare each significant field: `==` for primitives, `Float.compare`/`Double.compare` for floats, `Objects.equals` for objects

```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Point p)) return false;
    return x == p.x && y == p.y;
}
```

**Pitfalls**
- Overloading instead of overriding: `equals(Point p)` is not `equals(Object o)`. Always use `@Override`.
- Subclass that adds a value component breaks symmetry/transitivity (`Point` vs `ColorPoint`). There's no fix while keeping inheritance: **favor composition** (`ColorPoint` has a `Point` field).
- Always override `hashCode` too (Item 11).
- Easiest path: use a `record`, or let the IDE/Lombok generate it.

---

### Item 11: Always override `hashCode` when you override `equals`

**Contract**: equal objects must have equal hash codes. Otherwise `HashMap`/`HashSet` break:

```java
Map<Point, String> m = new HashMap<>();
m.put(new Point(1, 2), "a");
m.get(new Point(1, 2)); // null if hashCode isn't overridden - lands in a different bucket
```

Unequal objects *may* share a hash code, but distinct ones perform better (fewer collisions).

```java
@Override public int hashCode() {
    return Objects.hash(x, y); // simple; slightly slower (varargs + boxing)
}
```
Use the same fields as in `equals`. For hot paths, hand-roll: `31 * result + Integer.hashCode(field)`.

Tips: don't leave out significant fields to "speed up" hashing. If a class is immutable and hashing is costly, cache the value in a field. Never use a mutable field in the hash of an object stored in a `HashSet` - mutate it and the object is lost in the wrong bucket.

---

### Item 12: Always override `toString`

The default `Point@1b6d3586` is useless. A good `toString` makes logs, debugging and test failures readable.

```java
@Override public String toString() {
    return "Point(x=" + x + ", y=" + y + ")";
}
```
- Include all the interesting info.
- Decide whether to document the format. If you do, you're committing to it (and should provide a matching factory/constructor to parse it).
- Don't make callers parse the output; provide getters instead.
- Never include secrets (passwords, tokens) - it ends up in logs.

---

### Item 13: Override `clone` judiciously

`Cloneable` is a broken design: it has no `clone()` method, it only changes how `Object.clone()` behaves (field-by-field copy, no constructor call). `Object.clone()` is `protected`, extralinguistic, and the shallow copy shares mutable state.

Shallow copy problem:
```java
class Stack implements Cloneable {
    private Object[] elements;
    @Override public Stack clone() {
        try {
            Stack result = (Stack) super.clone();
            result.elements = elements.clone(); // deep-copy mutable fields, or both stacks share the array
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

**Prefer**: a copy constructor or copy factory.
```java
public Yum(Yum other) { ... }
public static Yum newInstance(Yum other) { ... }
```
They don't rely on a risky mechanism, don't conflict with `final` fields, can take interface types (`new HashSet<>(collection)`). Only arrays are cloned nicely with `clone()`.

---

### Item 14: Consider implementing `Comparable`

`compareTo` gives natural ordering: sorting, `TreeMap`/`TreeSet`, `Collections.sort`, `Arrays.sort` all use it.

**Contract**: returns negative/zero/positive for less/equal/greater. Same symmetry/transitivity rules as `equals`. Strongly recommended (not required): `x.compareTo(y) == 0` <=> `x.equals(y)`. `BigDecimal("1.0")` vs `("1.00")` violates it: a `HashSet` holds both, a `TreeSet` only one.

```java
public int compareTo(Person o) {
    int c = Integer.compare(age, o.age);   // compare primitives with static compare, NOT subtraction (overflow)
    if (c != 0) return c;
    return name.compareTo(o.name);         // most significant field first
}
```
Or use the comparator builders:
```java
private static final Comparator<Person> BY_AGE_NAME =
    Comparator.comparingInt((Person p) -> p.age).thenComparing(p -> p.name);
```
Avoid `<` and `>` operators in `compareTo`, and never `a - b` for ints (overflow).
