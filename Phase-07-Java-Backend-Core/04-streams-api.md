# 🌊 Java Streams API Deep Dive

---

## 0️⃣ Prerequisites

Before diving into Streams, you need to understand:

- **Collections**: Lists, Sets, Maps (covered in `03-collections-deep-dive.md`)
- **Lambda Expressions**: Anonymous functions like `x -> x * 2` or `(a, b) -> a + b`
- **Functional Interface**: An interface with exactly one abstract method (e.g., `Predicate`, `Function`, `Consumer`)
- **Method Reference**: Shorthand for lambdas like `String::toUpperCase` instead of `s -> s.toUpperCase()`

Quick refresher on lambdas:

```java
// Traditional anonymous class
Comparator<String> comp1 = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};

// Lambda expression (same thing, shorter)
Comparator<String> comp2 = (a, b) -> a.length() - b.length();

// Method reference (even shorter when possible)
Comparator<String> comp3 = Comparator.comparingInt(String::length);
```

If you understand that lambdas are just compact ways to pass behavior, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Imagine processing a list of orders to find the total revenue from orders over $100 placed by premium customers:

```java
// WITHOUT Streams - Imperative approach
List<Order> orders = getOrders();
double totalRevenue = 0;

for (Order order : orders) {
    if (order.getAmount() > 100) {
        Customer customer = getCustomer(order.getCustomerId());
        if (customer.isPremium()) {
            totalRevenue += order.getAmount();
        }
    }
}
```

**Problems with this approach**:

1. **Verbose**: 8 lines for a simple operation
2. **Mutable state**: `totalRevenue` changes, hard to parallelize
3. **Intertwined logic**: Filtering, mapping, and reducing all mixed together
4. **Hard to compose**: Can't easily add or remove steps
5. **Not parallelizable**: Can't safely run on multiple threads

### What Code Looked Like Before Streams

Before Java 8 (2014), processing collections required:

```java
// Finding all adult users, sorted by name
List<User> adults = new ArrayList<>();
for (User user : users) {
    if (user.getAge() >= 18) {
        adults.add(user);
    }
}
Collections.sort(adults, new Comparator<User>() {
    @Override
    public int compare(User a, User b) {
        return a.getName().compareTo(b.getName());
    }
});

// Getting just the names
List<String> names = new ArrayList<>();
for (User user : adults) {
    names.add(user.getName());
}
```

**15 lines for a simple filter-sort-map operation!**

### What Breaks Without Streams

1. **Boilerplate explosion**: Simple operations require many lines
2. **Parallel processing nightmare**: Manual thread management
3. **Composition impossible**: Can't chain operations elegantly
4. **Readability suffers**: Business logic buried in loop mechanics
5. **Bugs from mutable state**: Off-by-one errors, concurrent modification

### Real Examples of the Problem

**Big Data Processing**: Before Streams, teams wrote custom parallel processing code. Bugs were common, performance was inconsistent, and code was unmaintainable.

**ETL Pipelines**: Extract-Transform-Load operations required verbose, error-prone loops. Streams made these declarative and parallelizable.

---

## 2️⃣ Intuition and Mental Model

### The Assembly Line Analogy

Think of Streams like a factory assembly line:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAM AS ASSEMBLY LINE                               │
│                                                                          │
│   Raw Materials (Source)                                                │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┐                                    │
│   │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │  List of numbers                   │
│   └───┴───┴───┴───┴───┴───┴───┴───┘                                    │
│                    │                                                     │
│                    ▼                                                     │
│   ┌─────────────────────────────────┐                                   │
│   │  Station 1: FILTER              │  Keep only even numbers           │
│   │  (x -> x % 2 == 0)              │                                   │
│   └─────────────────────────────────┘                                   │
│                    │                                                     │
│                    ▼                                                     │
│   ┌───┬───┬───┬───┐                                                     │
│   │ 2 │ 4 │ 6 │ 8 │  Filtered items                                     │
│   └───┴───┴───┴───┘                                                     │
│                    │                                                     │
│                    ▼                                                     │
│   ┌─────────────────────────────────┐                                   │
│   │  Station 2: MAP                 │  Square each number               │
│   │  (x -> x * x)                   │                                   │
│   └─────────────────────────────────┘                                   │
│                    │                                                     │
│                    ▼                                                     │
│   ┌───┬────┬────┬────┐                                                  │
│   │ 4 │ 16 │ 36 │ 64 │  Transformed items                               │
│   └───┴────┴────┴────┘                                                  │
│                    │                                                     │
│                    ▼                                                     │
│   ┌─────────────────────────────────┐                                   │
│   │  Station 3: REDUCE              │  Sum all numbers                  │
│   │  (sum)                          │                                   │
│   └─────────────────────────────────┘                                   │
│                    │                                                     │
│                    ▼                                                     │
│                 ┌─────┐                                                  │
│                 │ 120 │  Final product                                  │
│                 └─────┘                                                  │
│                                                                          │
│   KEY INSIGHT: Items flow through stations one at a time                │
│   The assembly line doesn't store intermediate results                  │
│   Each station does ONE thing                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insights**:

