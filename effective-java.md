# Effective Java

Notes on *Effective Java* (Joshua Bloch, 3rd edition): 90 practical items on writing clear, correct, maintainable Java. Each item is a rule with the reasoning behind it. The notes below are in my own words with short examples.

## Chapters

| Ch | Title | Notes |
|----|-------|-------|
| 2 | Creating and Destroying Objects | done |
| 3 | Methods Common to All Objects | done |
| 4 | Classes and Interfaces | done |
| 5 | Generics | done |
| 6 | Enums and Annotations | done |
| 7 | Lambdas and Streams | done |
| 8 | Methods | done |
| 9 | General Programming | done |
| 10 | Exceptions | done |
| 11 | Concurrency | done |
| 12 | Serialization | done |

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

---

## Chapter 4: Classes and Interfaces

### Item 15: Minimize the accessibility of classes and members

Information hiding (encapsulation) decouples components so they can be developed, tested, changed and understood separately.

- Make each class and member as inaccessible as possible. Start `private`, widen only when needed.
- Top-level classes: package-private if only used in one package; `public` only if it is part of the exported API.
- Package-private/private nested classes used by one class only should be nested inside it.
- Public classes shouldn't have public mutable fields. Exception: `public static final` constants of primitives/immutable types.
- A `public static final` array (or other mutable object) is a security hole: callers can modify its contents. Fix it with an unmodifiable list or a defensive copy.
- With modules (Java 9), `exports` decides what is really visible; `public` in a non-exported package is hidden from other modules.

```java
// Bug: callers can do VALUES[0] = null
public static final Thing[] VALUES = { ... };

// Fix 1: unmodifiable view
private static final Thing[] PRIVATE_VALUES = { ... };
public static final List<Thing> VALUES = List.of(PRIVATE_VALUES);
// Fix 2: defensive copy
public static Thing[] values() { return PRIVATE_VALUES.clone(); }
```

---

### Item 16: In public classes, use accessor methods, not public fields

- Public fields expose representation: you can't change it, enforce invariants, or react when a field is accessed.
- Use private fields with getters (and setters only if the class is mutable).
- Exceptions: package-private or private nested classes can expose fields (less harm, less boilerplate). Public fields holding immutable values are less bad (`Point.x` in `java.awt` is a known mistake though).

```java
public class Point {
    private final double x, y;
    public Point(double x, double y) { this.x = x; this.y = y; }
    public double x() { return x; }
}
// Or simply: public record Point(double x, double y) {}
```

---

### Item 17: Minimize mutability

An immutable class cannot be modified after creation (`String`, `BigInteger`, boxed primitives, `java.time` types). They are simpler, thread-safe, shareable and make good map keys.

**Five rules**
1. No methods that modify state (no setters).
2. Class can't be extended (make it `final`, or use private constructors + static factories).
3. All fields `final`.
4. All fields `private`.
5. Exclusive access to mutable components: make defensive copies in constructors, accessors and `readObject`.

```java
public final class Complex {
    private final double re, im;
    public Complex(double re, double im) { this.re = re; this.im = im; }
    // "modifier" returns a NEW instance
    public Complex plus(Complex c) { return new Complex(re + c.re, im + c.im); }
}
```

- Pros: thread-safe without synchronization, can be freely shared (cache common instances), failure-atomic (no half-updated state), no defensive copies needed.
- Con: a separate object for each distinct value. If it's expensive, offer a mutable companion (`String` and `StringBuilder`).
- Rule: make small value classes immutable; for the rest, limit mutability as much as you can and make fields `final` unless there is a reason.
- Constructors should establish all invariants. Don't provide a second "init" method.

---

### Item 18: Favor composition over inheritance

Inheritance across package boundaries (extending a class you don't control) breaks encapsulation: the subclass depends on implementation details of the superclass, which can change.

```java
// Broken: HashSet.addAll() calls add() internally, so each element is counted twice
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;
    @Override public boolean add(E e) { addCount++; return super.add(e); }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);   // calls add() for each element => double counting
    }
}
```

**Fix: composition + forwarding (wrapper / decorator)**
```java
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;
    public InstrumentedSet(Set<E> s) { super(s); }
    @Override public boolean add(E e) { addCount++; return super.add(e); }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
}
// ForwardingSet implements Set<E>, holds a Set<E> field and forwards every method to it
```
- The new class has a private reference to an instance of the existing class and forwards calls to it.
- Works with any `Set` implementation and doesn't depend on the details of the wrapped class.
- Inheritance is appropriate only for a true "is-a" relationship, within the same package/control. Ask: does every `B` really have to be an `A`? Does `A` expose flaws you don't want to inherit?

---

### Item 19: Design and document for inheritance or else prohibit it

A class meant for inheritance must document its **self-use** of overridable methods (which methods call which) because subclasses depend on it.

- Document in Javadoc with `@implSpec`, what happens if you override a method.
- Provide `protected` hooks carefully (each is a commitment). Test by writing subclasses.
- **Constructors must not call overridable methods** (direct or indirect): the subclass override runs before the subclass constructor finishes, so its fields aren't initialized yet.

```java
public class Super {
    public Super() { overrideMe(); }       // BAD
    public void overrideMe() {}
}
public final class Sub extends Super {
    private final Instant instant;
    Sub() { instant = Instant.now(); }
    @Override public void overrideMe() { System.out.println(instant); } // prints null first!
}
```
- `Cloneable`/`Serializable` have the same trap: `clone`/`readObject` must not call overridable methods either.
- If you didn't design for inheritance, **prohibit it**: make the class `final` or give it private/package-private constructors with static factories.

---

### Item 20: Prefer interfaces to abstract classes

- A class can implement many interfaces but extend only one class. Interfaces are ideal for **mixins** (`Comparable`, `Serializable`) and for non-hierarchical type frameworks.
- Existing classes can be retrofitted to implement a new interface; they can't easily be made to extend a new abstract class.
- Combine both with a **skeletal implementation**: an interface that defines the type (plus default methods), and an abstract class `AbstractX` that implements most of the work (template method). Examples: `AbstractList`, `AbstractMap`.
- Default methods let you provide behavior in the interface, but they can't use `Object` methods (`equals`, `hashCode`) or state.

```java
public interface Shape { double area(); default String describe() { return "area=" + area(); } }
public abstract class AbstractShape implements Shape {
    @Override public String toString() { return describe(); }
}
```
- Skeletal classes are for implementers' convenience; the interface remains the real contract.

---

### Item 21: Design interfaces for posterity

- Default methods (Java 8) allow adding methods to interfaces, but there is no guarantee they work with all existing implementations. They can break at runtime.
- Example: `Collection.removeIf` default isn't safe for synchronized wrappers (`Collections.synchronizedCollection` from Apache Commons didn't synchronize it, which broke).
- Prefer to design a new interface carefully, with several implementations, **before** releasing it. It is hard to fix after.
- Use default methods to add behavior to existing interfaces only when needed, and test thoroughly.

---

### Item 22: Use interfaces only to define types

An interface should describe what clients can do with an instance of a class that implements it.

- Anti-pattern: the **constant interface** (an interface with only static final fields) leaks implementation details into the class's exported API and can't be undone.

```java
// Bad
interface PhysicalConstants { double AVOGADRO = 6.022e23; }

// Good: utility class with static imports, or an enum if values are related
public final class PhysicalConstants {
    private PhysicalConstants() {}
    public static final double AVOGADRO = 6.022e23;
}
// import static com.example.PhysicalConstants.*;
```

---

### Item 23: Prefer class hierarchies to tagged classes

