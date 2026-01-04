# 🚀 Modern Java Features (17+)

---

## 0️⃣ Prerequisites

Before diving into Modern Java Features, you should be familiar with:

- **OOP Fundamentals**: Classes, interfaces, inheritance
- **Generics**: Type parameters, bounded types
- **Lambda Expressions**: Functional interfaces, method references

---

## 1️⃣ Records (Java 16+)

### The Problem Records Solve

```java
// Traditional data class - lots of boilerplate!
public final class Person {
    private final String name;
    private final int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
    
    @Override
    public String toString() {
        return "Person[name=" + name + ", age=" + age + "]";
    }
}

// With records - one line!
public record Person(String name, int age) { }
```

### What Records Provide Automatically

```java
public record Person(String name, int age) { }

// You get automatically:
// 1. Private final fields
// 2. Public accessor methods (name(), age() - not getName(), getAge())
// 3. Public constructor
// 4. equals() based on all fields
// 5. hashCode() based on all fields
// 6. toString() with all fields

// Usage
Person person = new Person("Alice", 30);
String name = person.name();  // Note: name(), not getName()
int age = person.age();

System.out.println(person);  // Person[name=Alice, age=30]
```

### Customizing Records

```java
// Compact constructor - validation
public record Person(String name, int age) {
    public Person {  // No parameter list!
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
        name = name.trim();  // Can modify parameters before assignment
    }
}

// Additional constructor
public record Point(int x, int y) {
    public Point() {
        this(0, 0);  // Must delegate to canonical constructor
    }
}

// Additional methods
public record Rectangle(int width, int height) {
    public int area() {
        return width * height;
    }
    
    public boolean isSquare() {
        return width == height;
    }
}

// Implementing interfaces
public record Person(String name, int age) implements Comparable<Person> {
    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);
    }
}
```

### Record Limitations

```java
// Records CANNOT:
// 1. Extend other classes (implicitly extend Record)
// 2. Be extended (implicitly final)
// 3. Have mutable fields (implicitly final)
// 4. Have additional instance fields

// This is INVALID:
public record Person(String name, int age) {
    private String nickname;  // Compile error!
}

// But static fields are OK:
public record Person(String name, int age) {
    private static final int MAX_AGE = 150;
}
```

### Records with Collections

```java
// Be careful with mutable collections!
public record Team(String name, List<String> members) {
    public Team {
        // Defensive copy to ensure immutability
        members = List.copyOf(members);
    }
}

// Now the record is truly immutable
Team team = new Team("Dev", List.of("Alice", "Bob"));
team.members().add("Charlie");  // Throws UnsupportedOperationException
```

---

## 2️⃣ Sealed Classes (Java 17+)

### The Problem Sealed Classes Solve

```java
// Without sealed classes:
public abstract class Shape { }

// Anyone can extend Shape - you can't control the hierarchy
public class Circle extends Shape { }
public class Square extends Shape { }
public class Triangle extends Shape { }
public class WeirdShape extends Shape { }  // Unexpected subclass!

// Pattern matching can't be exhaustive:
double area = switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Square s -> s.side() * s.side();
    default -> throw new IllegalArgumentException();  // Required!
};
```

### Using Sealed Classes

```java
// Sealed class - explicitly lists permitted subclasses
public sealed class Shape permits Circle, Square, Triangle {
    // Common shape behavior
}

// Permitted subclasses must be final, sealed, or non-sealed

// Final - cannot be extended further
public final class Circle extends Shape {
    private final double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    public double radius() { return radius; }
}

// Sealed - can be extended by specific classes
public sealed class Rectangle extends Shape permits Square {
    private final double width, height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
}

// Non-sealed - open for extension (escape hatch)
public non-sealed class Triangle extends Shape {
    // Anyone can extend Triangle
}

// Final subclass of sealed class
public final class Square extends Rectangle {
    public Square(double side) {
        super(side, side);
    }
}
```