- **Source**: Where items come from (Collection, array, file, etc.)
- **Intermediate operations**: Stations that transform items (filter, map, sort)
- **Terminal operation**: Final station that produces result (collect, reduce, forEach)
- **Lazy evaluation**: Assembly line doesn't start until someone needs the final product
- **Single use**: Once items pass through, the line is done

---

## 3️⃣ How It Works Internally

### Stream Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAM PIPELINE STRUCTURE                             │
│                                                                          │
│   ┌──────────────┐                                                      │
│   │    SOURCE    │  Collection, Array, Generator, I/O                   │
│   └──────┬───────┘                                                      │
│          │                                                               │
│          ▼                                                               │
│   ┌──────────────┐                                                      │
│   │ INTERMEDIATE │  filter(), map(), flatMap(), sorted(),               │
│   │  OPERATIONS  │  distinct(), limit(), skip(), peek()                 │
│   │   (Lazy)     │                                                      │
│   └──────┬───────┘                                                      │
│          │         Can chain multiple intermediate operations           │
│          ▼                                                               │
│   ┌──────────────┐                                                      │
│   │   TERMINAL   │  collect(), reduce(), forEach(), count(),            │
│   │  OPERATION   │  findFirst(), anyMatch(), toArray()                  │
│   │   (Eager)    │                                                      │
│   └──────┬───────┘                                                      │
│          │                                                               │
│          ▼                                                               │
│   ┌──────────────┐                                                      │
│   │    RESULT    │  Collection, Value, Side Effect                      │
│   └──────────────┘                                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Lazy Evaluation Explained

```java
// This code does NOTHING until terminal operation!
Stream<Integer> stream = numbers.stream()
    .filter(n -> {
        System.out.println("Filtering: " + n);
        return n > 5;
    })
    .map(n -> {
        System.out.println("Mapping: " + n);
        return n * 2;
    });

// No output yet! Stream is just a recipe.

// NOW it executes
List<Integer> result = stream.collect(Collectors.toList());
// Prints: Filtering: 1, Filtering: 2, ... Filtering: 6, Mapping: 6, ...
```

### Short-Circuit Optimization

```java
// findFirst() stops as soon as it finds a match
Optional<Integer> first = numbers.stream()
    .filter(n -> {
        System.out.println("Checking: " + n);
        return n > 5;
    })
    .findFirst();

// If numbers = [1, 2, 3, 6, 7, 8, 9]
// Only prints: Checking: 1, Checking: 2, Checking: 3, Checking: 6
// Stops at 6! Doesn't check 7, 8, 9
```

### Internal Iteration vs External Iteration

```java
// EXTERNAL iteration (traditional) - YOU control the loop
for (int i = 0; i < list.size(); i++) {
    String item = list.get(i);
    // process item
}

// INTERNAL iteration (streams) - STREAM controls the loop
list.stream()
    .forEach(item -> {
        // process item
    });

// Why internal is better:
// 1. Stream can optimize (parallel, short-circuit)
// 2. Less boilerplate
// 3. Declarative: say WHAT, not HOW
```

---

## 4️⃣ Simulation-First Explanation

Let's trace a complete stream operation step by step.

### Scenario: Processing Orders

```java
List<Order> orders = List.of(
    new Order("O1", "Alice", 150.0, "ELECTRONICS"),
    new Order("O2", "Bob", 50.0, "BOOKS"),
    new Order("O3", "Alice", 200.0, "ELECTRONICS"),
    new Order("O4", "Charlie", 75.0, "CLOTHING"),
    new Order("O5", "Alice", 300.0, "ELECTRONICS")
);

// Find total spending by Alice on Electronics
double total = orders.stream()
    .filter(o -> o.getCustomer().equals("Alice"))
    .filter(o -> o.getCategory().equals("ELECTRONICS"))
    .mapToDouble(Order::getAmount)
    .sum();
```