A **tagged class** has a field (the tag) saying what "flavor" the instance is, and every method switches on it. It's verbose, error-prone and wastes memory.

```java
// Bad
class Figure {
    enum Shape { RECTANGLE, CIRCLE }
    final Shape shape;
    double length, width;   // only for rectangle
    double radius;          // only for circle
    double area() {
        switch (shape) { case RECTANGLE: return length * width; case CIRCLE: return Math.PI * radius * radius; default: throw new AssertionError(shape); }
    }
}

// Good: hierarchy (could also be a sealed interface + records in modern Java)
abstract class Figure { abstract double area(); }
class Circle extends Figure { final double radius; double area() { return Math.PI * radius * radius; } }
class Rectangle extends Figure { final double length, width; double area() { return length * width; } }
```
- Each type only has the fields it needs, `final` fields are possible, and adding a new type doesn't touch existing code.

---

### Item 24: Favor static member classes over nonstatic

A nested class is for use by its enclosing class. Four kinds:

| Kind | Use |
|------|-----|
| **Static member class** | Helper that doesn't need the outer instance (e.g. `Builder`, `Map.Entry`) |
| **Nonstatic member (inner) class** | Needs the enclosing instance (iterator of a collection, `Adapter`) |
| **Anonymous class** | One-off implementation, use lambdas where possible |
| **Local class** | Declared inside a method (rare) |

- Every inner (nonstatic) instance holds a hidden reference to the enclosing instance, which costs memory and can cause leaks (it prevents garbage collection of the outer object).
- **If a member class doesn't need access to the enclosing instance, declare it `static`.**

```java
public class MySet<E> extends AbstractSet<E> {
    @Override public Iterator<E> iterator() { return new MyIterator(); }
    private class MyIterator implements Iterator<E> { ... } // inner: needs the set's state
}
```

---

### Item 25: Limit source files to a single top-level class

- Never put multiple top-level classes in one source file. The compiler may accept it, but the result depends on the order the files are given to the compiler, which can cause duplicate-definition errors or silent behavior changes.
- If you want several, use static member classes inside one top-level class.

---

## Chapter 5: Generics

Generics (Java 5) let the compiler check types and remove casts. They are implemented by **erasure**: type info exists at compile time and is mostly removed at runtime.

### Item 26: Don't use raw types

A raw type is a generic type used without type arguments (`List` instead of `List<String>`). It bypasses compile-time type checks.

```java
// Raw: compiles, fails at runtime with ClassCastException
List stamps = new ArrayList();
stamps.add(new Coin());
Stamp s = (Stamp) stamps.get(0);

// Parameterized: error at compile time
List<Stamp> stamps = new ArrayList<>();
stamps.add(new Coin());   // compile error
```
- `List<Object>` is allowed to hold anything but is type-safe; raw `List` isn't checked. `List<String>` is **not** a subtype of `List<Object>`, but it is a subtype of raw `List`.
- When you don't care about the element type, use the **unbounded wildcard** `List<?>`: you can read as `Object` but can't add anything (except `null`).
- Exceptions where raw types are needed: class literals (`List.class`), and `instanceof` (`o instanceof Set` then cast to `Set<?>`).

---

### Item 27: Eliminate unchecked warnings

- Unchecked warnings show places where a `ClassCastException` may occur at runtime. Fix every one you can (use the diamond `<>` etc.).
- If you can't, but you can prove the code is type-safe, suppress with `@SuppressWarnings("unchecked")` on the **smallest scope possible** (a variable declaration or short method, not the whole class), and add a comment explaining why it is safe.

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // This cast is correct because the array we're creating has the same type as the one passed in, which is T[]
        @SuppressWarnings("unchecked") T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    ...
}
```

---

### Item 28: Prefer lists to arrays

| Arrays | Generics (lists) |
|--------|------------------|
| **Covariant**: `Sub[]` is a subtype of `Super[]` | **Invariant**: `List<Sub>` is not a subtype of `List<Super>` |
| **Reified**: know and enforce their element type at runtime | **Erased**: type info only at compile time |

```java
Object[] o = new Long[1];
o[0] = "I don't fit";     // compiles, throws ArrayStoreException at runtime

List<Object> l = new ArrayList<Long>();  // compile error: much better
```
- Because of this mismatch, arrays and generics don't mix: you can't create generic arrays (`new List<String>[1]`, `new E[...]`), which would not be type safe.
- If you get a generic array creation error or unchecked cast warnings, use `List<E>` instead of `E[]`. You may lose a bit of conciseness/performance but gain type safety.

---

### Item 29: Favor generic types

Make your own classes generic so clients don't have to cast.

```java
// Before: Object[] elements, clients must cast on pop()
// After:
public class Stack<E> {
    private E[] elements;
    @SuppressWarnings("unchecked")
    public Stack() { elements = (E[]) new Object[16]; } // can't do new E[16]
    public void push(E e) { elements[size++] = e; }
    public E pop() {
        E result = elements[--size];
        elements[size] = null;   // eliminate obsolete reference (Item 7)
        return result;
    }
}
```
Two ways to handle the generic array: (1) cast `Object[]` to `E[]` (as above; readable, but heap pollution risk), or (2) keep `Object[]` and cast each element read to `E`. The first is more common (once, in the constructor).
- Generic types can be bounded: `class DelayQueue<E extends Delayed>`.

---

### Item 30: Favor generic methods

Methods can be parameterized too. Static utility methods are good candidates.

```java
// Bad: raw types
public static Set union(Set s1, Set s2) { ... }

// Good
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```
- Type is inferred from the arguments, so no explicit type needed.
- **Recursive type bound**: express "something comparable to itself":
```java
public static <E extends Comparable<E>> E max(Collection<E> c) { ... }
```
- A **generic singleton factory** returns an immutable object usable for many types (`Collections.emptySet()`, `Collections.reverseOrder()`).

---

### Item 31: Use bounded wildcards to increase API flexibility

Parameterized types are invariant, so `Stack<Number>.pushAll(Iterable<Integer>)` fails. Wildcards fix this.

```java
// Producer: reads from the collection => extends
public void pushAll(Iterable<? extends E> src) { for (E e : src) push(e); }

// Consumer: writes to the collection => super
public void popAll(Collection<? super E> dst) { while (!isEmpty()) dst.add(pop()); }
```

**PECS: Producer-Extends, Consumer-Super.**
- If the parameter *produces* `T` values for you (you read from it), use `? extends T`.
- If it *consumes* `T` values (you write to it), use `? super T`.
- Don't use wildcards on return types (it forces clients to use wildcards too).
- `Comparable<T>`/`Comparator<T>` are always consumers: prefer `Comparable<? super T>`:
```java
public static <T extends Comparable<? super T>> T max(List<? extends T> list) { ... }
```
- Wildcard vs type parameter: if a type parameter appears only once in the signature, replace it with a wildcard.

---

### Item 32: Combine generics and varargs judiciously

A varargs parameter is an array, and a generic varargs `T...` creates a **non-reifiable** array, giving warnings and risk of **heap pollution** (a variable of a parameterized type refers to an object of a different type).

```java
static void dangerous(List<String>... stringLists) {
    Object[] objects = stringLists;
    objects[0] = List.of(42);              // heap pollution
    String s = stringLists[0].get(0);      // ClassCastException
}
```
- It's unsafe to store anything in a generic varargs array or to expose it to untrusted code.
- A generic varargs method is safe if it **doesn't store into the array** and **doesn't let the array escape** (only reads elements). Mark it `@SafeVarargs` (on static, final or private methods and constructors).
```java
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) { ... }
```
- Alternative: take a `List<T>` instead of varargs (e.g. `List.of(...)` is varargs-free for up to 10 args by overloads).

---

### Item 33: Consider typesafe heterogeneous containers

Normally a container (Set, Map) has a fixed number of type parameters. Sometimes you need a container with **many different types** of values, e.g. a database row with columns of different types or a favorites registry.

Trick: parameterize the **key** (not the container) with `Class<T>` as a type token.

```java
public class Favorites {
    private final Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void put(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), type.cast(instance)); // runtime check
    }
    public <T> T get(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}

