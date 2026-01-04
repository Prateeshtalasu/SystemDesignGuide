# ✨ Java Best Practices

---

## 0️⃣ Prerequisites

Before diving into Java Best Practices, you should be familiar with:

- **OOP Fundamentals**: Classes, interfaces, inheritance (covered in `01-oop-fundamentals.md`)
- **SOLID Principles**: Design principles (covered in `02-solid-principles.md`)
- **Collections Framework**: Lists, Maps, Sets (covered in `03-collections-deep-dive.md`)

---

## 1️⃣ Effective Java Principles

### Item 1: Consider Static Factory Methods Instead of Constructors

```java
// Constructor
public class User {
    public User(String name, String email) { }
}
User user = new User("Alice", "alice@example.com");

// Static factory method - more expressive
public class User {
    private User(String name, String email) { }
    
    public static User createWithEmail(String name, String email) {
        return new User(name, email);
    }
    
    public static User createGuest() {
        return new User("Guest", null);
    }
    
    public static User fromJson(String json) {
        // Parse and create
    }
}
User user = User.createWithEmail("Alice", "alice@example.com");
User guest = User.createGuest();
```

**Advantages**:
- Descriptive names (`createGuest` vs `new User()`)
- Can return cached instances
- Can return subtypes
- Reduce verbosity with type inference

### Item 2: Consider a Builder When Faced with Many Constructor Parameters

```java
// Telescoping constructor anti-pattern
public class Pizza {
    public Pizza(int size) { }
    public Pizza(int size, boolean cheese) { }
    public Pizza(int size, boolean cheese, boolean pepperoni) { }
    public Pizza(int size, boolean cheese, boolean pepperoni, boolean mushrooms) { }
    // Gets out of hand quickly!
}

// Builder pattern
public class Pizza {
    private final int size;
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean mushrooms;
    
    private Pizza(Builder builder) {
        this.size = builder.size;
        this.cheese = builder.cheese;
        this.pepperoni = builder.pepperoni;
        this.mushrooms = builder.mushrooms;
    }
    
    public static class Builder {
        // Required
        private final int size;
        
        // Optional with defaults
        private boolean cheese = false;
        private boolean pepperoni = false;
        private boolean mushrooms = false;
        
        public Builder(int size) {
            this.size = size;
        }
        
        public Builder cheese() {
            this.cheese = true;
            return this;
        }
        
        public Builder pepperoni() {
            this.pepperoni = true;
            return this;
        }
        
        public Builder mushrooms() {
            this.mushrooms = true;
            return this;
        }
        
        public Pizza build() {
            return new Pizza(this);
        }
    }
}

// Usage - readable and flexible
Pizza pizza = new Pizza.Builder(12)
    .cheese()
    .pepperoni()
    .build();
```

### Item 3: Enforce Singleton with Private Constructor or Enum

```java
// Enum singleton (preferred)
public enum Database {
    INSTANCE;
    
    private Connection connection;
    
    public Connection getConnection() {
        if (connection == null) {
            connection = createConnection();
        }
        return connection;
    }
}

// Usage
Database.INSTANCE.getConnection();
```

### Item 4: Enforce Noninstantiability with Private Constructor

```java
// Utility class - should not be instantiated
public final class StringUtils {
    
    private StringUtils() {
        throw new AssertionError("Cannot instantiate utility class");
    }
    
    public static boolean isEmpty(String s) {
        return s == null || s.isEmpty();
    }
    
    public static String capitalize(String s) {
        if (isEmpty(s)) return s;
        return Character.toUpperCase(s.charAt(0)) + s.substring(1);
    }
}
```

### Item 5: Prefer Dependency Injection to Hardwiring Resources