### Step-by-Step Execution

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAM EXECUTION TRACE                                │
│                                                                          │
│   Source: [O1, O2, O3, O4, O5]                                          │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing O1 (Alice, 150, ELECTRONICS):                              │
│   1. filter(customer=Alice) → PASS                                      │
│   2. filter(category=ELECTRONICS) → PASS                                │
│   3. mapToDouble → 150.0                                                │
│   4. Add to sum: 0 + 150 = 150                                          │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing O2 (Bob, 50, BOOKS):                                       │
│   1. filter(customer=Alice) → FAIL                                      │
│   ❌ Skipped! Doesn't reach map or sum.                                 │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing O3 (Alice, 200, ELECTRONICS):                              │
│   1. filter(customer=Alice) → PASS                                      │
│   2. filter(category=ELECTRONICS) → PASS                                │
│   3. mapToDouble → 200.0                                                │
│   4. Add to sum: 150 + 200 = 350                                        │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing O4 (Charlie, 75, CLOTHING):                                │
│   1. filter(customer=Alice) → FAIL                                      │
│   ❌ Skipped!                                                           │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Processing O5 (Alice, 300, ELECTRONICS):                              │
│   1. filter(customer=Alice) → PASS                                      │
│   2. filter(category=ELECTRONICS) → PASS                                │
│   3. mapToDouble → 300.0                                                │
│   4. Add to sum: 350 + 300 = 650                                        │
│                                                                          │
│   ═══════════════════════════════════════════════════════════════════   │
│   Final Result: 650.0                                                    │
│                                                                          │
│   KEY INSIGHT: Each element flows through ALL operations before         │
│   the next element starts. This is "loop fusion" - one pass through     │
│   the data, not multiple passes.                                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Netflix's Data Processing**:

```java
// Processing viewing history for recommendations
public List<Recommendation> getRecommendations(String userId) {
    return viewingHistory.stream()
        .filter(view -> view.getUserId().equals(userId))
        .filter(view -> view.getCompletionRate() > 0.7)  // Watched 70%+
        .map(View::getContentId)
        .distinct()
        .flatMap(contentId -> findSimilarContent(contentId).stream())
        .filter(content -> !hasWatched(userId, content))
        .sorted(Comparator.comparing(Content::getPopularityScore).reversed())
        .limit(20)
        .map(content -> new Recommendation(content, calculateScore(content)))
        .collect(Collectors.toList());
}
```

**Amazon's Order Processing**:

```java
// Daily order summary by category
public Map<String, OrderSummary> getDailySummary(LocalDate date) {
    return orders.stream()
        .filter(order -> order.getDate().equals(date))
        .filter(order -> order.getStatus() == OrderStatus.COMPLETED)
        .collect(Collectors.groupingBy(
            Order::getCategory,
            Collectors.collectingAndThen(
                Collectors.toList(),
                orders -> new OrderSummary(
                    orders.size(),
                    orders.stream().mapToDouble(Order::getAmount).sum(),
                    orders.stream().mapToDouble(Order::getAmount).average().orElse(0)
                )
            )
        ));
}
```

### Real Workflows and Tooling

**Log Analysis**:

```java
// Find all ERROR logs from the last hour, grouped by service
Map<String, List<LogEntry>> errorsByService = logs.stream()
    .filter(log -> log.getTimestamp().isAfter(Instant.now().minus(1, ChronoUnit.HOURS)))
    .filter(log -> log.getLevel() == LogLevel.ERROR)
    .collect(Collectors.groupingBy(LogEntry::getServiceName));
```

**Database Result Processing**:

```java
// Process large result set without loading all into memory
try (Stream<User> users = userRepository.streamAll()) {
    users.filter(User::isActive)
         .filter(user -> user.getLastLogin().isBefore(thirtyDaysAgo))
         .forEach(user -> sendReactivationEmail(user));
}
```

### Production War Stories

**The Parallel Stream Disaster**:

A team used parallel streams for database calls:

```java
// WRONG: Parallel stream with blocking I/O
users.parallelStream()
    .map(user -> userRepository.findDetails(user.getId()))  // Blocking DB call!
    .collect(Collectors.toList());

// Problem: Common ForkJoinPool was exhausted
// All other parallel streams in the JVM blocked!
```

**The fix**:

```java
// RIGHT: Use CompletableFuture with custom executor for I/O
ExecutorService dbExecutor = Executors.newFixedThreadPool(20);

List<CompletableFuture<UserDetails>> futures = users.stream()
    .map(user -> CompletableFuture.supplyAsync(
        () -> userRepository.findDetails(user.getId()),
        dbExecutor
    ))
    .collect(Collectors.toList());

List<UserDetails> details = futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

---

## 6️⃣ How to Implement: Complete Examples

### Creating Streams

```java
import java.util.stream.*;
import java.util.*;

public class StreamCreation {
    