Favorites f = new Favorites();
f.put(String.class, "Java");
f.put(Integer.class, 0xcafebabe);
String s = f.get(String.class);     // no cast needed
```
- The map's value type is `Object`, but the API guarantees type safety through the key.
- Limitations: can't be used with non-reifiable types (`List<String>.class` doesn't exist); a malicious client using raw `Class` can corrupt it, so `type.cast` guards the insert.
- You can restrict the types with a bounded type token: `Class<? extends Annotation>` (used in `AnnotatedElement.getAnnotation`).

---

## Chapter 6: Enums and Annotations

### Item 34: Use enums instead of `int` constants

The `int` enum pattern (`static final int APPLE_FUJI = 0;`) has no type safety, no namespace, brittle values compiled into clients, and no readable printing.

Java enums are full classes: one public static final instance per constant, can't be instantiated or extended, so they are **instance-controlled** (a generalization of singletons).

```java
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    EARTH  (5.975e+24, 6.378e6);

    private final double mass, radius;                 // fields are final
    Planet(double mass, double radius) { this.mass = mass; this.radius = radius; }
    double surfaceGravity() { return 6.67300E-11 * mass / (radius * radius); }
}
```
- Add data with fields and constructor; add behavior with methods. `values()` returns all constants; `toString()` is the name.
- **Constant-specific method bodies** when behavior differs per constant:
```java
public enum Operation {
    PLUS("+")  { public double apply(double x, double y) { return x + y; } },
    TIMES("*") { public double apply(double x, double y) { return x * y; } };
    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }
    public abstract double apply(double x, double y);   // compiler forces every constant to implement
}
```
- Avoid `switch` on `this` in an enum: adding a constant silently breaks it. Constant-specific methods are safer.
- **Strategy enum**: if several constants share behavior, nest a private enum (`PayType`) and delegate to it.
- Use enums any time you need a fixed set of constants known at compile time.

---

### Item 35: Use instance fields instead of ordinals

`ordinal()` is the position in the declaration. Don't derive a value from it: reordering or inserting constants silently changes the meaning, and you can't have two constants with the same value.

```java
// Bad
public int numberOfMusicians() { return ordinal() + 1; }

// Good
SOLO(1), DUET(2), TRIO(3);
private final int numberOfMusicians;
Ensemble(int size) { this.numberOfMusicians = size; }
```
`ordinal` is meant for enum-based data structures (`EnumSet`, `EnumMap`), not for application code.

---

### Item 36: Use `EnumSet` instead of bit fields

Bit fields (`STYLE_BOLD = 1 << 0`, combined with `|`) are hard to read when printed, have no type safety, and fix a max of 32/64 flags.

```java
public class Text {
    public enum Style { BOLD, ITALIC, UNDERLINE }
    public void applyStyles(Set<Style> styles) { ... }   // accept the interface (Item 64)
}
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```
`EnumSet` is internally a bit vector (a single `long` for up to 64 constants), so it is just as fast, with a full `Set` API.

---

### Item 37: Use `EnumMap` instead of ordinal indexing

Don't index an array by `ordinal()` (arrays aren't generic, you need unchecked casts, and an off-by-one bug gives wrong data silently).

```java
// Bad
Set<Plant>[] plantsByLifeCycle = (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);

// Good
Map<Plant.LifeCycle, Set<Plant>> byLifeCycle = new EnumMap<>(Plant.LifeCycle.class);
for (Plant.LifeCycle lc : Plant.LifeCycle.values()) byLifeCycle.put(lc, new HashSet<>());
for (Plant p : garden) byLifeCycle.get(p.lifeCycle).add(p);

// Even better with streams
Map<Plant.LifeCycle, List<Plant>> m = Arrays.stream(garden).collect(groupingBy(p -> p.lifeCycle, () -> new EnumMap<>(Plant.LifeCycle.class), toList()));
```
- `EnumMap` is as fast as an array (uses one internally) but type-safe and prints well.
- For a 2D relation (from-to, e.g. phase transitions), use `EnumMap<From, Map<To, Transition>>` instead of `int[][]` indexed by ordinals.

---

### Item 38: Emulate extensible enums with interfaces

Enums can't be extended (usually good), but sometimes you need extensible "operation codes". Define an interface and let enums implement it.

```java
public interface Operation { double apply(double x, double y); }

public enum BasicOperation implements Operation { PLUS("+") { public double apply(double x, double y) { return x + y; } }; ... }
public enum ExtendedOperation implements Operation { EXP("^") { public double apply(double x, double y) { return Math.pow(x, y); } }; ... }

// Use via bounded type token or bounded wildcard
static <T extends Enum<T> & Operation> void test(Class<T> opEnumType, double x, double y) {
    for (Operation op : opEnumType.getEnumConstants()) System.out.println(op.apply(x, y));
}
```
Downside: you can't inherit implementation from one enum to another.

---

### Item 39: Prefer annotations to naming patterns

Naming patterns (JUnit 3: test methods must start with `test`) fail silently on typos, can't ensure use on the right kind of element, and can't carry parameters.

```java
@Retention(RetentionPolicy.RUNTIME)   // keep at runtime
@Target(ElementType.METHOD)           // only on methods
public @interface Test {}

@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.METHOD)
public @interface ExceptionTest { Class<? extends Throwable> value(); }