### Exhaustive Pattern Matching

```java
// With sealed classes, switch can be exhaustive (no default needed)
public double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square s -> s.side() * s.side();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> 0.5 * t.base() * t.height();
        // No default needed - compiler knows all cases are covered!
    };
}
```

### Sealed Interfaces

```java
// Sealed interfaces work too
public sealed interface Result<T> permits Success, Failure {
}

public record Success<T>(T value) implements Result<T> { }
public record Failure<T>(String error) implements Result<T> { }

// Usage
Result<User> result = findUser(id);
String message = switch (result) {
    case Success<User> s -> "Found: " + s.value().name();
    case Failure<User> f -> "Error: " + f.error();
};
```

---

## 3️⃣ Pattern Matching

### Pattern Matching for instanceof (Java 16+)

```java
// Old way
if (obj instanceof String) {
    String s = (String) obj;  // Explicit cast
    System.out.println(s.length());
}

// New way - pattern variable
if (obj instanceof String s) {
    System.out.println(s.length());  // s is already String
}

// With negation
if (!(obj instanceof String s)) {
    return;  // Early return
}
// s is in scope here!
System.out.println(s.length());

// In conditions
if (obj instanceof String s && s.length() > 5) {
    // s is String and has length > 5
}
```

### Pattern Matching for switch (Java 21+)

```java
// Type patterns in switch
String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "Integer: " + i;
        case Long l -> "Long: " + l;
        case Double d -> "Double: " + d;
        case String s -> "String: " + s;
        case null -> "null";
        default -> "Unknown: " + obj.getClass();
    };
}

// Guarded patterns (with when clause)
String categorize(Object obj) {
    return switch (obj) {
        case Integer i when i < 0 -> "Negative integer";
        case Integer i when i == 0 -> "Zero";
        case Integer i -> "Positive integer";
        case String s when s.isEmpty() -> "Empty string";
        case String s -> "String: " + s;
        default -> "Other";
    };
}

// Record patterns (Java 21+)
record Point(int x, int y) { }
record Circle(Point center, int radius) { }

String describe(Object obj) {
    return switch (obj) {
        case Point(int x, int y) -> "Point at (" + x + ", " + y + ")";
        case Circle(Point(int x, int y), int r) -> 
            "Circle at (" + x + ", " + y + ") with radius " + r;
        default -> "Unknown";
    };
}
```

### Exhaustiveness in Pattern Matching

```java
sealed interface Animal permits Dog, Cat { }
record Dog(String name) implements Animal { }
record Cat(String name) implements Animal { }

// Exhaustive - no default needed
String sound(Animal animal) {
    return switch (animal) {
        case Dog d -> "Woof!";
        case Cat c -> "Meow!";
    };
}

// Record pattern deconstruction
String describe(Animal animal) {
    return switch (animal) {
        case Dog(String name) -> name + " says woof!";
        case Cat(String name) -> name + " says meow!";
    };
}
```

---

## 4️⃣ Text Blocks (Java 15+)

### The Problem Text Blocks Solve

```java
// Old way - messy string concatenation
String json = "{\n" +
    "  \"name\": \"Alice\",\n" +
    "  \"age\": 30,\n" +
    "  \"address\": {\n" +
    "    \"city\": \"NYC\"\n" +
    "  }\n" +
    "}";

// With text blocks - clean and readable
String json = """
    {
      "name": "Alice",
      "age": 30,
      "address": {
        "city": "NYC"
      }
    }
    """;
```

### Text Block Features