    public static void main(String[] args) {
        
        // 1. From Collection
        List<String> list = List.of("a", "b", "c");
        Stream<String> streamFromList = list.stream();
        
        // 2. From Array
        String[] array = {"a", "b", "c"};
        Stream<String> streamFromArray = Arrays.stream(array);
        
        // 3. From values directly
        Stream<String> streamOfValues = Stream.of("a", "b", "c");
        
        // 4. Empty stream
        Stream<String> emptyStream = Stream.empty();
        
        // 5. Infinite stream with generate
        Stream<Double> randoms = Stream.generate(Math::random);
        
        // 6. Infinite stream with iterate
        Stream<Integer> evenNumbers = Stream.iterate(0, n -> n + 2);
        
        // 7. Bounded iterate (Java 9+)
        Stream<Integer> oneToTen = Stream.iterate(1, n -> n <= 10, n -> n + 1);
        
        // 8. From file lines
        // Stream<String> lines = Files.lines(Path.of("file.txt"));
        
        // 9. Primitive streams (avoid boxing)
        IntStream intStream = IntStream.range(1, 100);      // 1 to 99
        IntStream intStreamClosed = IntStream.rangeClosed(1, 100);  // 1 to 100
        LongStream longStream = LongStream.of(1L, 2L, 3L);
        DoubleStream doubleStream = DoubleStream.of(1.0, 2.0, 3.0);
        
        // 10. From String characters
        IntStream chars = "hello".chars();
    }
}
```

### Intermediate Operations

```java
public class IntermediateOperations {
    
    public static void main(String[] args) {
        List<Person> people = getPeople();
        
        // FILTER: Keep elements matching predicate
        people.stream()
            .filter(p -> p.getAge() >= 18)
            .forEach(System.out::println);
        
        // MAP: Transform each element
        people.stream()
            .map(Person::getName)  // Person -> String
            .forEach(System.out::println);
        
        // FLATMAP: Transform each element to stream, then flatten
        // Useful for one-to-many transformations
        List<List<String>> nestedList = List.of(
            List.of("a", "b"),
            List.of("c", "d"),
            List.of("e")
        );
        nestedList.stream()
            .flatMap(List::stream)  // Flatten nested lists
            .forEach(System.out::println);  // a, b, c, d, e
        
        // DISTINCT: Remove duplicates (uses equals())
        List.of(1, 2, 2, 3, 3, 3).stream()
            .distinct()
            .forEach(System.out::println);  // 1, 2, 3
        
        // SORTED: Sort elements
        people.stream()
            .sorted(Comparator.comparing(Person::getName))
            .forEach(System.out::println);
        
        // SORTED with custom comparator
        people.stream()
            .sorted(Comparator.comparing(Person::getAge).reversed())
            .forEach(System.out::println);
        
        // LIMIT: Take first N elements
        people.stream()
            .limit(5)
            .forEach(System.out::println);
        
        // SKIP: Skip first N elements
        people.stream()
            .skip(2)
            .forEach(System.out::println);
        
        // PEEK: Perform action without consuming (for debugging)
        people.stream()
            .peek(p -> System.out.println("Processing: " + p))
            .filter(p -> p.getAge() >= 18)
            .peek(p -> System.out.println("Passed filter: " + p))
            .collect(Collectors.toList());
        
        // TAKEWHILE (Java 9+): Take while predicate is true
        Stream.of(1, 2, 3, 4, 5, 1, 2)
            .takeWhile(n -> n < 4)
            .forEach(System.out::println);  // 1, 2, 3
        
        // DROPWHILE (Java 9+): Drop while predicate is true
        Stream.of(1, 2, 3, 4, 5, 1, 2)
            .dropWhile(n -> n < 4)
            .forEach(System.out::println);  // 4, 5, 1, 2
    }
}
```

### Terminal Operations

```java
public class TerminalOperations {
    
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5);
        List<Person> people = getPeople();
        
        // FOREACH: Perform action on each element
        numbers.stream()
            .forEach(System.out::println);
        
        // COLLECT: Gather results into collection
        List<String> names = people.stream()
            .map(Person::getName)
            .collect(Collectors.toList());
        
        Set<String> uniqueNames = people.stream()
            .map(Person::getName)
            .collect(Collectors.toSet());
        
        // TOARRAY: Convert to array
        String[] nameArray = people.stream()
            .map(Person::getName)
            .toArray(String[]::new);
        
        // REDUCE: Combine elements into single result
        int sum = numbers.stream()
            .reduce(0, (a, b) -> a + b);
        
        // REDUCE with method reference
        int product = numbers.stream()
            .reduce(1, (a, b) -> a * b);
        
        // REDUCE without identity (returns Optional)
        Optional<Integer> max = numbers.stream()
            .reduce(Integer::max);
        
        // COUNT: Count elements
        long count = people.stream()
            .filter(p -> p.getAge() >= 18)
            .count();
        
        // MIN/MAX: Find minimum/maximum
        Optional<Person> youngest = people.stream()
            .min(Comparator.comparing(Person::getAge));
        
        Optional<Person> oldest = people.stream()
            .max(Comparator.comparing(Person::getAge));
        
        // FINDFIRST: Find first element
        Optional<Person> firstAdult = people.stream()
            .filter(p -> p.getAge() >= 18)
            .findFirst();
        
        // FINDANY: Find any element (useful in parallel)
        Optional<Person> anyAdult = people.parallelStream()
            .filter(p -> p.getAge() >= 18)
            .findAny();
        
        // ANYMATCH: Check if any element matches
        boolean hasAdult = people.stream()
            .anyMatch(p -> p.getAge() >= 18);
        
        // ALLMATCH: Check if all elements match
        boolean allAdults = people.stream()
            .allMatch(p -> p.getAge() >= 18);
        
        // NONEMATCH: Check if no elements match
        boolean noChildren = people.stream()
            .noneMatch(p -> p.getAge() < 18);
    }
}
```

### Collectors Deep Dive

```java
import java.util.stream.Collectors;

