# [Effective Java](https://www.goodreads.com/book/show/105099.Effective_Java_Programming_Language_Guide)

## Creating and destroying objects

### Consider static factory methods instead of constructors

```java
public static Boolean valueOf(boolean b) {
  return b ? Boolean.TRUE : Boolean.FALSE;
}
```

Advantages:
* They have names.
* They are not required to create a new object each time they are invoked.
* They can return an object of any subtype of their return type.
* The class of the returned object can vary from call to call as a function of the input parameters.
* The class of the returned object need to exist when the class containing the method is written.

Disadvantages:
* Classes that provide only static factory methods without public or protected constructors cannot be subclassed. This is preferable when using composition instead of inheritance.
* They are hard for programmers to find. Some conventions
  - `from`, as type-conversion method `Date d = Date.from(instant)`
  - `of`, an aggregation method `Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING)`
  - `valueOf`, more verbose alternative than `from` and `of`, `BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE)`
  - `instance` or `getInstance`, returns and instance that cannot be said to have the same values, `StackWalker luke = StackWalker.getInstance(options)`
  - `create` or `newInstance` the method guarantees each call returns a new instance, `Object newArray = Array.newInstance(classObject, arrayLength)`
  - `get`**Type**, like `getInstance`but when factory method is in a different class `FileStore fs = Files.getFileStore(path)`
  - `new`**Type**, like `newInstance` but when factory method is in a different class `BufferedReader br = Files.newBufferedReader(path)`
  - **type**, a concise alternative to `get`_Type_ and `new`_Type_, `List<Compliant> litany = Collections.list(legacyLitany)`

### Consider builder when faced with many constructor parameters

```java
public class NutritionFacts {
    private final int servingSize;  // (mL)            required
    private final int servings;     // (per container) required
    private final int calories;     // (per serving)   optional
    private final int fat;          // (g/serving)     optional
    private final int sodium;       // (mg/serving)    optional
    private final int carbohydrate; // (g/serving)     optional

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize  = servingSize;
        this.servings     = servings;
        this.calories     = calories;
        this.fat          = fat;
        this.sodium       = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```

The telescoping constructor pattern works, but it is hard to write client code when there are many parameters, and harder still to read it.

Avoid using JavaBeans pattern as it might leave the object in an inconsistent state partway through its construction. It does also precludes the possibility of making the class immutable.

```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        private final int servingSize;
        private final int servings;

        private int calories      = 0;
        private int fat           = 0;
        private int sodium        = 0;
        private int carbohydrate  = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings    = servings;
        }

        public Builder calories(int val)
            { calories = val;      return this; }
        public Builder fat(int val)
            { fat = val;           return this; }
        public Builder sodium(int val)
            { sodium = val;        return this; }
        public Builder carbohydrate(int val)
            { carbohydrate = val;  return this; }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
    .calories(100).sodium(35).carbohydrate(27).build();
```

The Builder pattern simulates named optional parameters and it is well suited to class hierarchies. **It is good choice when designing classes whose constructors or static factories would have more than a handful of parameters.**

### Enforce the singleton property with a private constructor or an enum type

Beware that making a class a singleton can make it difficult to test its clients because it's impossible to substitute a mock implementation /

**Singleton with public final field**, this approach is simpler and it makes it clear that the class is a singleton.

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }

    public void leaveTheBuilding() { ... }
}
```

**Singleton with static factory**

```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }

    public void leaveTheBuilding() { ... }
}
```

**Enum singleton**, similar to public field approach but more concise. It provides serialisation machinery for free and provides guarantee against multiple instantiation even on serialisation and reflection attacks. **This is the best way of implementing a singleton.**

```java
public enum Elvis {
    INSTANCE;

    public void leaveTheBuilding() { ... }
}
```

### Enforce noninstantiability with a private constructor

Ocassionally you'll want to write a class that is a grouping of static methods. People abuse them in order to avoid thinking in terms of objects, but they have valud use cases.

Such _utility clases_ are not designed to be instantiated.

Attempting to enforce noninstantiability by making the class abstract does not work. **A class can be made noninstantiable by including a private constructor.**

```java
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }
    // Remainder omitted
}
```

### Prefer dependency injection to hardwiring resources

Innapropiate use of static utility, inflexible and untestable.

```java
public class SpellChecker {
    private static final Lexicon dictionary = ...;

    private SpellChecker() {} // Noninstantiable

    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}
```

Dependency injection provides flexibility and testability

```java
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

A useful variant of the pattern is to pass a resource _factory_ to the constructor so the object can be called repeatedly to create instance of a type.

```java
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```

### Avoid creating unnecessary objects

```java
String s = new String("bikini");
```

Performance can be greatly improved

```java
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
        + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```

Reusing expensive object for improved performance

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})" +
        "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```

Be careful with autoboxing, as it blurs the distinction betwen primitive and boxed primitive types. The following code is hideously slow, can you spot object creation?

```java
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;

    return sum;
}
```

Prefer primitives to boxed primitives, and watch out for unintentional autoboxing.

### Eliminate obsolete object references