// processing via reflection
for (Method m : testClass.getDeclaredMethods()) {
    if (m.isAnnotationPresent(Test.class)) { m.invoke(null); }
}
```
- **Marker annotation**: no parameters, just marks the element (`@Test`). Parameterized ones carry values; use `@Repeatable` (container annotation) for multiple values.
- Use the predefined annotations your tools provide (`@Override`, `@FunctionalInterface`, `@SafeVarargs`...). Few programmers need to define their own, but all should use them.

---

### Item 40: Consistently use the `@Override` annotation

It tells the compiler "this must override a supertype method", so a typo or wrong signature becomes a compile error rather than a silent bug.

```java
public boolean equals(Bigram b) { ... }       // overloads, doesn't override: compiles, HashSet breaks
@Override public boolean equals(Object o) { } // right
```
- Use it on every method that overrides a class method or implements an interface method (IDEs do this for you).
- Exception: concrete class implementing abstract methods may omit, but there's no harm adding it.

---

### Item 41: Use marker interfaces to define types

A **marker interface** has no methods and just marks that a class has some property (`Serializable`).

- Advantages over marker annotations: it defines a **type** (so the compiler catches errors; methods can accept `Serializable` as a parameter type), and it can be targeted more precisely (extend an interface to restrict it, e.g. `Set` is a marker-like restriction of `Collection`).
- Advantage of marker annotations: they can be applied to any element (classes, fields, methods) and are part of the larger annotation system.
- Rule: if the marker applies to **classes/interfaces and you'll write methods taking the marked type**, use a marker interface. Otherwise (e.g. applies to methods or fields), use an annotation.

---

## Chapter 7: Lambdas and Streams

### Item 42: Prefer lambdas to anonymous classes

```java
Collections.sort(words, new Comparator<String>() {         // verbose
    public int compare(String a, String b) { return Integer.compare(a.length(), b.length()); }
});
words.sort(comparingInt(String::length));                     // lambda + method ref + sort
```
- A lambda is a function object with little ceremony. Types are inferred, so don't write them unless clearer. (Use generics properly or the compiler can't infer: `List<String>`, not raw `List`.)
- Lambdas lack names and docs: if it doesn't self-explain or exceeds ~3 lines, don't use a lambda.
- Constant-specific enum bodies can use lambdas stored in a field (`PLUS("+", (x, y) -> x + y)`).
- Use anonymous classes only for: abstract class instances, interfaces with several abstract methods, and when the object needs a reference to itself (`this` in a lambda is the enclosing instance).
- Don't serialize lambdas (or anonymous classes).

---

### Item 43: Prefer method references to lambdas

Often shorter and clearer: `map.merge(key, 1, Integer::sum)` instead of `(count, incr) -> count + incr`.
- Name the intent: a method reference with a good name documents what happens.
- If the lambda is shorter or clearer, keep the lambda (`service -> action()` when the class name is long).

| Kind | Method ref | Lambda equivalent |
|------|------------|-------------------|
| Static | `Integer::parseInt` | `str -> Integer.parseInt(str)` |
| Bound (instance) | `Instant.now()::isAfter` | `t -> Instant.now().isAfter(t)` |
| Unbound | `String::toLowerCase` | `s -> s.toLowerCase()` |
| Class constructor | `TreeMap<K,V>::new` | `() -> new TreeMap<K,V>()` |
| Array constructor | `int[]::new` | `len -> new int[len]` |

---

### Item 44: Favor the use of standard functional interfaces

Before writing a new functional interface, check `java.util.function` (43 interfaces). Six basic ones, others are variants:

| Interface | Signature | Example |
|-----------|-----------|---------|
| `UnaryOperator<T>` | `T apply(T)` | `String::toLowerCase` |
| `BinaryOperator<T>` | `T apply(T, T)` | `BigInteger::add` |
| `Predicate<T>` | `boolean test(T)` | `Collection::isEmpty` |
| `Function<T,R>` | `R apply(T)` | `Arrays::asList` |
| `Supplier<T>` | `T get()` | `Instant::now` |
| `Consumer<T>` | `void accept(T)` | `System.out::println` |

- Primitive variants (`IntPredicate`, `LongBinaryOperator`) avoid boxing. Don't box with `Function<Integer,...>` in performance-sensitive code.
- Write your own only if: it's commonly used and a descriptive name helps, has a strong contract, or would benefit from default methods (e.g. `Comparator`).
- Always annotate your own functional interfaces with `@FunctionalInterface`.
- Be careful with overloaded methods taking different functional interfaces in the same argument position (`ExecutorService.submit(Callable)` vs `Runnable` give ambiguity).

---

### Item 45: Use streams judiciously

A stream pipeline: source, intermediate operations, one terminal operation. It is **lazy** and has no effect without a terminal operation.

```java
// Overuse: hard to read
words.collect(groupingBy(w -> w.chars().sorted().collect(StringBuilder::new, (sb, c) -> sb.append((char) c), StringBuilder::append).toString())) ...

// Balanced: helper method with a clear name
static String alphabetize(String s) { char[] a = s.toCharArray(); Arrays.sort(a); return new String(a); }
words.collect(groupingBy(w -> alphabetize(w)))
```
- Streams work well for: uniformly transforming sequences, filtering, combining, accumulating into a collection, searching.
- Iteration is better when you need: access to local variables from several stages, to modify local variables, `return`/`break`/`continue`, or throw checked exceptions.
- No `char` streams (`"hello".chars()` is an `IntStream`, cast needed when printing).
- Use helper methods, with meaningful names, to preserve readability. If in doubt, try both and choose the clearer.

---

### Item 46: Prefer side-effect-free functions in streams

Stream operations should be **pure** (result depends only on input and doesn't mutate shared state).

```java
// Bad: forEach is doing the accumulation (mutating external state)
Map<String, Long> freq = new HashMap<>();
words.forEach(w -> freq.merge(w.toLowerCase(), 1L, Long::sum));

// Good: collect does the work
Map<String, Long> freq = words.collect(groupingBy(String::toLowerCase, counting()));
```
- `forEach` should only report the result of the computation (print it), not compute.
- Know the `Collectors`: `toList`, `toSet`, `toMap`, `groupingBy`, `joining`, `counting`...
- `toMap` throws on duplicate keys; use the merge function overload:
```java
toMap(Artist::name, a -> a, (a, b) -> a)                // keep first
groupingBy(Album::artist, maxBy(comparing(Album::sales))) // best per group
```
- Use `import static java.util.stream.Collectors.*` for readability.

---

### Item 47: Prefer `Collection` to `Stream` as a return type

`Stream` doesn't extend `Iterable`, so you can't use it in for-each, and it can be consumed only once. A `Collection` can be iterated *and* streamed.

- Return `Collection` (or a `List`/`Set`) if the data fits in memory. `ArrayList.stream()` and `for (x : list)` both work.
- If the result is large but structured (a power set), return a custom `AbstractList`/`AbstractCollection` that computes elements on demand.
- If you can't, return `Stream` or `Iterable`. For a client wanting the other, adapters: `stream::iterator` as `Iterable` (`for (T t : (Iterable<T>) stream::iterator)`) or `StreamSupport.stream(iterable.spliterator(), false)`.

---

### Item 48: Use caution when making streams parallel

`parallel()` doesn't guarantee a speedup; it can cause **liveness failures** (hangs) and **wrong results** if functions violate the stream contract.

- A pipeline with `Stream.iterate` or `limit` usually doesn't parallelize (can't be split well, ordering).
- Performance is best on data structures that split cheaply: `ArrayList`, `HashMap`, `HashSet`, arrays, `int`/`long` ranges. Reduction ops (`reduce`, `min`, `max`, `count`, `anyMatch`) are fine, `collect` (mutable reduction) is poor.
- Predicates and functions must be non-interfering, stateless, and associative (accumulator).
```java
// Parallel gives a real speedup: counting primes with a range
static long pi(long n) { return LongStream.rangeClosed(2, n).parallel().mapToObj(BigInteger::valueOf).filter(i -> i.isProbablePrime(50)).count(); }
```
- Rule: don't parallelize without good reason and **measurement**. Done wrongly, parallel streams can produce wrong answers or crash.

---

## Chapter 8: Methods

### Item 49: Check parameters for validity

Validate at the start of a method and fail fast, with a clear exception, instead of failing later in a confusing way (or worse, corrupting state).

```java
/**
 * @param m the modulus, which must be positive
 * @throws ArithmeticException if m is less than or equal to 0
 */