public class CollectorsExamples {
    
    public static void main(String[] args) {
        List<Person> people = getPeople();
        
        // toList, toSet, toCollection
        List<String> list = people.stream()
            .map(Person::getName)
            .collect(Collectors.toList());
        
        Set<String> set = people.stream()
            .map(Person::getName)
            .collect(Collectors.toSet());
        
        TreeSet<String> treeSet = people.stream()
            .map(Person::getName)
            .collect(Collectors.toCollection(TreeSet::new));
        
        // toMap: Key-value pairs
        Map<String, Integer> nameToAge = people.stream()
            .collect(Collectors.toMap(
                Person::getName,      // Key mapper
                Person::getAge        // Value mapper
            ));
        
        // toMap with merge function (handle duplicates)
        Map<String, Integer> nameToMaxAge = people.stream()
            .collect(Collectors.toMap(
                Person::getName,
                Person::getAge,
                Integer::max  // If duplicate key, keep max age
            ));
        
        // groupingBy: Group elements by classifier
        Map<String, List<Person>> byCity = people.stream()
            .collect(Collectors.groupingBy(Person::getCity));
        
        // groupingBy with downstream collector
        Map<String, Long> countByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.counting()
            ));
        
        Map<String, Double> avgAgeByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.averagingInt(Person::getAge)
            ));
        
        // Multi-level grouping
        Map<String, Map<String, List<Person>>> byCityThenGender = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.groupingBy(Person::getGender)
            ));
        
        // partitioningBy: Split into two groups (true/false)
        Map<Boolean, List<Person>> adultPartition = people.stream()
            .collect(Collectors.partitioningBy(p -> p.getAge() >= 18));
        
        List<Person> adults = adultPartition.get(true);
        List<Person> minors = adultPartition.get(false);
        
        // joining: Concatenate strings
        String allNames = people.stream()
            .map(Person::getName)
            .collect(Collectors.joining(", "));  // "Alice, Bob, Charlie"
        
        String withPrefixSuffix = people.stream()
            .map(Person::getName)
            .collect(Collectors.joining(", ", "[", "]"));  // "[Alice, Bob, Charlie]"
        
        // summarizingInt: Get statistics
        IntSummaryStatistics stats = people.stream()
            .collect(Collectors.summarizingInt(Person::getAge));
        
        System.out.println("Count: " + stats.getCount());
        System.out.println("Sum: " + stats.getSum());
        System.out.println("Min: " + stats.getMin());
        System.out.println("Max: " + stats.getMax());
        System.out.println("Average: " + stats.getAverage());
        
        // reducing: Custom reduction
        Optional<Person> oldestPerson = people.stream()
            .collect(Collectors.reducing(
                (p1, p2) -> p1.getAge() > p2.getAge() ? p1 : p2
            ));
        
        // collectingAndThen: Transform result
        List<String> unmodifiableNames = people.stream()
            .map(Person::getName)
            .collect(Collectors.collectingAndThen(
                Collectors.toList(),
                Collections::unmodifiableList
            ));
        
        // mapping: Map before collecting
        Map<String, Set<String>> namesByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.mapping(
                    Person::getName,
                    Collectors.toSet()
                )
            ));
        
        // filtering (Java 9+): Filter before collecting
        Map<String, List<Person>> adultsByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.filtering(
                    p -> p.getAge() >= 18,
                    Collectors.toList()
                )
            ));
        
        // flatMapping (Java 9+): FlatMap before collecting
        Map<String, Set<String>> skillsByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.flatMapping(
                    p -> p.getSkills().stream(),
                    Collectors.toSet()
                )
            ));
        
        // teeing (Java 12+): Apply two collectors, merge results
        var result = people.stream()
            .collect(Collectors.teeing(
                Collectors.counting(),
                Collectors.averagingInt(Person::getAge),
                (count, avgAge) -> "Count: " + count + ", Avg Age: " + avgAge
            ));
    }
}
```

### Parallel Streams

```java
public class ParallelStreamExamples {
    