```java
// Incidental whitespace is removed
String html = """
    <html>
        <body>
            <p>Hello</p>
        </body>
    </html>
    """;
// Leading spaces before <html> are removed (incidental)
// Indentation within is preserved (essential)

// Trailing whitespace is stripped by default
// Use \s to preserve trailing space
String table = """
    Name     \s
    Alice    \s
    Bob      \s
    """;

// Line continuation with \
String longLine = """
    This is a very long line that \
    continues on the next line \
    but will be rendered as one line.
    """;

// Escape sequences work
String withQuotes = """
    He said "Hello"
    """;

// String formatting with formatted()
String template = """
    {
      "name": "%s",
      "age": %d
    }
    """.formatted(name, age);
```

---

## 5️⃣ Local Variable Type Inference (var)

### Using var

```java
// var infers the type from the right-hand side
var name = "Alice";           // String
var age = 30;                 // int
var list = new ArrayList<String>();  // ArrayList<String>
var map = Map.of("a", 1);     // Map<String, Integer>

// Useful with complex generic types
// Without var:
Map<String, List<Map<String, Object>>> complex = new HashMap<>();

// With var:
var complex = new HashMap<String, List<Map<String, Object>>>();

// In for loops
for (var entry : map.entrySet()) {
    var key = entry.getKey();
    var value = entry.getValue();
}

// With try-with-resources
try (var reader = new BufferedReader(new FileReader(path))) {
    var line = reader.readLine();
}
```

### When NOT to Use var

```java
// DON'T: When type isn't obvious
var result = process();  // What type is result?

// DO: When type is clear
var users = new ArrayList<User>();  // Obviously ArrayList<User>
var name = user.getName();          // Obviously String

// DON'T: With diamond operator alone
var list = new ArrayList<>();  // ArrayList<Object>!

// DO: Specify type parameter
var list = new ArrayList<String>();

// DON'T: When it reduces readability
var x = getX();  // What is x?

// DO: Use explicit type
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
```

### var Limitations

```java
// var CANNOT be used for:

// 1. Fields
class MyClass {
    var field = "hello";  // Compile error!
}

// 2. Method parameters
void method(var param) { }  // Compile error!

// 3. Method return types
var method() { }  // Compile error!

// 4. Without initializer
var x;  // Compile error!
x = 5;

// 5. With null
var x = null;  // Compile error! Type cannot be inferred

// 6. Array initializers
var arr = {1, 2, 3};  // Compile error!
var arr = new int[]{1, 2, 3};  // OK
```

---

## 6️⃣ Other Modern Features

### Switch Expressions (Java 14+)

```java
// Old switch statement
String result;
switch (day) {
    case MONDAY:
    case FRIDAY:
        result = "Work";
        break;
    case SATURDAY:
    case SUNDAY:
        result = "Rest";
        break;
    default:
        result = "Unknown";
}

// New switch expression
String result = switch (day) {
    case MONDAY, FRIDAY -> "Work";
    case SATURDAY, SUNDAY -> "Rest";
    default -> "Unknown";
};

// With blocks
String result = switch (day) {
    case MONDAY, FRIDAY -> {
        System.out.println("Starting work...");
        yield "Work";  // yield instead of return
    }
    case SATURDAY, SUNDAY -> "Rest";
    default -> "Unknown";
};
```

### Helpful NullPointerExceptions (Java 14+)

```java
// Old NPE message:
// Exception in thread "main" java.lang.NullPointerException

// New helpful NPE message:
// Exception in thread "main" java.lang.NullPointerException: 
//     Cannot invoke "String.length()" because "str" is null

// For chained calls:
user.getAddress().getCity().toUpperCase();
// Exception: Cannot invoke "String.toUpperCase()" because 
//     the return value of "Address.getCity()" is null
```

### String Methods

```java
// Java 11+
"  hello  ".strip();       // "hello" (Unicode-aware trim)
"  hello  ".stripLeading(); // "hello  "
"  hello  ".stripTrailing(); // "  hello"
"".isBlank();              // true (empty or whitespace only)
"hello\nworld".lines();    // Stream<String>
"ab".repeat(3);            // "ababab"

// Java 12+
"hello".indent(4);         // "    hello\n"
"hello".transform(s -> s.toUpperCase());  // "HELLO"

// Java 15+
"hello".formatted();       // For text blocks
```

