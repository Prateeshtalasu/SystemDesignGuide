# 🔬 Advanced Java Features

---

## 0️⃣ Prerequisites

Before diving into Advanced Java Features, you need to understand:

- **OOP Fundamentals**: Classes, interfaces, inheritance (covered in `01-oop-fundamentals.md`)
- **Collections Framework**: Lists, Maps, Sets (covered in `03-collections-deep-dive.md`)
- **Streams API Basics**: map, filter, collect (covered in `04-streams-api.md`)

---

## 1️⃣ Generics

### The Problem Generics Solve

```java
// Without generics (Java 1.4 and earlier)
List list = new ArrayList();
list.add("Hello");
list.add(123);  // Compiles! No type safety

String s = (String) list.get(0);  // Must cast
String s2 = (String) list.get(1); // ClassCastException at runtime!

// With generics
List<String> list = new ArrayList<>();
list.add("Hello");
list.add(123);  // Compile error! Type safety

String s = list.get(0);  // No cast needed
```

### Generic Classes and Methods

```java
// Generic class
public class Box<T> {
    private T content;
    
    public void set(T content) {
        this.content = content;
    }
    
    public T get() {
        return content;
    }
}

// Usage
Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String s = stringBox.get();

Box<Integer> intBox = new Box<>();
intBox.set(42);
Integer i = intBox.get();

// Generic method
public class Util {
    public static <T> T getFirst(List<T> list) {
        return list.isEmpty() ? null : list.get(0);
    }
    
    // Multiple type parameters
    public static <K, V> Map<K, V> createMap(K key, V value) {
        Map<K, V> map = new HashMap<>();
        map.put(key, value);
        return map;
    }
}

// Usage - type inference
String first = Util.getFirst(List.of("a", "b", "c"));
Map<String, Integer> map = Util.createMap("age", 25);
```

### Bounded Type Parameters

```java
// Upper bound: T must be Number or subclass
public class NumberBox<T extends Number> {
    private T value;
    
    public double getDoubleValue() {
        return value.doubleValue();  // Can call Number methods
    }
}

NumberBox<Integer> intBox = new NumberBox<>();  // OK
NumberBox<Double> doubleBox = new NumberBox<>();  // OK
NumberBox<String> stringBox = new NumberBox<>();  // Compile error!

// Multiple bounds: T must implement multiple interfaces
public class Processor<T extends Comparable<T> & Serializable> {
    public int compare(T a, T b) {
        return a.compareTo(b);
    }
}

// Lower bound in wildcards (see below)
```