    public static void main(String[] args) {
        List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
            .boxed()
            .collect(Collectors.toList());
        
        // Create parallel stream
        long sum1 = numbers.parallelStream()
            .mapToLong(Integer::longValue)
            .sum();
        
        // Convert sequential to parallel
        long sum2 = numbers.stream()
            .parallel()
            .mapToLong(Integer::longValue)
            .sum();
        
        // Convert parallel to sequential
        long sum3 = numbers.parallelStream()
            .sequential()
            .mapToLong(Integer::longValue)
            .sum();
        
        // Check if parallel
        boolean isParallel = numbers.parallelStream().isParallel();  // true
        
        // WHEN TO USE PARALLEL STREAMS:
        // 1. Large data sets (10,000+ elements)
        // 2. CPU-bound operations (not I/O)
        // 3. Stateless, independent operations
        // 4. No shared mutable state
        // 5. Easily splittable source (ArrayList, not LinkedList)
        
        // GOOD: CPU-intensive, no shared state
        List<Integer> results = numbers.parallelStream()
            .map(n -> expensiveComputation(n))
            .collect(Collectors.toList());
        
        // BAD: I/O operations (blocks common pool)
        // numbers.parallelStream()
        //     .forEach(n -> saveToDatabase(n));  // DON'T DO THIS!
        
        // BAD: Shared mutable state
        // List<Integer> shared = new ArrayList<>();
        // numbers.parallelStream()
        //     .forEach(n -> shared.add(n));  // Race condition!
        
        // RIGHT: Use thread-safe collector
        List<Integer> safe = numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());  // Thread-safe
    }
    
    private static int expensiveComputation(int n) {
        // Simulate CPU work
        return (int) Math.sqrt(n * n + 1);
    }
}
```

### Custom Collector

```java
import java.util.stream.Collector;

public class CustomCollectorExample {
    
    /**
     * Custom collector that collects strings into a comma-separated 
     * string with a maximum length, adding "..." if truncated.
     */
    public static Collector<String, StringBuilder, String> 
            truncatedJoining(int maxLength) {
        
        return Collector.of(
            // Supplier: Create accumulator
            StringBuilder::new,
            
            // Accumulator: Add element to accumulator
            (sb, str) -> {
                if (sb.length() > 0) {
                    sb.append(", ");
                }
                sb.append(str);
            },
            
            // Combiner: Merge two accumulators (for parallel)
            (sb1, sb2) -> {
                if (sb1.length() > 0 && sb2.length() > 0) {
                    sb1.append(", ");
                }
                return sb1.append(sb2);
            },
            
            // Finisher: Transform accumulator to result
            sb -> {
                if (sb.length() <= maxLength) {
                    return sb.toString();
                }
                return sb.substring(0, maxLength - 3) + "...";
            }
        );
    }
    
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie", "David", "Eve");
        
        String result = names.stream()
            .collect(truncatedJoining(20));
        