```java
// BAD: Hardwired dependency
public class SpellChecker {
    private final Dictionary dictionary = new EnglishDictionary();  // Hardwired!
    
    public boolean isValid(String word) {
        return dictionary.contains(word);
    }
}

// GOOD: Dependency injection
public class SpellChecker {
    private final Dictionary dictionary;
    
    public SpellChecker(Dictionary dictionary) {  // Injected
        this.dictionary = Objects.requireNonNull(dictionary);
    }
    
    public boolean isValid(String word) {
        return dictionary.contains(word);
    }
}

// Can inject different implementations
SpellChecker english = new SpellChecker(new EnglishDictionary());
SpellChecker spanish = new SpellChecker(new SpanishDictionary());
SpellChecker test = new SpellChecker(mockDictionary);  // For testing
```

---

## 2️⃣ Code Smells and Refactoring

### Long Method

```java
// BAD: Method does too much
public void processOrder(Order order) {
    // Validate order (20 lines)
    // Calculate totals (15 lines)
    // Apply discounts (25 lines)
    // Update inventory (10 lines)
    // Send notifications (20 lines)
    // Generate invoice (15 lines)
}

// GOOD: Extract methods
public void processOrder(Order order) {
    validateOrder(order);
    calculateTotals(order);
    applyDiscounts(order);
    updateInventory(order);
    sendNotifications(order);
    generateInvoice(order);
}

private void validateOrder(Order order) { /* ... */ }
private void calculateTotals(Order order) { /* ... */ }
// etc.
```

### Large Class

```java
// BAD: God class
public class UserManager {
    public void createUser() { }
    public void deleteUser() { }
    public void sendEmail() { }
    public void generateReport() { }
    public void processPayment() { }
    public void updateInventory() { }
    // 50+ more methods...
}

// GOOD: Single responsibility
public class UserService {
    public void createUser() { }
    public void deleteUser() { }
}

public class EmailService {
    public void sendEmail() { }
}

public class ReportService {
    public void generateReport() { }
}
```

### Feature Envy

```java
// BAD: Method uses another class's data more than its own
public class OrderProcessor {
    public double calculateShipping(Order order) {
        // Uses Order's data extensively
        double weight = order.getWeight();
        String country = order.getAddress().getCountry();
        String city = order.getAddress().getCity();
        boolean isPremium = order.getCustomer().isPremium();
        // This logic belongs in Order!
    }
}

// GOOD: Move method to the class whose data it uses
public class Order {
    public double calculateShipping() {
        // Uses its own data
        double weight = this.weight;
        String country = this.address.getCountry();
        // ...
    }
}
```

### Primitive Obsession

```java
// BAD: Using primitives for domain concepts
public class User {
    private String email;        // Just a string?
    private String phoneNumber;  // Any format?
    private int age;             // Negative allowed?
}

// GOOD: Use value objects
public class User {
    private Email email;
    private PhoneNumber phoneNumber;
    private Age age;
}

public record Email(String value) {
    public Email {
        if (!value.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
    }
}

public record Age(int value) {
    public Age {
        if (value < 0 || value > 150) {
            throw new IllegalArgumentException("Invalid age");
        }
    }
}
```

### Magic Numbers/Strings

```java
// BAD: Magic numbers
if (user.getAge() > 18) { }
if (order.getTotal() > 100) { }
if (status.equals("ACTIVE")) { }

// GOOD: Named constants
private static final int ADULT_AGE = 18;
private static final double FREE_SHIPPING_THRESHOLD = 100.0;

if (user.getAge() > ADULT_AGE) { }
if (order.getTotal() > FREE_SHIPPING_THRESHOLD) { }
if (status == Status.ACTIVE) { }  // Use enum
```

---

## 3️⃣ Immutability Patterns

### Why Immutability Matters

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BENEFITS OF IMMUTABILITY                              │
│                                                                          │
│   1. THREAD SAFETY                                                      │
│      Immutable objects can be shared freely between threads             │
│      No synchronization needed                                          │
│                                                                          │
│   2. SIMPLICITY                                                         │
│      Object state is fixed at construction                              │
│      No need to track state changes                                     │
│                                                                          │
│   3. FAILURE ATOMICITY                                                  │
│      If construction fails, object doesn't exist                        │
│      No partially constructed objects                                   │
│                                                                          │
│   4. SAFE KEYS                                                          │
│      Safe to use as Map keys or Set elements                           │
│      Hash code never changes                                            │
│                                                                          │
│   5. DEFENSIVE COPIES UNNECESSARY                                       │
│      Can share references freely                                        │
│      No need to copy on get/set                                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Creating Immutable Classes