### Wildcards

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WILDCARD TYPES                                        │
│                                                                          │
│   ? extends T (Upper Bounded)                                           │
│   ─────────────────────────────                                         │
│   "Something that IS-A T or subtype of T"                              │
│   Can READ as T, cannot WRITE (except null)                            │
│                                                                          │
│   List<? extends Number>                                                │
│   - Can hold List<Integer>, List<Double>, List<Number>                 │
│   - Can read as Number                                                  │
│   - Cannot add (don't know exact type)                                 │
│                                                                          │
│   ? super T (Lower Bounded)                                             │
│   ─────────────────────────                                             │
│   "Something that IS-A T or supertype of T"                            │
│   Can WRITE T, cannot READ (except as Object)                          │
│                                                                          │
│   List<? super Integer>                                                 │
│   - Can hold List<Integer>, List<Number>, List<Object>                 │
│   - Can add Integer                                                     │
│   - Can only read as Object                                            │
│                                                                          │
│   PECS: Producer Extends, Consumer Super                               │
│   ─────────────────────────────────────────                            │
│   - If you READ from a structure, use extends (it produces values)     │
│   - If you WRITE to a structure, use super (it consumes values)        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

```java
// PECS in action
public class Collections {
    
    // Source produces values - use extends
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (T item : src) {      // Read from src (extends)
            dest.add(item);        // Write to dest (super)
        }
    }
}

// Example
List<Number> numbers = new ArrayList<>();
List<Integer> integers = List.of(1, 2, 3);

Collections.copy(numbers, integers);  // Works!
// numbers is List<? super Integer> - can write Integer
// integers is List<? extends Integer> - can read as Integer
```

### Type Erasure

```java
// What you write:
public class Box<T> {
    private T value;
    public T get() { return value; }
}

// What the compiler generates (type erasure):
public class Box {
    private Object value;
    public Object get() { return value; }
}

// Implications:
// 1. Cannot use instanceof with generic types
if (list instanceof List<String>) { }  // Compile error!
if (list instanceof List<?>) { }       // OK (unbounded wildcard)

// 2. Cannot create arrays of generic types
T[] array = new T[10];  // Compile error!

// 3. Cannot create instances of type parameters
T instance = new T();  // Compile error!

// Workaround: Pass class token
public <T> T create(Class<T> clazz) throws Exception {
    return clazz.getDeclaredConstructor().newInstance();
}
```

---

## 2️⃣ Lambda Expressions & Method References

### Lambda Syntax

```java
// Full syntax
(parameters) -> { statements; return value; }

// Simplified forms
(a, b) -> a + b                    // Expression body (implicit return)
a -> a * 2                         // Single parameter (no parentheses)
() -> System.out.println("Hello")  // No parameters

// Examples with functional interfaces
Runnable r = () -> System.out.println("Running");

Comparator<String> comp = (s1, s2) -> s1.length() - s2.length();

Function<String, Integer> length = s -> s.length();

BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

Predicate<String> isEmpty = s -> s.isEmpty();

Consumer<String> printer = s -> System.out.println(s);

Supplier<Double> random = () -> Math.random();
```

### Method References

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    METHOD REFERENCE TYPES                                │
│                                                                          │
│   Type                    Syntax              Lambda Equivalent         │
│   ────                    ──────              ─────────────────         │
│   Static method           Class::method       x -> Class.method(x)      │
│   Instance method         obj::method         x -> obj.method(x)        │
│   Instance method         Class::method       (obj, x) -> obj.method(x) │
│   Constructor             Class::new          x -> new Class(x)         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

```java
// Static method reference
Function<String, Integer> parse = Integer::parseInt;
// Equivalent: s -> Integer.parseInt(s)

// Instance method on specific object
String prefix = "Hello, ";
Function<String, String> greeter = prefix::concat;
// Equivalent: s -> prefix.concat(s)

// Instance method on arbitrary object
Function<String, String> upper = String::toUpperCase;
// Equivalent: s -> s.toUpperCase()

// Constructor reference
Supplier<ArrayList<String>> listFactory = ArrayList::new;
// Equivalent: () -> new ArrayList<>()

Function<Integer, ArrayList<String>> sizedList = ArrayList::new;
// Equivalent: size -> new ArrayList<>(size)

// Usage in streams
List<String> names = List.of("alice", "bob", "charlie");

names.stream()
    .map(String::toUpperCase)          // Method reference
    .forEach(System.out::println);     // Method reference
```

### Functional Interfaces

```java
// Built-in functional interfaces (java.util.function)

// Takes nothing, returns T
Supplier<String> supplier = () -> "Hello";

// Takes T, returns nothing
Consumer<String> consumer = s -> System.out.println(s);

// Takes T, returns boolean
Predicate<String> predicate = s -> s.length() > 5;

// Takes T, returns R
Function<String, Integer> function = s -> s.length();

// Takes T and U, returns R
BiFunction<String, String, Integer> biFunction = (a, b) -> a.length() + b.length();

// Takes T, returns T
UnaryOperator<String> unary = s -> s.toUpperCase();

// Takes T and T, returns T
BinaryOperator<Integer> binary = (a, b) -> a + b;

// Custom functional interface
@FunctionalInterface
public interface TriFunction<A, B, C, R> {
    R apply(A a, B b, C c);
}

TriFunction<Integer, Integer, Integer, Integer> sum = (a, b, c) -> a + b + c;
```

---

## 3️⃣ Optional Class

### Why Optional Exists

```java
// The billion-dollar mistake: null
public User findUser(String id) {
    // Might return null - caller must check!
    return userRepository.findById(id);
}

// Caller forgets to check
User user = findUser("123");
user.getName();  // NullPointerException!

// With Optional - explicit about absence
public Optional<User> findUser(String id) {
    return userRepository.findById(id);
}

// Caller is forced to handle absence
Optional<User> maybeUser = findUser("123");
// Can't call .getName() directly on Optional
```

### Creating Optionals

```java
// Create with value (throws if null)
Optional<String> opt1 = Optional.of("Hello");

// Create with nullable value
Optional<String> opt2 = Optional.ofNullable(possiblyNull);

// Create empty
Optional<String> opt3 = Optional.empty();
```

### Using Optionals

```java
Optional<User> maybeUser = findUser("123");

// WRONG: Defeats the purpose
if (maybeUser.isPresent()) {
    User user = maybeUser.get();
    // ...
}

// RIGHT: Use functional methods

// 1. Provide default value
String name = maybeUser.map(User::getName).orElse("Unknown");

// 2. Provide default via supplier (lazy)
User user = maybeUser.orElseGet(() -> createDefaultUser());

// 3. Throw if absent
User user = maybeUser.orElseThrow(() -> 
    new UserNotFoundException("User not found"));

// 4. Execute action if present
maybeUser.ifPresent(u -> sendWelcomeEmail(u));

// 5. Execute action if present, else action if absent (Java 9+)
maybeUser.ifPresentOrElse(
    u -> System.out.println("Found: " + u.getName()),
    () -> System.out.println("Not found")
);

// 6. Filter
Optional<User> admin = maybeUser.filter(User::isAdmin);

// 7. Transform
Optional<String> email = maybeUser.map(User::getEmail);

// 8. FlatMap for nested Optionals
Optional<String> street = maybeUser
    .flatMap(User::getAddress)    // Returns Optional<Address>
    .flatMap(Address::getStreet); // Returns Optional<String>

// 9. Stream (Java 9+)
List<String> names = users.stream()
    .map(this::findUser)
    .flatMap(Optional::stream)  // Filter out empty Optionals
    .map(User::getName)
    .toList();
```

### Optional Anti-Patterns

```java
// DON'T: Use Optional for fields
public class User {
    private Optional<String> middleName;  // BAD
}

// DON'T: Use Optional for method parameters
public void process(Optional<String> input) { }  // BAD

// DON'T: Use Optional in collections
List<Optional<String>> list;  // BAD

// DON'T: Use isPresent() + get()
if (opt.isPresent()) {
    return opt.get();  // BAD - use orElse/map instead
}

// DO: Use Optional for return types when absence is valid
public Optional<User> findById(String id) { }  // GOOD

// DO: Use orElse/map/flatMap
String name = opt.map(User::getName).orElse("Unknown");  // GOOD
```

---

## 4️⃣ Reflection API

### What Reflection Enables

```java
// Without reflection: Must know types at compile time
User user = new User();
user.setName("Alice");

// With reflection: Work with types at runtime
Class<?> clazz = Class.forName("com.example.User");
Object user = clazz.getDeclaredConstructor().newInstance();
Method setter = clazz.getMethod("setName", String.class);
setter.invoke(user, "Alice");
```

### Basic Reflection Operations

```java
// Get Class object
Class<User> clazz1 = User.class;
Class<?> clazz2 = user.getClass();
Class<?> clazz3 = Class.forName("com.example.User");

// Inspect class
System.out.println("Name: " + clazz1.getName());
System.out.println("Simple name: " + clazz1.getSimpleName());
System.out.println("Package: " + clazz1.getPackage().getName());
System.out.println("Superclass: " + clazz1.getSuperclass());
System.out.println("Interfaces: " + Arrays.toString(clazz1.getInterfaces()));
System.out.println("Modifiers: " + Modifier.toString(clazz1.getModifiers()));

// Get fields
Field[] fields = clazz1.getDeclaredFields();
for (Field field : fields) {
    System.out.println(field.getName() + ": " + field.getType().getSimpleName());
}

// Get methods
Method[] methods = clazz1.getDeclaredMethods();
for (Method method : methods) {
    System.out.println(method.getName() + 
        Arrays.toString(method.getParameterTypes()));
}

// Get constructors
Constructor<?>[] constructors = clazz1.getDeclaredConstructors();
```

### Working with Fields

```java
public class User {
    private String name;
    private int age;
}

User user = new User();
Class<?> clazz = user.getClass();

// Access private field
Field nameField = clazz.getDeclaredField("name");
nameField.setAccessible(true);  // Bypass private access

// Set value
nameField.set(user, "Alice");

// Get value
String name = (String) nameField.get(user);
System.out.println(name);  // "Alice"
```

### Working with Methods

```java
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
    
    private int multiply(int a, int b) {
        return a * b;
    }
}

Calculator calc = new Calculator();
Class<?> clazz = calc.getClass();

// Invoke public method
Method addMethod = clazz.getMethod("add", int.class, int.class);
int result = (int) addMethod.invoke(calc, 5, 3);
System.out.println(result);  // 8

// Invoke private method
Method multiplyMethod = clazz.getDeclaredMethod("multiply", int.class, int.class);
multiplyMethod.setAccessible(true);
int result2 = (int) multiplyMethod.invoke(calc, 5, 3);
System.out.println(result2);  // 15
```

### Creating Instances

```java
// Using default constructor
Class<User> clazz = User.class;
User user = clazz.getDeclaredConstructor().newInstance();

// Using parameterized constructor
Constructor<User> constructor = clazz.getDeclaredConstructor(String.class, int.class);
User user2 = constructor.newInstance("Alice", 25);
```

### Reflection Use Cases

```java
// 1. Frameworks (Spring, Hibernate) - create beans, inject dependencies
// 2. Serialization/Deserialization (Jackson, Gson)
// 3. Testing frameworks (JUnit, Mockito)
// 4. ORM mapping
// 5. Dependency injection
// 6. Plugin systems

// Example: Simple dependency injection
public class DIContainer {
    private Map<Class<?>, Object> beans = new HashMap<>();
    
    public <T> T getBean(Class<T> clazz) throws Exception {
        if (beans.containsKey(clazz)) {
            return clazz.cast(beans.get(clazz));
        }
        
        // Create instance
        T instance = clazz.getDeclaredConstructor().newInstance();
        
        // Inject dependencies
        for (Field field : clazz.getDeclaredFields()) {
            if (field.isAnnotationPresent(Inject.class)) {
                field.setAccessible(true);
                Object dependency = getBean(field.getType());
                field.set(instance, dependency);
            }
        }
        
        beans.put(clazz, instance);
        return instance;
    }
}
```

---

## 5️⃣ Annotations

### Built-in Annotations

```java
// Compiler annotations
@Override           // Method overrides superclass method
@Deprecated         // Element is deprecated
@SuppressWarnings   // Suppress compiler warnings
@FunctionalInterface // Interface is functional (one abstract method)

// Runtime annotations
@Retention(RetentionPolicy.RUNTIME)  // Available at runtime
@Target(ElementType.METHOD)          // Can only be on methods
```

### Creating Custom Annotations

```java
// Define annotation
@Retention(RetentionPolicy.RUNTIME)  // Available at runtime
@Target({ElementType.METHOD, ElementType.TYPE})  // For methods and classes
public @interface Cacheable {
    String value() default "";           // Cache name
    int ttlSeconds() default 300;        // Time to live
    boolean enabled() default true;      // Enable/disable
}

// Use annotation
@Cacheable(value = "users", ttlSeconds = 600)
public User getUser(String id) {
    return userRepository.findById(id);
}

// Process annotation at runtime
public class CacheProcessor {
    public Object process(Method method, Object[] args) throws Exception {
        Cacheable cacheable = method.getAnnotation(Cacheable.class);
        
        if (cacheable != null && cacheable.enabled()) {
            String cacheName = cacheable.value();
            int ttl = cacheable.ttlSeconds();
            
            // Check cache
            String key = generateKey(method, args);
            Object cached = cache.get(cacheName, key);
            if (cached != null) {
                return cached;
            }
            
            // Execute and cache
            Object result = method.invoke(target, args);
            cache.put(cacheName, key, result, ttl);
            return result;
        }
        
        return method.invoke(target, args);
    }
}
```

### Meta-Annotations

```java
// @Retention - When annotation is available
@Retention(RetentionPolicy.SOURCE)   // Compile time only (like @Override)
@Retention(RetentionPolicy.CLASS)    // In .class file, not at runtime
@Retention(RetentionPolicy.RUNTIME)  // Available at runtime via reflection

// @Target - Where annotation can be used
@Target(ElementType.TYPE)            // Class, interface, enum
@Target(ElementType.FIELD)           // Field
@Target(ElementType.METHOD)          // Method
@Target(ElementType.PARAMETER)       // Method parameter
@Target(ElementType.CONSTRUCTOR)     // Constructor
@Target(ElementType.LOCAL_VARIABLE)  // Local variable
@Target(ElementType.ANNOTATION_TYPE) // Another annotation
@Target(ElementType.PACKAGE)         // Package
@Target(ElementType.TYPE_PARAMETER)  // Type parameter (generics)
@Target(ElementType.TYPE_USE)        // Any type use

// @Documented - Include in Javadoc
@Documented

// @Inherited - Subclasses inherit annotation
@Inherited

// @Repeatable - Can be applied multiple times
@Repeatable(Schedules.class)
public @interface Schedule {
    String cron();
}

@Schedule(cron = "0 0 * * *")
@Schedule(cron = "0 12 * * *")
public void runTask() { }
```

### Annotation Processing (Compile-time)

```java
// Annotation processor
@SupportedAnnotationTypes("com.example.Builder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {
    
    @Override
    public boolean process(Set<? extends TypeElement> annotations, 
                          RoundEnvironment roundEnv) {
        
        for (Element element : roundEnv.getElementsAnnotatedWith(Builder.class)) {
            // Generate builder class
            TypeElement typeElement = (TypeElement) element;
            String className = typeElement.getSimpleName() + "Builder";
            
            // Use Filer to create new source file
            JavaFileObject file = processingEnv.getFiler()
                .createSourceFile(className);
            
            try (PrintWriter writer = new PrintWriter(file.openWriter())) {
                // Generate builder code
                writer.println("public class " + className + " {");
                // ... generate fields, setters, build method
                writer.println("}");
            }
        }
        
        return true;
    }
}
```

---

## 6️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between `List<Object>` and `List<?>`?**

A: `List<Object>` is a list that holds Objects. You can add any object to it.

`List<?>` is a list of unknown type. You can only read from it (as Object) but cannot add anything (except null).

```java
List<Object> objectList = new ArrayList<>();
objectList.add("String");  // OK
objectList.add(123);       // OK

List<?> unknownList = new ArrayList<String>();
unknownList.add("String"); // Compile error!
Object o = unknownList.get(0);  // OK, read as Object
```

**Q: What is a lambda expression?**

A: A lambda is a concise way to represent an anonymous function (a method without a name). It can be passed as an argument or stored in a variable.

Syntax: `(parameters) -> expression` or `(parameters) -> { statements }`

Lambdas work with functional interfaces (interfaces with one abstract method).

Example: `(a, b) -> a + b` is a lambda that takes two parameters and returns their sum.

### L5 (Mid-Level) Questions

**Q: Explain PECS (Producer Extends, Consumer Super).**

A: PECS is a guideline for using wildcards in generics:

- **Producer Extends**: If you only READ from a collection, use `? extends T`. The collection "produces" values.
- **Consumer Super**: If you only WRITE to a collection, use `? super T`. The collection "consumes" values.

```java
// Copy from src (producer) to dest (consumer)
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T item : src) {  // Read from extends
        dest.add(item);    // Write to super
    }
}
```

**Q: When should you use Optional and when should you avoid it?**

A: **Use Optional**:
- Return type when a method might not return a value
- When absence is a valid, expected case

**Avoid Optional**:
- As field types (use null instead)
- As method parameters (use overloading)
- In collections (use empty collection)
- When you'll just call `isPresent()` + `get()` (defeats the purpose)

The goal is to make absence explicit in the API and force callers to handle it.

### L6 (Senior) Questions

**Q: How does type erasure affect generics at runtime?**

A: Type erasure removes generic type information at compile time. The compiler:

1. Replaces type parameters with their bounds (or Object if unbounded)
2. Inserts casts where necessary
3. Generates bridge methods for polymorphism

Implications:
- Cannot use `instanceof` with generic types: `x instanceof List<String>` fails
- Cannot create generic arrays: `new T[10]` fails
- Cannot call `new T()`: no type info at runtime
- All `List<String>`, `List<Integer>` become just `List` at runtime

Workarounds include passing `Class<T>` tokens or using TypeReference patterns.

**Q: How would you implement a custom annotation processor?**

A: Steps:
1. Create annotation with `@Retention(SOURCE)` or `CLASS`
2. Extend `AbstractProcessor`
3. Override `process()` method
4. Use `processingEnv.getFiler()` to generate code
5. Register processor in `META-INF/services/javax.annotation.processing.Processor`

Use cases: Code generation (Lombok, MapStruct), validation, documentation generation.

The processor runs at compile time, can read annotated elements, and generate new source files that get compiled in the same compilation unit.

---

## 7️⃣ One Clean Mental Summary

**Generics** provide compile-time type safety and eliminate casts. Use bounded types (`<T extends Number>`) to restrict types, and wildcards (`? extends`, `? super`) for flexibility following PECS. Type erasure means generic info is gone at runtime. **Lambdas** are concise anonymous functions for functional interfaces - use method references (`Class::method`) when possible for readability. **Optional** makes absence explicit - use it for return types, not fields or parameters, and prefer `map`/`orElse` over `isPresent`/`get`. **Reflection** enables runtime type inspection and manipulation - powerful but slow and bypasses compile-time checks. Use it for frameworks, not application code. **Annotations** are metadata - create custom ones with `@interface`, use `@Retention(RUNTIME)` for runtime processing, or `SOURCE` for compile-time annotation processors that generate code.