        System.out.println(result);  // "Alice, Bob, Charl..."
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

**1. Reusing Streams**

```java
// WRONG: Stream can only be consumed once!
Stream<String> stream = list.stream().filter(s -> s.length() > 3);
stream.forEach(System.out::println);
stream.count();  // IllegalStateException: stream has already been operated upon

// RIGHT: Create new stream for each operation
list.stream().filter(s -> s.length() > 3).forEach(System.out::println);
long count = list.stream().filter(s -> s.length() > 3).count();

// OR: Collect to list first if you need multiple operations
List<String> filtered = list.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());
filtered.forEach(System.out::println);
long count = filtered.size();
```

**2. Side Effects in Lambdas**

```java
// WRONG: Modifying external state
List<String> results = new ArrayList<>();
list.stream()
    .filter(s -> s.length() > 3)
    .forEach(s -> results.add(s));  // Side effect!

// RIGHT: Use collect
List<String> results = list.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());
```

**3. Parallel Stream with Wrong Data Source**

```java
// WRONG: LinkedList is not efficiently splittable
LinkedList<Integer> linkedList = new LinkedList<>(numbers);
linkedList.parallelStream()  // Poor performance!
    .map(n -> n * 2)
    .collect(Collectors.toList());

// RIGHT: Use ArrayList or array
ArrayList<Integer> arrayList = new ArrayList<>(numbers);
arrayList.parallelStream()  // Good performance!
    .map(n -> n * 2)
    .collect(Collectors.toList());
```

**4. Using Parallel Streams for I/O**

```java
// WRONG: Parallel stream with blocking I/O
urls.parallelStream()
    .map(url -> fetchFromNetwork(url))  // Blocks ForkJoinPool!
    .collect(Collectors.toList());

// RIGHT: Use CompletableFuture with custom executor
ExecutorService executor = Executors.newFixedThreadPool(10);
List<CompletableFuture<String>> futures = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(
        () -> fetchFromNetwork(url), executor))
    .collect(Collectors.toList());

List<String> results = futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

**5. Ignoring Optional Results**

```java
// WRONG: Assuming result exists
String first = list.stream()
    .filter(s -> s.startsWith("X"))
    .findFirst()
    .get();  // NoSuchElementException if no match!

// RIGHT: Handle Optional properly
String first = list.stream()
    .filter(s -> s.startsWith("X"))
    .findFirst()
    .orElse("default");

// OR
list.stream()
    .filter(s -> s.startsWith("X"))
    .findFirst()
    .ifPresent(System.out::println);
```

**6. Infinite Streams Without Limit**

```java
// WRONG: Infinite loop!
Stream.iterate(0, n -> n + 1)
    .forEach(System.out::println);  // Never terminates!

// RIGHT: Use limit()
Stream.iterate(0, n -> n + 1)
    .limit(100)
    .forEach(System.out::println);

// OR: Use bounded iterate (Java 9+)
Stream.iterate(0, n -> n < 100, n -> n + 1)
    .forEach(System.out::println);
```

### Performance Considerations

| Scenario                         | Sequential | Parallel | Recommendation          |
| -------------------------------- | ---------- | -------- | ----------------------- |
| Small collection (<1000)         | ✅         | ❌       | Sequential              |
| Large collection, CPU-bound      | ✅         | ✅       | Parallel                |
| I/O operations                   | ✅         | ❌       | Sequential + async      |
| Order matters                    | ✅         | ⚠️       | Sequential or forEachOrdered |
| LinkedList source                | ✅         | ❌       | Sequential              |
| Stateful operations              | ✅         | ⚠️       | Be careful              |

---

## 8️⃣ When NOT to Use Streams

### Situations Where Loops Are Better

**1. Simple Iterations**

```java
// OVERKILL: Stream for simple loop
list.stream().forEach(System.out::println);

// SIMPLER: Enhanced for loop
for (String item : list) {
    System.out.println(item);
}
```

**2. Early Exit with Complex Logic**

```java
// AWKWARD: Stream with early exit
boolean found = false;
for (Item item : items) {
    if (item.isSpecial()) {
        processSpecial(item);
        found = true;
        break;
    }
    if (item.isError()) {
        handleError(item);
        return;  // Early exit
    }
}

// Streams don't have break/return like this
// Would need complex workarounds
```

**3. Index-Based Operations**

```java
// AWKWARD: Need index in stream
IntStream.range(0, list.size())
    .forEach(i -> System.out.println(i + ": " + list.get(i)));

// CLEANER: Traditional loop
for (int i = 0; i < list.size(); i++) {
    System.out.println(i + ": " + list.get(i));
}
```

**4. Modifying Elements In-Place**

```java
// IMPOSSIBLE: Streams don't modify source
list.stream()
    .map(s -> s.toUpperCase())  // Creates new stream, doesn't modify list

// CORRECT: Use replaceAll
list.replaceAll(String::toUpperCase);

// OR: Traditional loop
for (int i = 0; i < list.size(); i++) {
    list.set(i, list.get(i).toUpperCase());
}
```

**5. Performance-Critical Code**

```java
// SLOWER: Stream overhead
int sum = numbers.stream()
    .mapToInt(Integer::intValue)
    .sum();

// FASTER: Primitive loop (no boxing, no stream overhead)
int sum = 0;
for (int n : numbers) {
    sum += n;
}
```

---

## 9️⃣ Comparison with Alternatives

### Streams vs Traditional Loops

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAMS vs TRADITIONAL LOOPS                          │
│                                                                          │
│   Traditional Loop:                    Stream:                          │
│   ─────────────────                    ───────                          │
│   List<String> result = new ArrayList<>();                              │
│   for (Person p : people) {            List<String> result =            │
│       if (p.getAge() >= 18) {              people.stream()              │
│           result.add(p.getName());             .filter(p -> p.getAge() >= 18)
│       }                                        .map(Person::getName)    │
│   }                                            .collect(toList());      │
│                                                                          │
│   Pros:                                Pros:                            │
│   - Familiar                           - Declarative                    │
│   - No overhead                        - Composable                     │
│   - Easy debugging                     - Parallelizable                 │
│   - Full control                       - Concise                        │
│                                                                          │
│   Cons:                                Cons:                            │
│   - Verbose                            - Learning curve                 │
│   - Mutable state                      - Debugging harder               │
│   - Not composable                     - Overhead for small data        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Streams vs RxJava/Reactor

| Feature                | Java Streams      | RxJava/Reactor          |
| ---------------------- | ----------------- | ----------------------- |
| Async support          | No                | Yes                     |
| Backpressure           | No                | Yes                     |
| Reusability            | Single use        | Reusable                |
| Error handling         | Exceptions        | onError callbacks       |
| Operators              | ~40               | 200+                    |
| Use case               | Collection processing | Async/reactive systems |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between intermediate and terminal operations?**

A: Intermediate operations transform a stream into another stream and are lazy. They don't execute until a terminal operation is called. Examples: `filter()`, `map()`, `sorted()`.

Terminal operations produce a result or side effect and trigger the stream pipeline execution. After a terminal operation, the stream is consumed and cannot be reused. Examples: `collect()`, `forEach()`, `count()`.

The key insight is laziness: `list.stream().filter(...).map(...)` does nothing until you add something like `.collect(toList())`.

**Q: What is the difference between `map()` and `flatMap()`?**

A: `map()` transforms each element one-to-one. If you have a stream of 5 elements and map each, you get 5 transformed elements.

`flatMap()` transforms each element to zero or more elements and flattens the result. It's used for one-to-many transformations.

Example:
```java
// map: Person -> Name (one-to-one)
people.stream().map(Person::getName)  // Stream<String>

// flatMap: Person -> Skills (one-to-many)
people.stream().flatMap(p -> p.getSkills().stream())  // Stream<String>

// Without flatMap, you'd get Stream<List<String>> instead of Stream<String>
```

### L5 (Mid-Level) Questions

**Q: When should you use parallel streams?**

A: Use parallel streams when:
1. **Large data set**: At least 10,000+ elements
2. **CPU-bound operations**: Computation, not I/O
3. **Independent operations**: No shared mutable state
4. **Splittable source**: ArrayList, arrays (not LinkedList)
5. **No ordering requirement**: Or use `forEachOrdered()`

Don't use parallel streams for:
- Small collections (overhead exceeds benefit)
- I/O operations (blocks common ForkJoinPool)
- Operations with side effects
- When order matters and you can't use ordered operations

**Q: How would you handle exceptions in streams?**

A: Streams don't handle checked exceptions well. Options:

```java
// Option 1: Wrap in unchecked exception
list.stream()
    .map(item -> {
        try {
            return riskyOperation(item);
        } catch (CheckedException e) {
            throw new RuntimeException(e);
        }
    })
    .collect(toList());

// Option 2: Use Either/Try pattern
list.stream()
    .map(item -> Try.of(() -> riskyOperation(item)))
    .filter(Try::isSuccess)
    .map(Try::get)
    .collect(toList());

// Option 3: Use Optional for recoverable errors
list.stream()
    .map(item -> {
        try {
            return Optional.of(riskyOperation(item));
        } catch (Exception e) {
            return Optional.<Result>empty();
        }
    })
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(toList());
```

### L6 (Senior) Questions

**Q: Explain the internal implementation of parallel streams.**

A: Parallel streams use the Fork/Join framework internally:

1. **Splitting**: The stream source is split into chunks using `Spliterator`. ArrayList splits in half recursively; LinkedList can't split efficiently.

2. **Fork**: Each chunk is submitted as a task to the common `ForkJoinPool`. By default, the pool has `Runtime.getRuntime().availableProcessors() - 1` threads.

3. **Process**: Each thread processes its chunk independently.

4. **Join**: Results are combined using the collector's combiner function.

Key considerations:
- The common ForkJoinPool is shared across the JVM. Blocking operations can starve other parallel streams.
- Use custom pool for isolation: `ForkJoinPool.commonPool().submit(() -> stream.parallel()...)`
- Spliterator characteristics affect optimization: SIZED, ORDERED, DISTINCT, etc.

**Q: How would you implement a custom Collector?**

A: A Collector has four components:

```java
Collector.of(
    // Supplier: Create mutable accumulator
    () -> new ArrayList<>(),
    
    // Accumulator: Add element to accumulator
    (list, element) -> list.add(element),
    
    // Combiner: Merge two accumulators (for parallel)
    (list1, list2) -> { list1.addAll(list2); return list1; },
    
    // Finisher: Transform accumulator to final result
    list -> Collections.unmodifiableList(list),
    
    // Characteristics: CONCURRENT, UNORDERED, IDENTITY_FINISH
    Collector.Characteristics.UNORDERED
);
```

Real example: Collecting to ImmutableList (Guava):
```java
public static <T> Collector<T, ?, ImmutableList<T>> toImmutableList() {
    return Collector.of(
        ImmutableList::builder,
        ImmutableList.Builder::add,
        (b1, b2) -> b1.addAll(b2.build()),
        ImmutableList.Builder::build
    );
}
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Java Streams are like an assembly line for data processing: items flow from a source through transformation stations (filter, map, sort) to a final destination (collect, reduce). The key insight is that streams are lazy. Nothing happens until a terminal operation triggers the pipeline. This enables optimizations like short-circuiting (`findFirst` stops early) and loop fusion (one pass through data). Use streams for declarative, composable data transformations on collections. Use parallel streams only for large, CPU-bound operations with no shared state. Avoid streams for simple loops, I/O operations, or when you need early exit. The power of streams isn't just shorter code. It's expressing WHAT you want, not HOW to do it, letting the runtime optimize execution.