```java
// Immutable class checklist:
// 1. Class is final (can't be subclassed)
// 2. All fields are final
// 3. All fields are private
// 4. No setters
// 5. Defensive copies of mutable fields

public final class ImmutablePerson {
    private final String name;
    private final int age;
    private final List<String> hobbies;
    private final Address address;
    
    public ImmutablePerson(String name, int age, List<String> hobbies, Address address) {
        this.name = name;
        this.age = age;
        // Defensive copy of mutable collection
        this.hobbies = List.copyOf(hobbies);
        // Defensive copy of mutable object (assuming Address is mutable)
        this.address = new Address(address);
    }
    
    public String getName() {
        return name;  // String is immutable, safe to return
    }
    
    public int getAge() {
        return age;  // Primitive, safe to return
    }
    
    public List<String> getHobbies() {
        return hobbies;  // Already immutable (List.copyOf)
    }
    
    public Address getAddress() {
        return new Address(address);  // Defensive copy on return
    }
    
    // "Setter" returns new instance
    public ImmutablePerson withAge(int newAge) {
        return new ImmutablePerson(this.name, newAge, this.hobbies, this.address);
    }
}

// Even simpler with records (Java 16+)
public record Person(String name, int age, List<String> hobbies) {
    public Person {
        hobbies = List.copyOf(hobbies);  // Defensive copy in compact constructor
    }
}
```

### Immutable Collections

```java
// Creating immutable collections (Java 9+)
List<String> list = List.of("a", "b", "c");
Set<String> set = Set.of("a", "b", "c");
Map<String, Integer> map = Map.of("a", 1, "b", 2);

// From existing collection
List<String> immutableList = List.copyOf(mutableList);

// Collectors
List<String> collected = stream.collect(Collectors.toUnmodifiableList());

// These throw UnsupportedOperationException on modification
list.add("d");  // Throws!
```

---

## 4️⃣ Null Safety Strategies

### The Problem with Null

```java
// Null can mean many things:
String name = null;  // Not set? Unknown? Error? Empty?

// NullPointerException is the most common exception
user.getAddress().getCity().toUpperCase();  // NPE if any is null!
```

### Strategy 1: Use Optional for Return Types

```java
// Instead of returning null
public User findUser(String id) {
    User user = repository.find(id);
    return user;  // Might be null - caller must check
}

// Return Optional
public Optional<User> findUser(String id) {
    User user = repository.find(id);
    return Optional.ofNullable(user);  // Explicit about possible absence
}

// Caller is forced to handle absence
findUser("123")
    .map(User::getName)
    .orElse("Unknown");
```

### Strategy 2: Use Annotations

```java
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

public class UserService {
    
    // Parameter must not be null
    public User createUser(@NotNull String name, @NotNull String email) {
        // IDE warns if null is passed
        return new User(name, email);
    }
    
    // Return value might be null
    @Nullable
    public User findUser(String id) {
        return repository.find(id);
    }
}
```

### Strategy 3: Fail Fast with Objects.requireNonNull

```java
public class Order {
    private final Customer customer;
    private final List<Item> items;
    
    public Order(Customer customer, List<Item> items) {
        // Fail immediately if null
        this.customer = Objects.requireNonNull(customer, "Customer cannot be null");
        this.items = Objects.requireNonNull(items, "Items cannot be null");
    }
}
```

### Strategy 4: Use Null Object Pattern

```java
// Instead of null, return a "null object"
public interface Logger {
    void log(String message);
}

public class ConsoleLogger implements Logger {
    @Override
    public void log(String message) {
        System.out.println(message);
    }
}

// Null object - does nothing but is safe to call
public class NullLogger implements Logger {
    @Override
    public void log(String message) {
        // Do nothing
    }
}

// Usage - no null checks needed
Logger logger = getLogger();  // Returns NullLogger if no logger configured
logger.log("Message");  // Safe, even if "null"
```