Can you find the _memory leak_?

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    /**
     * Ensure space for at least one more element, roughly
     * doubling the capacity each time the array needs to grow.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

Don't forget to null out references once they become obsolete!

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();

    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;

}
```

Be mindful about this, **nulling out object references should be the exception rather than the norm.** The best way to eliminate an obsolete reference is to let the variable that contained the reference fall out of scope.

Whenever a class manages its own memory, the programmer should be alert for memory leaks.

Common sources of memory leaks are caches, listeners and callbacks.

### Avoid finalisers and cleaners

Finalisers are unpredictable, often dangerous, and generally unnecessary. Cleaners are less dangerous than finalisers, but still unpredictable, slow, and generally unnecessary.

Never do anything time-critical in a finalisers or cleaners. There is no guarantee they'll be executed promptly.

Never depend on a finaliser or cleaner to update persistent state. The language specification provides no guarantee that finalisers or cleaners will run at all.

There is a sever performance penalty for using finalisers and cleaners.

Finalisers have a serious security problem: they open your class up to finaliser attacks. Throwing an exception from a constructor should be sufficient to prevent an object form coming into existence; in the presence of finalisers, it is not. To protect nonfinal classes from finaliser attacks, write a final `finalize` method that does nothing.

Instead of writing a finaliser or cleaner, just have your class implement `AutoCloseable`.

### Prefer `try`-with-resources to `try-finally`

`try-finally` is no longer the best way to close resources.

```java
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}
```

`try-finally` is ugly when used with more than one resource.

```java
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}
```

`try`-with-resources is the best way to close resources.

```
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
       return br.readLine();
    }
}
```

`try`-with-resources on multiple resources, short and sweet.

```
static void copy(String src, String dst) throws IOException {
    try (InputStream   in = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
}
```

## Methods common to all objects

### Obey the general contract when overriding `equals`

* Each instance of the class is inherently unique
* there is no need for the class to provide a "logical equality" test
* A superclass has overriden `equals`, and the superclass behaviour is appropiate for this class
* The class is private or package-private, and you are certain that its `equals` method will never be invoked.

`equals` method implements _equivalence relation_ for any non-null reference values:
* **Reflexive**: `x.equals(x)` must return `true`
* **Symmetric**:  `x.equals(y)` returns `true` if `y.equals(x)` returns `true`
* **Transitive**:  `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` must return `true`
* **Consistent**:  `x.equals(y)` should consistently return `true` or consistently return `false`
* `x.equals(null)` must return `false`

Do not write an `equals` method that depends on unreliable resources.

All objects objects must be unequal to `null`.

A few more caveats:
* Always override `hashCode` when you override `equals`.
* Don't try to be too clever
* Don't substitute another type for `Object` in the `equals` declaration.

### Always override `hashCode` when you override equals

Equal objects must have equal hash codes.

Do not be tempted to exclude significant fields from the hash code computation to improve performance.

Don't provide a detailed specification for the value returned by `hashCode`, so clients can't reasonably depend on it; this gives you the flexibility to change it.

### Always override `toString`

Providing a good `toString` implementation makes your class much more pleasant to use and makes systems using the class easier to debug.

When practical, the `toString` method should return all the interesting information in the object.

Whether or not you decide to specify the format, you should clearly document your intentions.

### Override `clone` judiciously

In practice, a class implementing `Cloneable` is expected to provide a properly functioning public `clone` method.

Immutable classes should never provide a `clone` method, because that would only encourage wasteful copying.

The `clone` method functions as a constructor; you must ensure that it does no harm to the original object and that it properly establishes invariants on the clone.

The `Cloneable` architecture is incompatible with normal use of final fields referring to mutable objects.

Public `clone` methods should omit the `throws` clause, as methods that don't throw checked exceptions are easier to use.

A better approach to object copying is to provide a copy constructor or copy factory.

```java
public Yum(Yum yum) { ... }
```

```java
public static Yum newInstance(Yum yum) { ... };
```

Advantages of using copy constructor or copy factory:
* They don't rely on a risk-prone extralinguistic object creation mechanism.
* They don't demand unenforceable adherence to thinly documented conventions.
* They don't conflict with the proper use of final fields.
* They don't throw unnecessary checked exceptions.
* They don't require casts.
* They can take an argument whose type is an interface implemented by the class.
* Interface-based copy constructors and factories (_conversion_ constructors and _conversion_ factories) allow the client to accept the implementation type of the original.

### Consider implementing `Comparable`

The notation `sgn`(expression) designates the mathematical _signum_ function, which is defined to return -1, 0, or 1

* The implementor must ensure that `sgn(x.compareTo(y)) == -sgn(y. compareTo(x))` for all `x` and `y`. Implies that `x.compareTo(y)` must throw an exception if and only if `y.compareTo(x)` throws an exception.
* Transitive: ``(x. compareTo(y) > 0 && y.compareTo(z) > 0)`` implies `x.compareTo(z) > 0`.
* `x.compareTo(y) == 0` implies that `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`, for all `z`.
* It is strongly recommended, but not required, that `(x.compareTo(y) == 0) == (x.equals(y))`.

Use of the relational operators `<` and `>` in compareTo methods is verbose and error-prone and no longer recommended.

If a class has multiple significant fields, the order in which you compare them is critical. Start with the most significant field and work your way down.

The `Comparator` interface is outfitted with a set of comparator construction methods, which enable fluent construction of comparators. Many programmers prefer the conciseness of this approach, though it does come at a modest performance cost.

```java
private static final Comparator<PhoneNumber> COMPARATOR =
    comparingInt((PhoneNumber pn) -> pn.areaCode)
        .thenComparingInt(pn -> pn.prefix)
        .thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```

**Avoid comparators based on differences, they violate transitivity**

```java
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();
    }
};
```

## Classes and interfaces

### Minimise the accessibility of classes and members

A well-designed component hides all its implementation details, cleanly separating their API from its implementation.

Information hiding or _encapsulation_ is important because:
* _Decouples_ the component that comprise the system, allowing them to be developed, tested, optimised, used, understood and modified in isolation.
* Speeds up development because components can be developed in parallel.
* Eases the burden of maintenance because components can be understood more quickly and debugged or replaced with little fear or harming other components.
* Components can be optimised without affecting correctness of others.
* Increases software reuse because components that aren't tightly coupled often prove useful in other contexts.
* Decreases the risk in building large systems because individual components may prove successful even if the system does not.

Make each class or member as inaccessible as possible.

Instance fields of public classes should rarely be public. Classes with public mutable fields are generally thread-safe.

Nonzero-length array is always mutable, so it is wrong for a class to have a public static final array field, or accessor that returns such a field. It is a potential security hole.

```java
public static final Thing[] VALUES = { ... };
```

Alternatively, return a copy of a private array

```java
private static final Thing[] PRIVATE_VALUES = { ... };

public static final List<Thing> VALUES =
   Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```

### In public classes, use accessor methods, not public fields

Degenerate classes like this should not be public

```java
class Point {
   public double x;
   public double y;
}
```

Because the data fields of such classes are accessed directly, these classes do not offer the benefits of _encapsulation_. You can't change the representation without changing the API, you can't enforce invariants, and you can't take auxiliary action when a field is accessed.

If a class is accessed outside its package, provide accessor methods.

If a class is package-private or is a private nested class, there is nothing inherently wrong with exposing its data fields.

### Minimise mutability

An immutable class is simply a class whose instances cannot be modified. **Immutable classes are easier to design, implement, and use than mutable classes. They are less prone to error and are more secure.**

1. Don't provide methods that modify the object's state (mutators).
2. Ensure that the class can't be extended.
3. Make all fields final.
4. Make all fields private.
5. Ensure exclusive access to any mutable components.

Returning new objects on operations is known as _functional approach_ because methods return the result of applying a function to their operand, without modifying it. Contrast it to the _procedural_ or _imperative_ approach in which methods apply a procedure to their operand, causing its state to change. **Method names in immutable objects are prepositions (such as `plus`) rather than names (such as `add`).**

Advantages of immutable objects:
* Simple.
* Inherently thread-safe; they require no synchronisation.
* Can be shared freely.
* Not only you can share immutable objects, but they can share their internals.
* Make great building blocks for other objects.
* Provide failure atomicity for free. There is no possibility of temporary inconsistency.

The main disadvantage immutable objects have is that they require a separate object for each distinct value.

Classes should be immutable unless there's a very good reason to make them mutable.

If a class cannot be made immutable, limit its mutability as much as possible. Declare every field `private final` unless there's a good reason to do otherwise.

Constructors should create fully initialised objects with all of their invariants established.

### Favour composition over inheritance

Using inheritance inappropriately lead to fragile software. It is safe to use inheritance within a package where programmers have classes and subclasses under control. It is safe to use inheritance when extending classes specifically designed and documented for extension.

Unlike method invocation, implementation inheritance violates encapsulation. Subclasses depend on the implementation details of its superclass for its proper function.

To avoid fragility use composition and forwarding instead of inheritance, especially if an appropriate interface to implement a wrapper exists. Wrapper classes are not only more robust than subclasses, they are also more powerful.

### Design and document for inheritance or else prohibit it

The class must document its self-use of overridable methods.

A class may have to provide hooks into its internal workings in the form of judiciously chosen protected methods.

**The only way to test a class designed for inheritance is to write subclasses.**

You must test your class by writing subclasses before you release it.

Constructors must not invoke overridable methods. Superclass constructor runs before the subclass constructor. If the overriding method depends on any initialisation performed by the subclass constructor, the method will not behave as expected.

```java
public class Super {
    // Broken - constructor invokes an overridable method
    public Super() {
        overrideMe();
    }

    public void overrideMe() {
    }
}

public final class Sub extends Super {
    // Blank final, set by constructor
    private final Instant instant;

    Sub() {
        instant = Instant.now();
    }

    // Overriding method invoked by superclass constructor
    @Override public void overrideMe() {
        System.out.println(instant);
    }

    public static void main(String[] args) {
        Sub sub = new Sub();//null
        sub.overrideMe();//instant
    }
}
```

Same restriction applies to `clone` and `readObject`, they should not invoke overridable methods, directly or indirectly.

Designing a class for inheritance requires great effort and places substantial limitations on the class.

The best solution to this problem is to prohibit subclassing in classes that are not designed and documented to be safely subclassed. There are two ways of doing this. Declare the class final or make all constructors private or package-private and to add public static factories instead of constructors.

### Prefer interfaces to abstract classes

Existing classes can easily be retrofitted to implement a new interface.

Interfaces are ideal for defining _mixings_, a type that a class can implement in addition to its "primary type". For example Comparable is a mixing that allows a class can be ordered.

Interfaces allow for the construction of nonhierarchical type frameworks.

Interfaces enable safe, powerful functionality enhancements via the wrapper class idiom. If you use abstract classes, you leave the programmer who wants to add functionality with no alternative than to use inheritance.

You can combine the advantages of interfaces and abstract classes by providing an abstract _skeletal implementation class_ to go with an interface. The interface defines the type, while the skeletal implementation class implements the primitive interface methods (_Template Method_ pattern).

Concrete implementation built on top of an skeletal implementation.

```java
static List<Integer> intArrayAsList(int[] a) {
    Objects.requireNonNull(a);

    // The diamond operator is only legal here in Java 9 and later
    // If you're using an earlier release, specify <Integer>
    return new AbstractList<>() {
        @Override public Integer get(int i) {
            return a[i];  // Autoboxing (Item 6)
        }

        @Override public Integer set(int i, Integer val) {
            int oldVal = a[i];
            a[i] = val;     // Auto-unboxing
            return oldVal;  // Autoboxing
        }

        @Override public int size() {
            return a.length;
        }
    };
}
```
Skeletal implementation classes provide implementation assistance of abstract classes without imposing constraints as type definitions.

Good documentation is essential in a skeletal implementation.

### Design interfaces for posterity

It is not always possible to write a default method that maintains all the invariants of every conceivable implementation.

In the presence of default methods, existing implementations of an interface may compile without error or warning but fail at runtime.

Is still of the utmost importance to design interfaces with great care. While it may be possible to correct some interface flaws after an interface is released, you cannot count on it.

### Use interfaces only to define types

Constant interface anti-pattern (only constants) are a poor use of interfaces. Implementing a constant interface causes this implementation detail to leak into the class's exported API.

```java
public interface PhysicalConstants {
    static final double AVOGADROS_NUMBER   = 6.022_140_857e23;
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    static final double ELECTRON_MASS      = 9.109_383_56e-31;
}
```

If the constants are strongly tied to a class or interface, you should add them there. If not, a utility class are a good choice

```java
public class PhysicalConstants {
  private PhysicalConstants() { }  // Prevents instantiation

  public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
  public static final double BOLTZMANN_CONST  = 1.380_648_52e-23;
  public static final double ELECTRON_MASS    = 9.109_383_56e-31;
}
```

### Prefer class hierarchies to tagged classes

Tagged class, vastly inferior to a class hierarchy.

```java
class Figure {
    enum Shape { RECTANGLE, CIRCLE };

    // Tag field - the shape of this figure
    final Shape shape;

    // These fields are used only if shape is RECTANGLE
    double length;
    double width;

    // This field is used only if shape is CIRCLE
    double radius;

    // Constructor for circle
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    // Constructor for rectangle
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area() {
        switch(shape) {
          case RECTANGLE:
            return length * width;
          case CIRCLE:
            return Math.PI * (radius * radius);
          default:
            throw new AssertionError(shape);
        }
    }
}
```

Tagged classes are verbose, error-prone and inefficient.

A tagged class is just a pallid imitation of a class hierarchy.

With class hierarchy

```java
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;

    Circle(double radius) { this.radius = radius; }

    @Override double area() { return Math.PI * (radius * radius); }
}

class Rectangle extends Figure {
    final double length;
    final double width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width  = width;
    }

    @Override double area() { return length * width; }
}
```

### Favour static member classes over nonstatic

Typical use of a nonstatic member class

```java
public class MySet<E> extends AbstractSet<E> {
    ... // Bulk of the class omitted

    @Override public Iterator<E> iterator() {
        return new MyIterator();
    }

    private class MyIterator implements Iterator<E> {
        ...
    }

}
```

If you declare a member class that does not require access to an enclosing instance, always put the `static` modifier in its declaration. If you omit this modifier, each instance will have a hidden extraneous reference to its enclosing instance and it will take time and space.

### Limit source files to a single top-level class

Use static member classes instead of multiple top-level classes. If you do otherwise, your program might not be able to compile or run properly.

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }

    private static class Utensil {
        static final String NAME = "pan";
    }

    private static class Dessert {
        static final String NAME = "cake";
    }
}
```

Never put multiple top-level classes or interfaces in a single source file.

## Generics

### Don't use raw types

```java
private final Collection stamps = ... ;
```

If you use raw types, you lose all the safety and expressiveness benefits of generics.

```java
private final Collection<Stamp> stamps = ... ;
```

You lose type safety if you use a raw type such as `List`, but not if you use a parameterised type such as `List<Object>`.

This method works but it uses raw types, which is dangerous.

```java
static int numElementsInCommon(Set s1, Set s2) {
    int result = 0;
    for (Object o1 : s1)
        if (s2.contains(o1))
            result++;
    return result;
}
```

Better to use wildcard type.

```java
static int numElementsInCommon(Set<?> s1, Set<?> s2) { ... }
```

You can put _any_ element into a collection with a raw type, easily corrupting the collection type invariant. **You can't put any element (other than null) into a `Collection<?>`**

There are a couple of exceptions to the use of raw types
* You must use raw types in class literals, as `List.class` is legal, but `List<String>.class` is not.
* Raw types is the preferred way to use the `instanceof` operator with generic types
    ```java
    if (o instanceof Set) {       // Raw type
        Set<?> s = (Set<?>) o;    // Wildcard type
        ...
    }
    ```

### Eliminate unchecked warnings

Eliminate every unchecked warning that you can. That will ensure your code is typesafe.

If you can't eliminate a warning, but you can prove that the code that provoked the warning is typesafe, then (and only then) suppress the warning with an `@SupressWarnings("unchecked")` annotation.

Always use the `SuppressWarnings` annotation on the smallest scope possible.

Every time you use a `@SuppressWarnings("unchecked")` annotation, add a comment saying why it is safe to do so.

### Prefer lists to arrays

Arrays are _covariant_, which means that if a `Sub` is a subtype of `Super`, then array type `Sub[]` is a subtype of the array type `Super[]`. Generics, by contrast, are _invariant_, for any two distinct types `Type1` and `Type2`, `List<Type1>` is neither a subtype or a supertype of `List<Type2>`.

Arrays are deficient, this code fragment is legal and it fails at runtime

```java
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in"; // Throws ArrayStoreException
```

This one won't compile

```java
List<Object> ol = new ArrayList<Long>(); // Incompatible types
ol.add("I don't fit in");
```

### Favour generic types

Generic types are safer and easier to use than types that require casts in client code.When you design new types, make sure that they can be used without such casts.

### Favour generic methods

Generic methods, like generic types, are safer and easier to use than methods requiring their clients to put explicit casts on input parameters and return types.

### Use bounded wildcards to increase API flexibility

Wildcard type for a parameter that serves as an E producer

```java
public void pushAll(Iterable<? extends E> src) {
    for (E e : src)
        push(e);
}
```

This would make `Stack<Number>` work also on `stack.pushAll(intVal)`, as `Integer` is a subtype of `Number`.

Sometimes we would want to do the opposite, have a wildcard type for parameter that serves as an `E` consumer.

```java
public void popAll(Collection<? super E> dst) {
    while (!isEmpty())
        dst.add(pop());
}
```

So we can do

```java
Collection<Object> objects = ...;
numberStack.popAll(objects);
```

For maximum flexibility, use wildcard types on input parameters that represent producers or consumers.

_PECS_ stands for producer-`extends` and consumer-`super`.

Do not use bounded wildcard types as return types. It would force wildcard types on client code.

**If the user of a class has to think about wildcard types, there is probably something wrong with its API.**

If a type parameter appears only once in a method declaration, replace it  with a wildcard.

### Combine generics and varargs judiciously

Mixing generics and varargs can violate type safety.

```java
static void dangerous(List<String>... stringLists) {
    List<Integer> intList = List.of(42);
    Object[] objects = stringLists;
    objects[0] = intList;             // Heap pollution
    String s = stringLists[0].get(0); // ClassCastException
}
```

It is unsafe to store a value in a generic varargs array parameter.

The `SafeVarargs` annotation constitutes a promise by the author of a method that it is typesafe.

This is unsafe, it exposes a reference to its generic parameter array. The compiler might not have enough information to make an accurate determination of the type of the argument and it can propagate heap pollution up the stack.

```java
static <T> T[] toArray(T... args) {
    return args;
}
```

It is unsafe to give another method access to a generic varargs parameter array.

Safe method with a generic varargs parameter
```java
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

Use `@SafeVarargs` on every method with a varargs parameter of a generic or parameterised type.

A generic varargs method is safe if:
1. Does not store anything in the varargs parameter array
2. Does not make the array (or a clone) visible to untrusted code.

Use `List` as a typesafe alternative to a generic varargs parameter

```java
static <T> List<T> flatten(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

### Consider typesafe heterogeneous containers

Typesafe heterogeneous container pattern API

```java
public class Favorites {
    public <T> void putFavorite(Class<T> type, T instance);
    public <T> T getFavorite(Class<T> type);
}
```

Typesafe heterogeneous container pattern client

```java
public static void main(String[] args) {

    Favorites f = new Favorites();

    f.putFavorite(String.class, "Java");

    f.putFavorite(Integer.class, 0xcafebabe);

    f.putFavorite(Class.class, Favorites.class);


    String favoriteString = f.getFavorite(String.class);

    int favoriteInteger = f.getFavorite(Integer.class);

    Class<?> favoriteClass = f.getFavorite(Class.class);

    System.out.printf("%s %x %s%n", favoriteString,

        favoriteInteger, favoriteClass.getName());

}
```

A `Favorites` instance is _typesafe_: it will never return an `Integer` when you ask it for a `String`. It is also _heterogeneous_: unlike an ordinary map, all the keys are of different types. Therefore, we call `Favorites` a _typesafe heterogeneous container_.

Typesafe heterogeneous container pattern - implementation

```java
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

Achieving runtime type safety with a dynamic cast

```java
public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(type, type.cast(instance));
}
```

## Enums and annotations

### Use enums instead of int constants

The _`int` enum pattern_ provides nothing in the way of type safety and little in the way of expressive power.

```java
public static final int APPLE_FUJI         = 0;
public static final int APPLE_PIPPIN       = 1;
public static final int APPLE_GRANNY_SMITH = 2;
public static final int ORANGE_NAVEL  = 0;
public static final int ORANGE_TEMPLE = 1;
public static final int ORANGE_BLOOD  = 2;
```

```java
public enum Apple  { FUJI, PIPPIN, GRANNY_SMITH }
public enum Orange { NAVEL, TEMPLE, BLOOD }
```

Java's enum types are classes that export one instance for each enumeration constant via a public static field num. Enum are effectively final because they have no accessible constructors.

They are type safe, if you declare a parameter to be of type `Apple`, you are guaranteed that any non-null object reference passed to the parameter is one of the three valid `Apple` values.

Enum types let you add arbitrary methods, provide high-quality implementations of all the `Object` methods, they implement `Comparable` and `Serializable`.

To associate data with enum constants, declare instance fields and write constructor that takes the data and stores it in the fields.

Enum type that switches on its own value, questionable.

```java
public enum Operation {
    PLUS, MINUS, TIMES, DIVIDE;

    // Do the arithmetic operation represented by this constant
    public double apply(double x, double y) {
        switch(this) {
            case PLUS:   return x + y;
            case MINUS:  return x - y;
            case TIMES:  return x * y;
            case DIVIDE: return x / y;
        }
        throw new AssertionError("Unknown op: " + this);
    }
}
```

Enum type with constant-specific method implementations

```java
public enum Operation {
  PLUS  {public double apply(double x, double y){return x + y;}},
  MINUS {public double apply(double x, double y){return x - y;}},
  TIMES {public double apply(double x, double y){return x * y;}},
  DIVIDE{public double apply(double x, double y){return x / y;}};

  public abstract double apply(double x, double y);
}
```

Enum type with constant-specific class bodies and data

```java
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };

    private final String symbol;

    Operation(String symbol) { this.symbol = symbol; }

    @Override public String toString() { return symbol; }

    public abstract double apply(double x, double y);
}
```

**Use enums any time you need a set of constants whose members are known at compile time.** It is not necessary that the set of constants in an enum type stay fixed for all time, the enum was specifically designed to allow evolution of enum types.

### Use instance fields instead of ordinals

Abuse of ordinal to derive an associated value, **don't do this**.

```java
public enum Ensemble {
    SOLO,   DUET,   TRIO, QUARTET, QUINTET,
    SEXTET, SEPTET, OCTET, NONET,  DECTET;

    public int numberOfMusicians() { return ordinal() + 1; }
}
```

Never derive a value associated with an enum from its ordinal; store it in an instance field instead.

```java
public enum Ensemble {
    SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
    SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
    NONET(9), DECTET(10), TRIPLE_QUARTET(12);

    private final int numberOfMusicians;

    Ensemble(int size) { this.numberOfMusicians = size; }
    public int numberOfMusicians() { return numberOfMusicians; }
}
```

### Use `EnumSet` instead of bit fields

Bit field enumeration constants, obsolete.

```java
public class Text {
    public static final int STYLE_BOLD          = 1 << 0;  // 1
    public static final int STYLE_ITALIC        = 1 << 1;  // 2
    public static final int STYLE_UNDERLINE     = 1 << 2;  // 4
    public static final int STYLE_STRIKETHROUGH = 1 << 3;  // 8

    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStyles(int styles) { ... }
}
```

This representation lets you use the bitwise `OR` operation to combine several constants into a set, known as a _bit field_

```java
text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

_Bit fields_ have all the disadvantages than `int` enum pattern. A better alternative is to use `EnumSet`

```java
public class Text {
    public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }

    // Any Set could be passed in, but EnumSet is clearly best
    public void applyStyles(Set<Style> styles) { ... }
}
```

So you could use it like

```java
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

**Just because an enumerated type will be used in sets, there is no reason to represent it with bit fields.**

### Use `EnumMap` instead of ordinal indexing

```java
class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }

    final String name;
    final LifeCycle lifeCycle;

    Plant(String name, LifeCycle lifeCycle) {
        this.name = name;
        this.lifeCycle = lifeCycle;
    }

    @Override public String toString() {
        return name;
    }
}
```

Using `ordinal()` to index into an array, **don't do this**.

```java
Set<Plant>[] plantsByLifeCycle =
    (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];

for (int i = 0; i < plantsByLifeCycle.length; i++)
    plantsByLifeCycle[i] = new HashSet<>();

for (Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);

// Print the results
for (int i = 0; i < plantsByLifeCycle.length; i++) {
    System.out.printf("%s: %s%n",
        Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
}
```

Using an `EnumMap` to associate data with an enum

```
Map<Plant.LifeCycle, Set<Plant>>  plantsByLifeCycle =
    new EnumMap<>(Plant.LifeCycle.class);
for (Plant.LifeCycle lc : Plant.LifeCycle.values())
    plantsByLifeCycle.put(lc, new HashSet<>());
for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);
System.out.println(plantsByLifeCycle);
```

Or even a shorter version, a bit naive as it doesn't produce an `EnumMap`

```java
System.out.println(Arrays.stream(garden)
    .collect(groupingBy(p -> p.lifeCycle)));
```

Fixed version

```java
System.out.println(Arrays.stream(garden)
    .collect(groupingBy(p -> p.lifeCycle,
        () -> new EnumMap<>(LifeCycle.class), toSet())));
```

**It is rarely appropriate to use ordinals to index into arrays: use EnumMap instead**.

### Emulate extensible enums with interfaces

In the case for _opcodes_, _operation codes_ that represent operations in some machine, is desirable to let the user of an API provide their own operations, effectively extending the set of operations provided by the API.

You can do this by implementing interfaces.

```java
public interface Operation {
    double apply(double x, double y);
}

public enum BasicOperation implements Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };
    private final String symbol;

    BasicOperation(String symbol) {
        this.symbol = symbol;
    }

    @Override public String toString() {
        return symbol;
    }
}
```

And an emulated extension enum

```java
public enum ExtendedOperation implements Operation {
    EXP("^") {
        public double apply(double x, double y) {
            return Math.pow(x, y);
        }
    },
    REMAINDER("%") {
        public double apply(double x, double y) {
            return x % y;
        }
    };

    private final String symbol;

    ExtendedOperation(String symbol) {
        this.symbol = symbol;
    }

    @Override public String toString() {
        return symbol;
    }
}
```

Now you can use your new operations anywhere you could use the basic operations, provided that API's are written to take the interface type `Operation`, not the implementation `BasicOperation`.

```java
private void test(Collection<? extends Operation> opSet);
```

While you cannot write an extensible enum type, you can emulate it by writing an interface to accompany a basic enum type that implements the interface.

### Prefer annotations to naming patterns

It's been common to use _naming patterns_ to indicate that some program elements demanded special treatment by a tool or framework. Like prefixing your tests with `test` word, so JUnit 3 would pick it up. These have many disadvantages:
1. Typographical errors result in silent failures. Imagine you accidentally named a test method `tsetSomething` instead of `testSomething`. JUnit 3 wouldn't complain, but it wouldn't run the test either.
2. There is no way to ensure that they are used only on appropriate program elements. Imagine creating a class `TestSafetyMechanism` in the hopes that JUnit 3 would automatically test its methods.
3. They provide no good way to associate parameter values with program elements. Suppose you want to support a category of test that succeeds only if it throws a particular exception, the exception type is the parameter of the test. You could encode the exception type in the test name but the compiler would have no way of knowing to check the string that was supposed to name the exception.

Annotations solve all these problems nicely, and JUnit adopted them starting with release 4.

Marker annotation type declaration

```java
import java.lang.annotation.*;

/**
 * Indicates that the annotated method is a test method.
 * Use only on parameterless static methods.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test {
}
```

Program to process marker annotations

```java
import java.lang.reflect.*;

public class RunTests {
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;
        Class<?> testClass = Class.forName(args[0]);
        for (Method m : testClass.getDeclaredMethods()) {
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " failed: " + exc);
                } catch (Exception exc) {
                    System.out.println("Invalid @Test: " + m);
                }
            }
        }
        System.out.printf("Passed: %d, Failed: %d%n",
                          passed, tests - passed);
    }
}
```

There is no reason to use naming patterns when you can use annotations instead.

With the exception of toolshmiths, most programmers will have no need to define annotation types. All programmers should use the predefined annotations types that Java provides.

### Consistently use the override annotation

Buggy code, spot the error.

```java
public boolean equals(Bigram b) {
    return b.first == first && b.second == second;
}
```

With an `Override`, the program won't compile

```java
@Override public boolean equals(Bigram b) {
    return b.first == first && b.second == second;
}
```

Override will check overriding method at compile time. `equals` requires the parameter to be type `Object`. The fixed version

```java
@Override public boolean equals(Object o) {
    if (!(o instanceof Bigram))
        return false;

    Bigram b = (Bigram) o;
    return b.first == first && b.second == second;

}
```

Use the `Override` annotation on every method declaration that you believe to override a superclass declaration.

### Use marker interfaces to define types

A _marker interface_ is an interface that contains no method declarations but merely designates (or "marks") a class that implements the interface as having some property. An example of this is the `Serializable` interface.

Marker interfaces define a type that is implemented by instances of the marked class; marker annotations do not. **The existence of a marker interface type allows you to catch errors at compile time that you couldn't catch until runtime if you used a marker annotation.**

Another advantage of marker interfaces over marker annotations is that they can be targeted more precisely. If an annotation is declared with target `ElementType.TYPE`, it can be applied to _any_ class or interface.

The chief advantage of a marker annotation over marker interfaces is that they are part of the larger annotation facility. Marker annotations allow for consistency in annotation-based frameworks.

If you find yourself writing a marker annotation type whose target is `ElementType.TYPE`, take the time to figure out whether it is really should be an annotation type or whether a marker interface would be more appropriate.

## Lambdas and streams

### Prefer lambdas to anonymous classes

Obsolete

```java
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

Lambda expression as function object

```java
Collections.sort(words,
        (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

Omit the types of all lambda parameters unless their presence makes your program clearer.

Enum with function object fields and constant-specific behaviour

```java
public enum Operation {
    PLUS  ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override public String toString() { return symbol; }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

Lambdas lack names and documentation; if a computation isn't self-explanatory, or exceeds a few lines (more than three), don't put it in a lambda.


You should rarely, if eve, serialise a lambda.

Don't use anonymous classes for function objects unless you have to create instances of types that aren't functional interfaces.

### Prefer method references to lambdas

Where method references are shorter and clearer, use them; where they aren't, stick with lambdas.

```java
map.merge(key, 1, (count, incr) -> count + incr);
```

to

```java
map.merge(key, 1, Integer::sum);
```

All kinds of method references are summarised below

Method Ref Type   | Example                  | Lambda equivalent
------------------|--------------------------|------------------------------------------------------
Static            | `Integer::parseInt`      | `str -> Integer.parseInt(str)`
Bound             | `Instant.now()::isAfter` | `Instant then = Instant.now(); t -> then.isAfter(t)`
Unbound           | `String::toLowerCase`    | `str -> str.toLowerCase()`
Class Constructor | `TreeMap<K,V>::new`      | `() -> new TreeMap<K,V>`
Array Constructor | `int[]::new`             | `len -> new int[len]`

### Favour the use of standard functional interfaces

Unnecessary functional interface; use a standard one instead.

```java
@FunctionalInterface interface EldestEntryRemovalFunction<K,V>{
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
}
```

If one of the standard functional interfaces does the job, you should generally use it in preference to a purpose-built functional interface.

The six basic functional interfaces are summarised below

Interface           | Function Signature    | Example
--------------------|-----------------------|-----------------------
`UnaryOperator<T>`  | `T apply(T t)`        | `String::toLowerCase`
`BinaryOperator<T>` | `T apply(T t1, T t2)` | `BigInteger::add`
`Predicate<T>`      | `boolean test(T t)`   | `Collection::isEmpty`
`Function<T,R>`     | `R apply(T t)`        | `Arrays::asList`
`Supplier<T>`       | `T get()`             | `Instant::now`
`Consumer<T>`       | `void accept(T t)`    | `System.out.println`

Don't be tempted to use basic functional interfaces with boxed primitives instead of primitive functional interfaces (prefer primitive types to boxed primitives, performance will suffer otherwise).

**Always annotate your functional interfaces with the `@FunctionalInterface` annotation.** Its behaviour is similar to `@Override`, it tells the readers that the class was designed to enable lambdas; it won't compile unless it has exactly one abstract method; it prevents maintainers from accidentally adding abstract methods to the interface as it evolves.

### Use streams judiciously

```java
words.collect(groupingBy(word -> alphabetize(word)))
      .values().stream()
      .filter(group -> group.size() >= minGroupSize)
      .forEach(g -> System.out.println(g.size() + ": " + g));
```

Stream pipelines are evaluated `lazily`, evaluation doesn't start until the terminal operation is invoked.

Overusing streams makes programs hard to read and maintain.

In the absence of explicit types, careful naming of lambda parameters is essential to the readability of stream pipelines.

Using helper methods is even more important for readability in stream pipelines than in iterative code because pipelines lack explicit type information.

If you are not sure whether a task is better served by streams or iteration, try both and see which works better.

### Prefer side-effect-free functions in streams

The following code uses the streams API but not the paradigm, don't do this.

```
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}
```

Proper use of streams

```
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words
        .collect(groupingBy(String::toLowerCase, counting()));
}
```

**The `forEach` operation should be used only to report the result of a stream computation, not to perform the computation.**

Pipeline to get a top-ten list of words from a frequency table

```java
List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```

**It is customary and wise to statically import all members of `Collectors` because it makes stream pipelines more readable.**

### Prefer `Collection` to `Stream` as a return type

Streams do not make iteration obsolete, writing good code requires combining streams and iteration judiciously.

The `Collection` interface is a subtype of `Iterable` and has a `stream` method, so it provides both iteration and stream access. **`Collection` or an appropriate subtype is generally the best return type for a public, sequence-returning method.** But do not store large sequence in memory just to return it as a collection. If the sequence you're returning is large but can be represented concisely, consider implementing a special-purpose collection.

### Use caution when making streams parallel

Parallelising a pipeline is unlikely to increase its performance if the source is from `Stream.iterate`, or the intermediate operation `limit` is used.

Do not parallelise stream pipelines indiscriminately.

As a rule, performance gains from parallelism are best on streams over `ArrayList`, `HashMap`, `HashSet`, and `ConcurrentHashMap` instances; arrays; `int` ranges; and `long` ranges. What all these data structures have in common is that they can be cheaply split into subranges of any desired sizes, which makes them easy to divide work among parallel threads.

**Not only can parallelising a stream lead to poor performance, including liveness failures; it can lead to incorrect results and unpredictable behaviour.**

Under the right circumstances, it is possible to achieve near-linear speedup in the number of processor cores simply by adding a `parallel` call to a stream pipeline.

## Methods