### Collection Factory Methods (Java 9+)

```java
// Immutable collections
List<String> list = List.of("a", "b", "c");
Set<String> set = Set.of("a", "b", "c");
Map<String, Integer> map = Map.of("a", 1, "b", 2);

// For larger maps
Map<String, Integer> largeMap = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3)
);

// Copy to immutable
List<String> immutable = List.copyOf(mutableList);
```

---

## 7️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is a record in Java?**

A: A record is a special kind of class that's a transparent carrier for immutable data. It automatically provides:
- Private final fields for each component
- Public accessor methods (name(), not getName())
- Constructor
- equals(), hashCode(), and toString()

Records are implicitly final and cannot extend other classes. They're ideal for DTOs, value objects, and any class that's just "data."

```java
public record Person(String name, int age) { }
```

**Q: What is the difference between `var` and explicit type declaration?**

A: `var` is local variable type inference - the compiler infers the type from the initializer. It's purely a compile-time feature; the bytecode is identical.

Use `var` when the type is obvious from the right side. Don't use it when it reduces readability.

```java
var list = new ArrayList<String>();  // Clear - use var
var result = process();              // Unclear - use explicit type
```

### L5 (Mid-Level) Questions

**Q: What are sealed classes and when would you use them?**

A: Sealed classes restrict which classes can extend them. You explicitly list permitted subclasses.

Use cases:
1. **Domain modeling**: When you have a fixed set of subtypes (Payment: CreditCard, DebitCard, PayPal)
2. **Exhaustive pattern matching**: Compiler can verify all cases are handled
3. **API design**: Prevent unexpected extensions of your classes

Permitted subclasses must be `final`, `sealed`, or `non-sealed`.

**Q: Explain pattern matching in switch expressions.**

A: Pattern matching in switch allows you to:
1. Match on type: `case Integer i ->`
2. Deconstruct records: `case Point(int x, int y) ->`
3. Add guards: `case Integer i when i > 0 ->`
4. Handle null: `case null ->`

With sealed classes, the switch can be exhaustive (no default needed).

```java
return switch (shape) {
    case Circle(Point center, int r) -> "Circle at " + center;
    case Rectangle r when r.isSquare() -> "Square";
    case Rectangle r -> "Rectangle";
};
```

### L6 (Senior) Questions

**Q: How do modern Java features improve API design?**

A: Several features work together:

1. **Records** for immutable DTOs - less boilerplate, guaranteed immutability
2. **Sealed classes** for closed hierarchies - exhaustive handling, clear API boundaries
3. **Pattern matching** for type-safe processing - no casts, compiler-verified exhaustiveness
4. **Text blocks** for embedded DSLs - SQL, JSON, HTML templates

Example: A result type API:
```java
sealed interface Result<T> permits Success, Failure { }
record Success<T>(T value) implements Result<T> { }
record Failure<T>(String error) implements Result<T> { }

// Client code is exhaustive and type-safe
String message = switch (result) {
    case Success(var value) -> "Got: " + value;
    case Failure(var error) -> "Error: " + error;
};
```

---

## 8️⃣ One Clean Mental Summary

Modern Java (17+) eliminates boilerplate and adds expressiveness. **Records** replace data classes with one-line declarations - automatic constructors, accessors, equals, hashCode, toString. **Sealed classes** create closed hierarchies where you control all subclasses, enabling exhaustive pattern matching. **Pattern matching** for instanceof and switch eliminates casts and enables destructuring records directly. **Text blocks** make multi-line strings readable. **var** reduces verbosity when types are obvious. **Switch expressions** return values and support multiple case labels. Together, these features make Java code more concise, safer, and more expressive while maintaining backward compatibility. The key insight: use these features to reduce boilerplate and let the compiler catch more errors at compile time.