### Strategy 5: Return Empty Collections Instead of Null

```java
// BAD: Return null for empty
public List<Order> getOrders(String userId) {
    List<Order> orders = repository.findByUserId(userId);
    return orders;  // Might be null!
}

// Caller must check
List<Order> orders = getOrders(userId);
if (orders != null) {
    for (Order order : orders) { }
}

// GOOD: Return empty collection
public List<Order> getOrders(String userId) {
    List<Order> orders = repository.findByUserId(userId);
    return orders != null ? orders : Collections.emptyList();
}

// Caller can use directly
for (Order order : getOrders(userId)) {
    // Works even if empty
}
```

---

## 5️⃣ Exception Best Practices

### Use Exceptions for Exceptional Conditions

```java
// BAD: Using exceptions for control flow
public int findIndex(List<String> list, String target) {
    try {
        for (int i = 0; ; i++) {
            if (list.get(i).equals(target)) {
                return i;
            }
        }
    } catch (IndexOutOfBoundsException e) {
        return -1;  // Using exception to exit loop!
    }
}

// GOOD: Use normal control flow
public int findIndex(List<String> list, String target) {
    for (int i = 0; i < list.size(); i++) {
        if (list.get(i).equals(target)) {
            return i;
        }
    }
    return -1;
}
```

### Prefer Specific Exceptions

```java
// BAD: Generic exception
public void processFile(String path) throws Exception {
    // What could go wrong? Everything!
}

// GOOD: Specific exceptions
public void processFile(String path) throws FileNotFoundException, IOException {
    // Caller knows exactly what to handle
}
```

### Include Useful Information in Exceptions

```java
// BAD: Useless message
throw new IllegalArgumentException("Invalid value");

// GOOD: Include context
throw new IllegalArgumentException(
    "User age must be between 0 and 150, but was: " + age);

// Even better: Custom exception with context
public class InvalidAgeException extends IllegalArgumentException {
    private final int providedAge;
    private final int minAge;
    private final int maxAge;
    
    public InvalidAgeException(int providedAge, int minAge, int maxAge) {
        super(String.format("Age must be between %d and %d, but was: %d", 
            minAge, maxAge, providedAge));
        this.providedAge = providedAge;
        this.minAge = minAge;
        this.maxAge = maxAge;
    }
    
    // Getters for programmatic access
}
```

### Don't Swallow Exceptions

```java
// BAD: Swallowing exception
try {
    processFile(path);
} catch (IOException e) {
    // Silently ignored - bugs will be hard to find!
}

// GOOD: At minimum, log it
try {
    processFile(path);
} catch (IOException e) {
    log.error("Failed to process file: " + path, e);
    throw new ProcessingException("File processing failed", e);
}
```

### Use Try-with-Resources

```java
// BAD: Manual resource management
InputStream in = null;
try {
    in = new FileInputStream(path);
    // Use stream
} finally {
    if (in != null) {
        try {
            in.close();
        } catch (IOException e) {
            // Swallowed!
        }
    }
}

// GOOD: Try-with-resources
try (InputStream in = new FileInputStream(path)) {
    // Use stream
}  // Automatically closed, even on exception
```

---

## 6️⃣ API Design Best Practices

### Make APIs Hard to Misuse

```java
// BAD: Easy to swap parameters
public void transfer(Account from, Account to, double amount) { }
transfer(toAccount, fromAccount, 100);  // Oops! Swapped accounts

// GOOD: Use types to prevent errors
public void transfer(SourceAccount from, DestinationAccount to, Money amount) { }
// Can't accidentally swap - different types

// Or use builder pattern
TransferRequest.builder()
    .from(sourceAccount)
    .to(destinationAccount)
    .amount(Money.of(100, USD))
    .build()
    .execute();
```

### Return Appropriate Types