public BigInteger mod(BigInteger m) {
    if (m.signum() <= 0) throw new ArithmeticException("Modulus <= 0: " + m);
    ...
}
```
- Document restrictions and exceptions (`IllegalArgumentException`, `NullPointerException`, `IndexOutOfBoundsException`) in Javadoc.
- `Objects.requireNonNull(arg, "message")` for null checks (also `checkIndex`).
- For non-public methods use `assert`: `assert a != null; assert offset >= 0;` (enabled with `-ea`; free if disabled).
- Exception: when validation is expensive or implicit in the computation (sorting a list checks comparability when it compares).

---

### Item 50: Make defensive copies when needed

Assume clients will try to break your class's invariants (by bug or on purpose). Mutable parameters can be modified after validation.

```java
public final class Period {
    private final Date start, end;
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());   // copy FIRST, then validate the copies
        this.end   = new Date(end.getTime());
        if (this.start.compareTo(this.end) > 0) throw new IllegalArgumentException();
    }
    public Date start() { return new Date(start.getTime()); }  // copy on return too
}
```
- Copy before validating, to avoid a time-of-check/time-of-use (TOCTOU) attack from another thread.
- Don't use `clone` on parameters of untrusted types (a subclass can return something nasty).
- Prefer immutable types (`Instant`, `LocalDateTime`) to avoid the need to copy at all.
- Don't copy if the class trusts its callers (document the transfer of ownership).

---

### Item 51: Design method signatures carefully

- **Choose names carefully**: follow conventions, be consistent with the rest of the package, and keep to well-known names.
- Don't go overboard with convenience methods: every method adds to learning, testing and maintenance cost.
- **Avoid long parameter lists** (4 or fewer). Long lists of same-type parameters are error-prone. Remedies: split the method, create helper classes to group parameters, or use the **builder** pattern (Item 2).
- Prefer interfaces to classes as parameter types (`Map` over `HashMap`).
- Prefer two-element enum types to `boolean` parameters: `Thermometer.newInstance(TemperatureScale.CELSIUS)` is clearer than `newInstance(true)`.

---

### Item 52: Use overloading judiciously

Overloads are chosen at **compile time** by the static type of the arguments, while overrides are chosen at runtime. This is confusing:

```java
public static String classify(Collection<?> c) { return "Collection"; }
public static String classify(Set<?> s)        { return "Set"; }
public static String classify(List<?> l)       { return "List"; }

for (Collection<?> c : new Collection<?>[] { new HashSet<String>(), new ArrayList<String>() })
    System.out.println(classify(c));   // prints "Collection" twice: static type decides
```
- Fix: one method with `instanceof` checks, or different names.
- Safe policy: **never export two overloads with the same number of parameters**. Use different names instead (`writeInt`, `writeLong`, like `ObjectOutputStream`).
- Autoboxing + generics make it worse: `list.remove(i)` vs `list.remove((Integer) i)` on a `List<Integer>`.
- Lambdas and method references make overloads taking different functional interfaces ambiguous (`submit(Callable)` vs `submit(Runnable)`).

---

### Item 53: Use varargs judiciously

Varargs creates an array on every call. If you require at least one argument, don't check at runtime; declare it:

```java
// Bad: fails at runtime when no args
static int min(int... args) { if (args.length == 0) throw new IllegalArgumentException(); ... }

// Good: first param is required, compile-time check
static int min(int firstArg, int... remainingArgs) { ... }
```
- In performance-critical code, add overloads for the first few arities (`foo()`, `foo(a)`, `foo(a,b)`, `foo(a,b,c,...)`) like `EnumSet.of` does.
- Don't retrofit varargs onto methods that take an array (can't tell by caller code).

---

### Item 54: Return empty collections or arrays, not nulls

Returning `null` forces every caller to add a null check, and forgetting causes an NPE.

```java
// Bad
public List<Cheese> getCheeses() { return cheesesInStock.isEmpty() ? null : new ArrayList<>(cheesesInStock); }

// Good
public List<Cheese> getCheeses() { return new ArrayList<>(cheesesInStock); }
// avoid allocating: use Collections.emptyList(), Set.of(), or a zero-length array constant
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];
return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
```
- Allocating an empty collection is rarely a performance issue; only optimize if profiling shows it.

---

### Item 55: Return optionals judiciously

When a method may not be able to return a value, options are: throw an exception (costly, for exceptional conditions), return `null` (callers must remember to check), or return `Optional<T>` (callers are forced to handle absence).

```java
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty()) return Optional.empty();
    E result = null;
    for (E e : c) if (result == null || e.compareTo(result) > 0) result = Objects.requireNonNull(e);
    return Optional.of(result);
}

String lastWord = max(words).orElse("No words...");
Toy t = max(toys).orElseThrow(TemporaryToyException::new);
```
- Never return `null` from a method that returns `Optional`.
- Don't wrap containers (`Optional<List<T>>`): return an empty collection instead.
- Don't use Optional for map values, collection elements, fields (usually), or parameters.
- Primitive variants `OptionalInt`, `OptionalLong`, `OptionalDouble`: don't use `Optional<Integer>` etc.
- Optional is itself an object: cost is allocation, so avoid in performance-critical places.

---

### Item 56: Write doc comments for all exposed API elements

Documentation is part of the API. Write a Javadoc comment for every exported class, interface, constructor, method and field.

- Method: a summary of **what** it does (contract), not **how**. Include preconditions (`@throws`), postconditions, side effects, thread-safety, and parameters (`@param`), return (`@return`), exceptions (`@throws`).
- Summary sentence appears in index: first sentence of the comment, distinct within the class, ending with a period.
- Use `{@code ...}` for code (escapes `<`, `>`), `{@literal}`, `{@implSpec}` (for implementation requirements in classes designed for inheritance), `{@inheritDoc}`.
- Generics: document all type parameters; enums: document constants; annotations: document all members.
- Document thread safety and serializability.

```java
/**
 * Returns the element at the specified position in this list.
 *
 * <p>This method is <i>not</i> guaranteed to run in constant time.
 *
 * @param index index of element to return; must be non-negative and less than the size of this list
 * @return the element at the specified position
 * @throws IndexOutOfBoundsException if the index is out of range ({@code index < 0 || index >= this.size()})
 */
E get(int index);
```

---

## Chapter 9: General Programming

### Item 57: Minimize the scope of local variables

- Declare a variable **where it is first used**, ideally with an initializer. Declaring at the top of a block invites misuse and stale values.
- Prefer `for` loops over `while` for iteration: the loop variable's scope is limited to the loop, which prevents copy-paste bugs.
- Keep methods small and focused, so local scopes are small.

```java
// Good: i is scoped to the loop; cannot be reused by accident
for (Iterator<Element> i = c.iterator(); i.hasNext(); ) { Element e = i.next(); ... }

// Bug-prone: a second loop copy-pasted uses 'i2' and compiles even if you meant 'i'
Iterator<Element> i = c.iterator(); while (i.hasNext()) { ... }
Iterator<Element> i2 = c2.iterator(); while (i.hasNext()) { ... }  // oops: i not i2
```
- Another idiom: cache the limit when it's expensive: `for (int i = 0, n = expensiveComputation(); i < n; i++)`.

---

### Item 58: Prefer for-each loops to traditional `for` loops

The for-each loop hides the iterator or index, which removes a class of bugs (typos, off-by-one, mixing up nested iterators).

```java
// Bug: next() called twice per inner iteration, throws NoSuchElementException
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); )
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
        deck.add(new Card(i.next(), j.next()));