```java
// BAD: Return array (mutable, no methods)
public String[] getNames() {
    return names.toArray(new String[0]);
}

// GOOD: Return List (rich API, can be immutable)
public List<String> getNames() {
    return List.copyOf(names);
}
```

### Use Overloading Judiciously

```java
// BAD: Confusing overloads
public void process(List<String> items) { }
public void process(Set<String> items) { }
public void process(Collection<String> items) { }

// Which is called?
process(new ArrayList<>());  // List version
process(new HashSet<>());    // Set version
Collection<String> c = new ArrayList<>();
process(c);  // Collection version! Not List!

// GOOD: Different names
public void processList(List<String> items) { }
public void processSet(Set<String> items) { }
```

---

## 7️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between checked and unchecked exceptions?**

A: **Checked exceptions** (extend `Exception` but not `RuntimeException`):
- Must be declared in method signature or caught
- Represent recoverable conditions
- Examples: `IOException`, `SQLException`

**Unchecked exceptions** (extend `RuntimeException`):
- Don't need to be declared or caught
- Usually represent programming errors
- Examples: `NullPointerException`, `IllegalArgumentException`

Use checked exceptions for recoverable conditions where the caller can take action. Use unchecked for programming errors.

**Q: Why is immutability important?**

A: Immutability provides several benefits:

1. **Thread safety**: Immutable objects can be shared between threads without synchronization
2. **Simplicity**: No need to track state changes
3. **Safe as keys**: Can be used in HashMaps/HashSets without issues
4. **Defensive copies unnecessary**: Can share references freely

Create immutable classes by: making class final, all fields final and private, no setters, defensive copies of mutable fields.

### L5 (Mid-Level) Questions

**Q: What are some common code smells and how do you fix them?**

A: Common code smells:

1. **Long Method**: Extract smaller methods with descriptive names
2. **Large Class**: Split into smaller classes with single responsibility
3. **Feature Envy**: Move method to the class whose data it uses
4. **Primitive Obsession**: Create value objects for domain concepts
5. **Magic Numbers**: Replace with named constants or enums
6. **Duplicate Code**: Extract to shared method or class

The key is recognizing when code is hard to understand, test, or modify, then applying appropriate refactoring.

**Q: How do you handle null safely in Java?**

A: Multiple strategies:

1. **Optional** for return types when absence is valid
2. **@NotNull/@Nullable** annotations for documentation and IDE support
3. **Objects.requireNonNull()** to fail fast at method entry
4. **Null Object pattern** for default behavior
5. **Empty collections** instead of null for collections
6. **Defensive programming** - validate inputs early

The best approach depends on context. Use Optional for APIs, fail-fast for internal code, annotations for documentation.

### L6 (Senior) Questions

**Q: How would you design an API that's hard to misuse?**

A: Key principles:

1. **Type safety**: Use different types for different concepts (SourceAccount vs DestinationAccount)
2. **Builder pattern**: For complex objects with many parameters
3. **Immutability**: Prevent unexpected modifications
4. **Fail fast**: Validate at construction, not at use
5. **Consistent naming**: Follow conventions (getX for getters, isX for booleans)
6. **Minimal API surface**: Only expose what's necessary
7. **Good defaults**: Sensible behavior without configuration
8. **Documentation**: Clear javadoc with examples

Example: Instead of `transfer(Account, Account, double)`, use a builder with typed parameters that validates at build time.

---

## 8️⃣ One Clean Mental Summary

Java best practices center on **clarity**, **safety**, and **maintainability**. Use **static factory methods** over constructors for expressiveness, **builders** for complex objects, and **dependency injection** for flexibility. Make classes **immutable** when possible - it eliminates whole categories of bugs. Handle **nulls** explicitly: use Optional for return types, fail-fast with requireNonNull, return empty collections instead of null. **Exceptions** should be for exceptional conditions, specific, and include context. Recognize **code smells** (long methods, large classes, magic numbers) and refactor. Design **APIs** that are hard to misuse through type safety and validation. The overarching principle: write code that's easy to understand, test, and modify. When in doubt, prefer simplicity and explicitness over cleverness.