// Fix
for (Suit suit : suits) for (Rank rank : ranks) deck.add(new Card(suit, rank));
```
Works on arrays and anything implementing `Iterable`. Can't be used when:
- **Destructive filtering**: removing elements as you go (use `removeIf` or `Iterator.remove`).
- **Transforming**: replacing elements in a list or array (need the index).
- **Parallel iteration**: stepping several collections in lockstep.

---

### Item 59: Know and use the libraries

Don't reinvent the wheel: library code is tested, optimized, and improves with each release.

```java
// Flawed hand-rolled random: skewed distribution, may return negative values
// Use the library
int n = ThreadLocalRandom.current().nextInt(bound);   // not Random for new code (Java 7+)
```
- Every programmer should know `java.lang`, `java.util` (collections framework, `java.util.function`, streams), and `java.io`.
- Use `Collections`, `Arrays`, `Objects`, `Math.floorMod`, `Math.addExact`, `List.of`, `Files.readAllLines`, `String.join`, `Optional`.
- Prefer library functionality to custom code and benefit from future improvements. Check the release notes for new features.

---

### Item 60: Avoid `float` and `double` if exact answers are required

`float` and `double` use binary floating point: good for engineering/science approximations, but they can't represent 0.1 exactly. **Never use them for money or exact decimal values.**

```java
System.out.println(1.03 - 0.42);   // 0.6100000000000001
```
- Use `BigDecimal` (exact, rounding control, but slower and clumsier), `int`, or `long` (count cents).
- With `BigDecimal`, create from `String`, not `double`: `new BigDecimal("0.10")`.

```java
// Using long cents
long itemsBought = 0; long funds = 100;   // cents
for (long price = 10; funds >= price; price += 10) { funds -= price; itemsBought++; }
```
- Use `int` up to 9 digits, `long` up to 18, `BigDecimal` beyond or if you need rounding modes.

---

### Item 61: Prefer primitive types to boxed primitives

Differences: boxed types have **identity** (so `==` compares references), can be `null`, and are slower and larger.

```java
Comparator<Integer> naturalOrder = (i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
naturalOrder.compare(new Integer(42), new Integer(42));   // returns 1! i == j compares references

// Fix: unbox into locals first
(iBoxed, jBoxed) -> { int i = iBoxed, j = jBoxed; return i < j ? -1 : (i == j ? 0 : 1); }
// or just Comparator.naturalOrder() / Integer.compare
```
```java
static Integer i;          // default null
if (i == 42) ...           // NullPointerException (auto-unboxing null)

Long sum = 0L;
for (long i = 0; i <= Integer.MAX_VALUE; i++) sum += i;   // creates ~2^31 Long objects (slow)
```
- Use boxed types for type parameters (`List<Integer>`), reflection and when you need `null`.
- When mixing primitives and boxed in an operation, the boxed is auto-unboxed: NPE risk.

---

### Item 62: Avoid strings where other types are more appropriate

Strings are poor substitutes for:
- **Other value types**: parse input into `int`, `boolean`, enum, etc.
- **Enums**: use an enum instead of string constants.
- **Aggregate types**: `String key = className + "#" + i;` is fragile; make a class (`record Key(String className, int i)`).
- **Capabilities** (access tokens): a string key for thread-local variables is global and can collide; use an unforgeable key object (`ThreadLocal<T>`).

---

### Item 63: Beware the performance of string concatenation

Strings are immutable, so `s += x` copies both strings: **O(n²)** when done repeatedly in a loop.

```java
// Slow for many items
String result = "";
for (Item i : items) result += i.toString();

// Fast
StringBuilder b = new StringBuilder(numItems * LINE_WIDTH);
for (Item i : items) b.append(i);
return b.toString();
```
- A few concatenations (`"a" + b + "c"`) are fine and compiled efficiently (modern JVMs use `invokedynamic`).
- Inside loops, use `StringBuilder`, `String.join`, or `Collectors.joining`.

---

### Item 64: Refer to objects by their interfaces

If appropriate interface types exist, use them for parameters, return values, variables and fields.

```java
Set<Son> sonSet = new LinkedHashSet<>();   // good
LinkedHashSet<Son> sonSet = new LinkedHashSet<>();  // avoid
```
- You can change the implementation later by changing one line.
- Use a class type only if no suitable interface exists (value classes like `String`, `BigInteger`; framework class hierarchies; classes with extra methods not in the interface like `PriorityQueue.comparator()`).
- If the implementation changes the semantics (e.g. `TreeSet` to `HashSet`), the type still compiles but behaves differently: check the contract.

---

### Item 65: Prefer interfaces to reflection

Reflection (`java.lang.reflect`) lets a program use classes not known at compile time, but you lose:
- compile-time type checking (errors appear at runtime),
- clean code (verbose, with checked exceptions),
- performance (slower than normal calls).

```java
// Create an instance reflectively but use it through an interface
Class<? extends Set<String>> cl = (Class<? extends Set<String>>) Class.forName(args[0]);
Constructor<? extends Set<String>> cons = cl.getDeclaredConstructor();
Set<String> s = cons.newInstance();   // from here on, normal interface calls
```
- Use reflection only for **instantiation** (and then access with an interface or superclass), such as in service frameworks, DI containers, and class-loading plugins.
- Real needs: class browsers, test runners, serialization/ORM libraries. Avoid in application code.

---

### Item 66: Use native methods judiciously

JNI lets Java call C/C++. It was useful to access platform-specific facilities and legacy libraries, and for performance, but today the JVM is fast and most platform access is available in Java.
- Native code isn't memory-safe (memory corruption), is harder to debug, less portable, and has a cost crossing the boundary.
- Use only to access legacy libraries or platform features that Java can't. (In newer Java, the Foreign Function & Memory API is the modern alternative.)

---

### Item 67: Optimize judiciously

> "Premature optimization is the root of all evil." (Knuth)

Three aphorisms: don't sacrifice sound architecture for performance; strive to write good programs, not fast ones; **measure** before and after optimizing.

- Good design usually leads to good performance. Bad design is often impossible to fix later.
- Think about performance when designing APIs: public mutable types, inheritance instead of composition, implementation types instead of interfaces can lock in poor performance.
- Don't warp an API to get speed.
- Profile (e.g. JMH for microbenchmarks, a profiler for production hotspots) to find real bottlenecks. Guessing is usually wrong.

---

### Item 68: Adhere to generally accepted naming conventions

| Type | Examples |
|------|----------|
| Package/module | `com.google.common.collect` (lowercase, dot-separated) |
| Class/interface | `Timer`, `HttpServlet`, `Comparator` (UpperCamelCase) |
| Method/field | `remove`, `ensureCapacity`, `isDigit` (lowerCamelCase) |
| Constant | `MIN_VALUE`, `NEGATIVE_INFINITY` (UPPER_SNAKE, `static final`) |
| Local variable | `i`, `denom`, `houseNumber` |
| Type parameter | `T` (type), `E` (element), `K`/`V` (key/value), `X` (exception), `R` (return) |

- Boolean methods: `isEmpty`, `hasSiblings`. Conversions: `toString`, `asList`. Getters: `getX` (or `x()` for records).
- Follow the conventions: they make code readable to others.

---

## Chapter 10: Exceptions

### Item 69: Use exceptions only for exceptional conditions

Don't use exceptions for ordinary control flow.

```java
// Terrible: loops until ArrayIndexOutOfBoundsException
try { int i = 0; while (true) range[i++].climb(); } catch (ArrayIndexOutOfBoundsException e) {}

// Right
for (Mountain m : range) m.climb();
```
- Exceptions are slower (stack trace creation), obscure the logic, can mask real bugs from inside the `try`.
- API design: a **state-testing method** (`hasNext()`) with a state-dependent one (`next()`), or an **observable return** (`Optional`, `null`) rather than throwing in a normal case.
- Use state-testing if there's no concurrent access; use the observable return in concurrent contexts or when state-testing is expensive.

---

### Item 70: Use checked exceptions for recoverable conditions and runtime exceptions for programming errors

Three kinds of throwables:
| Kind | Use |
|------|-----|
| **Checked** (`Exception` subclasses) | Recoverable conditions; the caller can reasonably recover (retry, alternative) |
| **Runtime** (`RuntimeException`) | **Programming errors** (precondition violations): `IllegalArgumentException`, `IllegalStateException`, `NullPointerException` |
| **Errors** (`Error`) | Reserved for the JVM (`OutOfMemoryError`); don't subclass or throw |

- Provide information with the checked exception that helps recovery (accessors, not only message).
- Convention: all unchecked throwables should subclass `RuntimeException`. Don't extend `Throwable` directly.

---

### Item 71: Avoid unnecessary use of checked exceptions

Checked exceptions force callers to handle them, and a method that throws one can't be used in streams. Overuse makes APIs painful.

- Use checked only if: (1) the caller can't prevent the condition, **and** (2) the caller can take a useful recovery action.
- Otherwise, use an unchecked exception.
- Ways to remove a checked exception: return `Optional`, or split the method into a state-testing method + an action method (Item 69).

```java
// Instead of: try { obj.action(args); } catch (TheCheckedException e) { ... }
if (obj.actionPermitted(args)) { obj.action(args); } else { /* handle */ }
```

---

### Item 72: Favor the use of standard exceptions

Reuse JDK exceptions: they make APIs easier to learn and read.

| Exception | Use |
|-----------|-----|
| `IllegalArgumentException` | Non-null argument has an inappropriate value |
| `IllegalStateException` | Object's state is inappropriate for the call |
| `NullPointerException` | Null where not allowed |
| `IndexOutOfBoundsException` | Index out of range |
| `ConcurrentModificationException` | Concurrent modification detected |
| `UnsupportedOperationException` | Object doesn't support the method |
| `ArithmeticException`, `NumberFormatException` | Arithmetic / parsing problems |

Don't reuse `Exception`, `RuntimeException`, `Throwable` or `Error` directly. If the argument is null, use NPE rather than IAE. If both an arg and state are wrong, pick `IllegalStateException`.

---

### Item 73: Throw exceptions appropriate to the abstraction

A higher layer shouldn't leak lower-level exceptions. **Exception translation**: catch the low-level exception and throw one that fits your abstraction.

```java
public E get(int index) {
    try { return listIterator(index).next(); }
    catch (NoSuchElementException e) { throw new IndexOutOfBoundsException("Index: " + index); }
}
```
**Exception chaining**: pass the cause for debugging:
```java
catch (LowerLevelException cause) { throw new HigherLevelException(cause); }
```
- Best: avoid the problem by making sure lower layers succeed (validate arguments upfront).
- Next best: handle (or log) the lower-level exception without propagating it.

---

### Item 74: Document all exceptions thrown by each method

- Declare each checked exception individually with `@throws` (don't write `throws Exception`).
- Document unchecked exceptions with `@throws` as well (not in the `throws` clause), as they describe the preconditions.
- If the same exception is thrown for the same reason by many methods, document it in the **class** comment (e.g. "all methods throw `NullPointerException` if a parameter is null").

---

### Item 75: Include failure-capture information in detail messages

The exception's detail message is what you see in the stack trace in a bug report. Include **all parameters and fields** that contributed to the failure.

```java
public IndexOutOfBoundsException(int lowerBound, int upperBound, int index) {
    super(String.format("Lower bound: %d, Upper bound: %d, Index: %d", lowerBound, upperBound, index));
    this.lowerBound = lowerBound; this.upperBound = upperBound; this.index = index;   // accessors for programmatic use
}
```
- Don't include passwords or sensitive keys.
- Make the constructor take the details, so the message is built consistently.
- Not user-facing text; keep it for programmers (no localization).

---

### Item 76: Strive for failure atomicity

A failed method call should leave the object in the state before the call.

- Make objects **immutable** (trivial).
- **Check parameters first**, before modifying anything.
- Order the computation so that the parts that can fail come before parts that modify state.
- Operate on a temporary copy and replace the original when done (e.g. `Collections.sort` copies to an array).
- Write recovery code (rollback), rarely used.
- Not always achievable (concurrent modification of a collection) or worthwhile; at least document the resulting state.

---

### Item 77: Don't ignore exceptions

An empty `catch` block defeats the purpose of exceptions.

```java
try { ... } catch (SomeException e) { }   // bad: swallows the failure
```
- If you truly ignore it, put a comment saying why and name the variable `ignored`:
```java
try { numColors = f.get(1L, TimeUnit.SECONDS); }
catch (TimeoutException | ExecutionException ignored) { /* fall back on default */ }
```
- Same for `Future` result exceptions or `close()` failures that you swallow.

---

## Chapter 11: Concurrency

### Item 78: Synchronize access to shared mutable data

`synchronized` gives two things: **mutual exclusion** (one thread at a time) **and reliable communication** (**visibility**: changes by one thread become visible to others). Without synchronization, a thread may never see another's update.

```java
// Broken: may loop forever, since the JIT can hoist the read of stopRequested out of the loop
private static boolean stopRequested;
new Thread(() -> { int i = 0; while (!stopRequested) i++; }).start();
stopRequested = true;

// Fix 1: synchronized getter and setter (both required)
// Fix 2: volatile (visibility only, no atomicity)
private static volatile boolean stopRequested;
```
- `volatile` guarantees visibility but not atomicity: `nextSerialNumber++` is read-modify-write, so it needs `synchronized` or `AtomicLong`.
```java
private static final AtomicLong nextSerialNum = new AtomicLong();
public static long generateSerialNumber() { return nextSerialNum.getAndIncrement(); }
```
- Best: **don't share mutable data**. Share immutable data or confine mutable data to one thread.
- Safe publication: pass objects via `static` init, `volatile`, `final`, a lock, or a concurrent collection.

---

### Item 79: Avoid excessive synchronization

Too much synchronization causes reduced performance, deadlock, or nondeterministic behavior.

- **Never call an alien method (overridable or client-provided) from within a synchronized region.** It may try to take another lock (deadlock), or modify the data you're iterating (`ConcurrentModificationException`).
```java
// Fix: do alien calls outside the lock, on a snapshot
List<SetObserver<E>> snapshot;
synchronized (observers) { snapshot = new ArrayList<>(observers); }
for (SetObserver<E> o : snapshot) o.added(this, element);
```
- Or use `CopyOnWriteArrayList` for observer lists.
- Do as little work as possible inside synchronized regions.
- Cost isn't the lock itself but contention and lost parallelism.
- A class is either thread-safe internally (with fine-grained locking) or relies on callers. If you're unsure, don't synchronize, document it (like `StringBuilder` vs `StringBuffer`).

---

### Item 80: Prefer executors, tasks, and streams to threads

Don't write your own work queues. Use the **executor framework**.

```java
ExecutorService exec = Executors.newSingleThreadExecutor();
exec.execute(runnable);
exec.shutdown();   // graceful termination
```
- Choose the pool: `newCachedThreadPool` (light servers, but unbounded), `newFixedThreadPool` (heavily loaded), or `ThreadPoolExecutor` directly for more control.
- A **task** (`Runnable`/`Callable`) is the unit of work; the executor decides how to run it. Also `ScheduledThreadPoolExecutor` (replaces `Timer`) and `ForkJoinPool` (parallel streams use it).
- Try to avoid creating `Thread` directly.

---

### Item 81: Prefer concurrency utilities to `wait` and `notify`

`wait`/`notify` are hard to get right. Use the higher-level `java.util.concurrent` tools:

- **Concurrent collections**: `ConcurrentHashMap` (use `putIfAbsent`, `computeIfAbsent`, `merge`), `BlockingQueue` (producer-consumer), `CopyOnWriteArrayList`. Don't wrap with `Collections.synchronizedMap` for concurrency: slower.
- **Synchronizers**: `CountDownLatch` (wait for N events), `Semaphore`, `CyclicBarrier`, `Phaser`, `Exchanger`.
```java
CountDownLatch ready = new CountDownLatch(concurrency);
CountDownLatch start = new CountDownLatch(1);
// each worker: ready.countDown(); start.await(); action.run(); done.countDown();
ready.await(); start.countDown(); done.await();
```
- For timing, use `System.nanoTime` (not `currentTimeMillis`).
- If you must use `wait`: always call it **in a loop** that tests the condition (guards against spurious wakeups), and prefer `notifyAll` to `notify`.
```java
synchronized (obj) { while (!condition) obj.wait(); /* act */ }
```

---

### Item 82: Document thread safety

Don't rely on the `synchronized` modifier appearing in Javadoc: it's an implementation detail. Document the **level** of thread safety:

| Level | Meaning | Example |
|-------|---------|---------|
| Immutable | Never changes, no sync needed | `String`, `Long`, `BigInteger` |
| Unconditionally thread-safe | Mutable, but internal sync | `AtomicLong`, `ConcurrentHashMap` |
| Conditionally thread-safe | Some methods need external sync | `Collections.synchronized*` iterators |
| Not thread-safe | Caller must synchronize | `ArrayList`, `HashMap` |
| Thread-hostile | Unsafe even with external sync | (rare, usually bugs) |

- For public locks, document them. Better to use a **private lock object** to prevent a client from interfering (denial of service):
```java
private final Object lock = new Object();
public void foo() { synchronized (lock) { ... } }
```
- Make the lock `final`.

---

### Item 83: Use lazy initialization judiciously

"Don't do it unless you need to." Lazy init delays the cost until first use, but complicates the code and can make it slower.

- Normal initialization is preferred: `private final FieldType field = computeFieldValue();`
- To break a circular initialization problem or for performance, use:
  - **Synchronized accessor** (simple, default choice),
  - **Lazy initialization holder class** idiom (for static fields):
```java
private static class FieldHolder { static final FieldType field = computeFieldValue(); }
private static FieldType getField() { return FieldHolder.field; }   // class loads on first access
```
  - **Double-check idiom** (instance fields; field must be `volatile`):
```java
private volatile FieldType field;
private FieldType getField() {
    FieldType result = field;
    if (result == null) {
        synchronized (this) {
            if (field == null) field = result = computeFieldValue();
        }
    }
    return result;
}
```

---

### Item 84: Don't depend on the thread scheduler

The OS scheduler decides which runnable threads run; it varies across JVMs and platforms. A program whose **correctness or performance** depends on the scheduler isn't portable.

- Keep the number of **runnable** threads close to the number of processors (threads should do useful work, not wait).
- Don't **busy-wait** (spinning on a shared variable) — it burns CPU:
```java
while (!ready) { }   // terrible
```
- Don't rely on `Thread.yield` (no testable semantics) or thread priorities (non-portable). Use `Thread.sleep(1)` rather than `yield` in tests if really needed, but better to restructure.

---

## Chapter 12: Serialization

Serialization converts an object graph to a byte stream (`ObjectOutputStream`) and back (`ObjectInputStream`). It is **dangerous**: this chapter mostly says "avoid it".

### Item 85: Prefer alternatives to Java serialization

- **Attack surface**: deserialization (`readObject`) can instantiate arbitrary classes on the classpath and run their code. Attackers build "gadget chains" from library classes to execute code, cause DoS (deserialization bombs), etc. This has caused many real vulnerabilities.
- Simplest defense: **don't deserialize untrusted data**.
- Use **cross-platform structured-data** formats: **JSON** or **Protocol Buffers** (protobuf, with schema; strongly typed and more efficient).
- If you can't avoid it: use object **deserialization filters** (`ObjectInputFilter`, Java 9) as an allowlist of acceptable classes.
- Don't use Java serialization for new systems.

---

### Item 86: Implement `Serializable` with great caution

Implementing `Serializable` is not free:
- **Decreases flexibility**: the serialized form becomes part of the exported API, so private fields are effectively public and changing the class is constrained.
- **Increases bugs and security holes**: deserialization is a hidden constructor that bypasses normal construction and must preserve invariants.
- **Increases testing burden** across versions.
- Always declare an explicit `serialVersionUID` (`private static final long serialVersionUID = 1L;`), otherwise it is computed from the class and any change breaks compatibility.
- Classes designed for inheritance shouldn't implement it; interfaces shouldn't extend it (it forces all implementers).
- Inner (non-static) classes shouldn't be serializable.
- If you must make a class extendable and serializable, provide an accessible no-arg constructor in the first non-serializable superclass.

---

### Item 87: Consider using a custom serialized form

Don't accept the **default serialized form** without thought: it exposes the physical representation.

```java
// Default form of a linked list: serializes every node recursively, can overflow the stack
private static class Entry { String data; Entry next; Entry previous; }

// Custom: write only the logical data
private void writeObject(ObjectOutputStream s) throws IOException {
    s.defaultWriteObject();
    s.writeInt(size);
    for (Entry e = head; e != null; e = e.next) s.writeObject(e.data);
}
```
- The default form is OK only if the physical representation equals the logical content (e.g. a `Name` with 3 strings).
- Mark fields that aren't part of the logical state `transient`.
- If using the default form, `readObject` must still validate invariants.
- Synchronize `writeObject` if the class has synchronized methods.
- Always declare an explicit `serialVersionUID`.

---

### Item 88: Write `readObject` methods defensively

`readObject` acts like a **public constructor** that takes a byte stream: an attacker can craft bytes to violate invariants or retain references to internal objects.

```java
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // Defensive copies of mutable components (before validation)
    start = new Date(start.getTime());
    end   = new Date(end.getTime());
    // Check invariants
    if (start.compareTo(end) > 0) throw new InvalidObjectException(start + " after " + end);
}
```
- Don't use `clone` for defensive copies here.
- Field-by-field: for classes with `final` fields (can't be reassigned in `readObject`), use a serialization proxy instead (Item 90).
- Don't call overridable methods from `readObject`.

---

### Item 89: For instance control, prefer enum types to `readResolve`

A singleton that implements `Serializable` stops being a singleton: deserialization creates a **new instance**. `readResolve` replaces the deserialized object with the canonical one:

```java
private Object readResolve() { return INSTANCE; }   // all fields must be transient or it can be attacked
```
- Weakness: if any instance field is non-transient object reference, an attacker can capture the object before `readResolve` runs.
- **Best**: use a single-element **enum**: the JVM guarantees one instance, including against deserialization and reflection.
```java
public enum Elvis { INSTANCE; private String[] favoriteSongs = { "Hound Dog" }; }
```

---

### Item 90: Consider serialization proxies instead of serialized instances

Serialization proxy pattern: a tiny, private, static nested class that represents the logical state of the enclosing class, and is serialized instead.

```java
public final class Period implements Serializable {
    private final Date start, end;
    public Period(Date start, Date end) { ... validation ... }

    private static class SerializationProxy implements Serializable {
        private final Date start, end;
        SerializationProxy(Period p) { this.start = p.start; this.end = p.end; }
        private Object readResolve() { return new Period(start, end); }   // goes through the public constructor
        private static final long serialVersionUID = 234098243823485285L;
    }

    private Object writeReplace() { return new SerializationProxy(this); }   // serialize the proxy

    private void readObject(ObjectInputStream stream) throws InvalidObjectException {
        throw new InvalidObjectException("Proxy required");   // block attacks on the real class
    }
}
```
- Deserialization calls the normal constructor, so all invariants and validation are enforced; no bypass.
- Works with `final` fields and with classes where a normal approach isn't safe (stops "bogus byte-stream" and "internal field theft" attacks).
- Limitations: not compatible with classes that clients can extend, and some object graphs with cycles; slightly slower (extra object).
- Use it whenever you must write a `readObject` or `writeObject` and can't use the default form safely.
