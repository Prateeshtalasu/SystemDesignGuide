# 🏗️ SOLID Principles in Java

---

## 0️⃣ Prerequisites

Before diving into SOLID principles, you need to understand:

- **OOP Fundamentals**: Classes, objects, inheritance, polymorphism, interfaces (covered in `01-oop-fundamentals.md`)
- **Coupling**: How much one class depends on another. Tight coupling means changes in one class force changes in another.
- **Cohesion**: How focused a class is on a single purpose. High cohesion means the class does one thing well.
- **Dependency**: When class A uses class B, A depends on B. If B changes, A might need to change.

If you understand that good code should be easy to change without breaking other parts, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Imagine you're working on a 3-year-old codebase. You need to add a simple feature: "Send SMS notifications in addition to email."

Without SOLID principles, you might find:

```java
// The nightmare scenario
public class OrderService {
    public void processOrder(Order order) {
        // 500 lines of code doing EVERYTHING:
        // - Validate order
        // - Calculate prices
        // - Apply discounts
        // - Check inventory
        // - Process payment
        // - Update database
        // - Send email notification  // <-- You need to change this
        // - Generate invoice
        // - Update analytics
        // - Log everything
        
        // To add SMS, you must:
        // 1. Understand all 500 lines
        // 2. Find where email is sent
        // 3. Add SMS code there
        // 4. Hope you don't break anything
        // 5. Test EVERYTHING because it's all tangled
    }
}
```

**Problems**:

1. **Rigid**: Hard to change without breaking things
2. **Fragile**: Changes cause unexpected bugs elsewhere
3. **Immobile**: Can't reuse code in other projects
4. **Viscous**: Doing things right is harder than doing them wrong

### What Systems Looked Like Before SOLID

In the 1990s-2000s, many codebases became "Big Ball of Mud":

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        BIG BALL OF MUD                                   │
│                                                                          │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                                                                   │  │
│   │    Everything connected to everything                            │  │
│   │                                                                   │  │
│   │         ┌───┐     ┌───┐     ┌───┐                               │  │
│   │         │ A │◄───►│ B │◄───►│ C │                               │  │
│   │         └─┬─┘     └─┬─┘     └─┬─┘                               │  │
│   │           │  ╲   ╱  │  ╲   ╱  │                                 │  │
│   │           │   ╲ ╱   │   ╲ ╱   │                                 │  │
│   │           │    ╳    │    ╳    │                                 │  │
│   │           │   ╱ ╲   │   ╱ ╲   │                                 │  │
│   │           │  ╱   ╲  │  ╱   ╲  │                                 │  │
│   │         ┌─▼─┐     ┌─▼─┐     ┌─▼─┐                               │  │
│   │         │ D │◄───►│ E │◄───►│ F │                               │  │
│   │         └───┘     └───┘     └───┘                               │  │
│   │                                                                   │  │
│   │    Change one thing → Everything might break                     │  │
│   │                                                                   │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### What Breaks Without SOLID

1. **Adding features takes weeks instead of hours**
2. **Bug fixes create new bugs**
3. **Testing requires the entire system**
4. **New developers take months to become productive**
5. **Technical debt compounds exponentially**

### Real Examples of the Problem

**Healthcare.gov (2013)**: The initial launch failed catastrophically. Post-mortem revealed tightly coupled code, no clear separation of concerns, and changes in one module breaking others.

**Knight Capital (2012)**: Lost $440 million in 45 minutes due to a deployment that activated old code. Tightly coupled systems made it impossible to safely deploy changes.

---

## 2️⃣ Intuition and Mental Model

### The LEGO Analogy

Think of SOLID principles like building with LEGO:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         THE LEGO ANALOGY                                 │
│                                                                          │
│   BAD Design (Glued together):        GOOD Design (SOLID):              │
│   ─────────────────────────           ────────────────────              │
│                                                                          │
│   ┌─────────────────────┐            ┌───┐ ┌───┐ ┌───┐                 │
│   │                     │            │ A │ │ B │ │ C │                 │
│   │   Everything glued  │            └─┬─┘ └─┬─┘ └─┬─┘                 │
│   │   into one piece    │              │     │     │                    │
│   │                     │              ▼     ▼     ▼                    │
│   │   Can't change      │            ┌───────────────┐                 │
│   │   Can't reuse       │            │   Standard    │                 │
│   │   Can't extend      │            │   Connectors  │                 │
│   │                     │            └───────────────┘                 │
│   └─────────────────────┘                                              │
│                                       Each piece:                       │
│                                       - Does one thing                  │
│                                       - Standard interface              │
│                                       - Replaceable                     │
│                                       - Reusable                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: SOLID principles help you build software like LEGO, small, focused pieces with standard interfaces that snap together and can be rearranged.

This analogy will be referenced throughout:

- **S**: Each LEGO piece does one thing (wheel, window, door)
- **O**: Add new pieces without modifying existing ones
- **L**: Any 2x4 brick works wherever a 2x4 brick is expected
- **I**: Don't force a wheel piece to have window features
- **D**: Pieces connect through standard studs, not custom glue

---

## 3️⃣ How It Works Internally

### The Five SOLID Principles

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SOLID PRINCIPLES                                 │
│                                                                          │
│   S - Single Responsibility Principle                                   │
│       "A class should have only one reason to change"                   │
│                                                                          │
│   O - Open/Closed Principle                                             │
│       "Open for extension, closed for modification"                     │
│                                                                          │
│   L - Liskov Substitution Principle                                     │
│       "Subtypes must be substitutable for their base types"             │
│                                                                          │
│   I - Interface Segregation Principle                                   │
│       "Many specific interfaces are better than one general interface"  │
│                                                                          │
│   D - Dependency Inversion Principle                                    │
│       "Depend on abstractions, not concretions"                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

Let's understand each one deeply with code examples.

---

## S - Single Responsibility Principle (SRP)

### Definition

**A class should have only one reason to change.**

Another way to say it: A class should have only one job, one responsibility, one axis of change.

### Why It Exists

When a class has multiple responsibilities, changes to one responsibility risk breaking the others. The class becomes a magnet for changes from multiple directions.

### The Problem (Violating SRP)

```java
// VIOLATION: This class has MULTIPLE reasons to change
public class Employee {
    private String name;
    private String email;
    private double salary;
    
    // Reason to change #1: Employee data structure changes
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    // Reason to change #2: Salary calculation rules change
    public double calculatePay() {
        // Tax rules change? This class changes.
        // Overtime rules change? This class changes.
        double basePay = salary / 12;
        double tax = basePay * 0.3;
        return basePay - tax;
    }
    
    // Reason to change #3: Report format changes
    public String generatePaySlip() {
        // PDF format changes? This class changes.
        // New fields needed? This class changes.
        return "Pay Slip for " + name + "\n" +
               "Net Pay: $" + calculatePay();
    }
    
    // Reason to change #4: Database schema changes
    public void saveToDatabase() {
        // Database changes? This class changes.
        // ORM changes? This class changes.
        String sql = "INSERT INTO employees VALUES (?, ?, ?)";
        // Execute SQL...
    }
    
    // Reason to change #5: Email format/provider changes
    public void sendPaySlipByEmail() {
        // Email provider changes? This class changes.
        // Email format changes? This class changes.
        String paySlip = generatePaySlip();
        // Send email with paySlip...
    }
}
```

**Problems**:

- Payroll team changes tax rules → Employee class changes
- DBA changes schema → Employee class changes
- UX team wants new report format → Employee class changes
- IT switches email providers → Employee class changes
- **Every team touches this class = merge conflicts, bugs, chaos**

### The Solution (Following SRP)

```java
// SOLUTION: Separate responsibilities into focused classes

// Responsibility 1: Employee data (Entity)
public class Employee {
    private String id;
    private String name;
    private String email;
    private double salary;
    
    public Employee(String id, String name, String email, double salary) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.salary = salary;
    }
    
    // Only getters and setters - pure data
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public double getSalary() { return salary; }
}

// Responsibility 2: Pay calculation (Business Logic)
public class PayrollCalculator {
    private static final double TAX_RATE = 0.3;
    private static final double MONTHS_PER_YEAR = 12;
    
    public double calculateMonthlyPay(Employee employee) {
        double basePay = employee.getSalary() / MONTHS_PER_YEAR;
        double tax = basePay * TAX_RATE;
        return basePay - tax;
    }
    
    public double calculateAnnualPay(Employee employee) {
        return calculateMonthlyPay(employee) * MONTHS_PER_YEAR;
    }
}

// Responsibility 3: Report generation (Presentation)
public class PaySlipGenerator {
    private final PayrollCalculator calculator;
    
    public PaySlipGenerator(PayrollCalculator calculator) {
        this.calculator = calculator;
    }
    
    public String generatePaySlip(Employee employee) {
        double netPay = calculator.calculateMonthlyPay(employee);
        return String.format("""
            ═══════════════════════════════════
            PAY SLIP
            ═══════════════════════════════════
            Employee: %s
            Employee ID: %s
            Net Pay: $%.2f
            ═══════════════════════════════════
            """, employee.getName(), employee.getId(), netPay);
    }
}

// Responsibility 4: Persistence (Data Access)
public class EmployeeRepository {
    public void save(Employee employee) {
        // Database logic here
    }
    
    public Employee findById(String id) {
        // Database logic here
        return null;
    }
}

// Responsibility 5: Notifications (Infrastructure)
public class EmailService {
    public void sendEmail(String to, String subject, String body) {
        // Email sending logic here
    }
}

// Coordinator that brings them together
public class PayrollService {
    private final EmployeeRepository repository;
    private final PayrollCalculator calculator;
    private final PaySlipGenerator generator;
    private final EmailService emailService;
    
    public PayrollService(EmployeeRepository repository,
                         PayrollCalculator calculator,
                         PaySlipGenerator generator,
                         EmailService emailService) {
        this.repository = repository;
        this.calculator = calculator;
        this.generator = generator;
        this.emailService = emailService;
    }
    
    public void processPayroll(String employeeId) {
        Employee employee = repository.findById(employeeId);
        String paySlip = generator.generatePaySlip(employee);
        emailService.sendEmail(
            employee.getEmail(),
            "Your Pay Slip",
            paySlip
        );
    }
}
```

**Benefits**:

- Tax rules change? Only `PayrollCalculator` changes
- New report format? Only `PaySlipGenerator` changes
- Switch to MongoDB? Only `EmployeeRepository` changes
- **Each class has ONE reason to change**

### How to Identify SRP Violations

Ask yourself:

1. "What does this class do?" If you use "AND", it might violate SRP.
   - Bad: "This class manages employee data AND calculates pay AND sends emails"
   - Good: "This class calculates employee pay"

2. "Who would request changes to this class?" If multiple stakeholders, it might violate SRP.
   - Payroll team, DBA, UX team all requesting changes = violation

3. "Can I describe the class in one sentence without 'and'?"

---

## O - Open/Closed Principle (OCP)

### Definition

**Software entities should be open for extension but closed for modification.**

You should be able to add new functionality without changing existing code.

### Why It Exists

Every time you modify existing code:

- You risk breaking what already works
- You need to retest everything
- You might introduce bugs in stable code

### The Problem (Violating OCP)

```java
// VIOLATION: Adding new shapes requires modifying this class
public class AreaCalculator {
    
    public double calculateArea(Object shape) {
        if (shape instanceof Rectangle) {
            Rectangle r = (Rectangle) shape;
            return r.getWidth() * r.getHeight();
        } 
        else if (shape instanceof Circle) {
            Circle c = (Circle) shape;
            return Math.PI * c.getRadius() * c.getRadius();
        }
        else if (shape instanceof Triangle) {
            // Added later - had to MODIFY this class
            Triangle t = (Triangle) shape;
            return 0.5 * t.getBase() * t.getHeight();
        }
        // What about Pentagon? Hexagon? Ellipse?
        // Every new shape = modify this class = risk breaking existing shapes
        
        throw new IllegalArgumentException("Unknown shape");
    }
}
```

**Problems**:

- Adding Pentagon requires modifying AreaCalculator
- Must retest Rectangle and Circle calculations
- The if-else chain grows forever
- Violates SRP too (knows about ALL shapes)

### The Solution (Following OCP)

```java
// SOLUTION: Use abstraction to allow extension without modification

// Define the contract
public interface Shape {
    double calculateArea();
}

// Each shape implements its own area calculation
public class Rectangle implements Shape {
    private final double width;
    private final double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double calculateArea() {
        return width * height;
    }
}

public class Circle implements Shape {
    private final double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

// Adding new shape - NO modification to existing code!
public class Triangle implements Shape {
    private final double base;
    private final double height;
    
    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }
    
    @Override
    public double calculateArea() {
        return 0.5 * base * height;
    }
}

// AreaCalculator is now CLOSED for modification
// but OPEN for extension (works with any Shape)
public class AreaCalculator {
    
    public double calculateTotalArea(List<Shape> shapes) {
        return shapes.stream()
                     .mapToDouble(Shape::calculateArea)
                     .sum();
    }
}
```

**Benefits**:

- Add Pentagon? Create new class, implement Shape. Done.
- AreaCalculator never changes
- Existing shapes never retested
- New shapes can be added by different teams

### Real-World Example: Payment Processing

```java
// VIOLATION: Adding new payment method requires modification
public class PaymentProcessor {
    public void processPayment(String type, double amount) {
        if (type.equals("CREDIT_CARD")) {
            // Credit card processing
        } else if (type.equals("PAYPAL")) {
            // PayPal processing
        } else if (type.equals("CRYPTO")) {
            // Added later - modified existing class!
        }
    }
}

// SOLUTION: Open for extension, closed for modification
public interface PaymentMethod {
    void processPayment(double amount);
    boolean supports(String type);
}

public class CreditCardPayment implements PaymentMethod {
    @Override
    public void processPayment(double amount) {
        // Connect to credit card processor
    }
    
    @Override
    public boolean supports(String type) {
        return "CREDIT_CARD".equals(type);
    }
}

public class PayPalPayment implements PaymentMethod {
    @Override
    public void processPayment(double amount) {
        // Connect to PayPal API
    }
    
    @Override
    public boolean supports(String type) {
        return "PAYPAL".equals(type);
    }
}

// Adding crypto - no modification to existing code!
public class CryptoPayment implements PaymentMethod {
    @Override
    public void processPayment(double amount) {
        // Connect to crypto exchange
    }
    
    @Override
    public boolean supports(String type) {
        return "CRYPTO".equals(type);
    }
}

// Processor is closed for modification
public class PaymentProcessor {
    private final List<PaymentMethod> paymentMethods;
    
    public PaymentProcessor(List<PaymentMethod> paymentMethods) {
        this.paymentMethods = paymentMethods;
    }
    
    public void processPayment(String type, double amount) {
        PaymentMethod method = paymentMethods.stream()
            .filter(pm -> pm.supports(type))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Unknown payment type: " + type));
        
        method.processPayment(amount);
    }
}
```

---

## L - Liskov Substitution Principle (LSP)

### Definition

**Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.**

If S is a subtype of T, then objects of type T can be replaced with objects of type S without altering the correctness of the program.

### Why It Exists

Inheritance should represent "is-a" relationships that actually work in code. If a subclass can't be used wherever the parent is expected, the inheritance is wrong.

### The Problem (Violating LSP)

The classic example: Square extending Rectangle.

```java
// VIOLATION: Square cannot substitute for Rectangle

public class Rectangle {
    protected int width;
    protected int height;
    
    public void setWidth(int width) {
        this.width = width;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
    
    public int getArea() {
        return width * height;
    }
}

// Mathematically, a square IS-A rectangle
// But in code, this breaks LSP!
public class Square extends Rectangle {
    
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // Must keep square!
    }
    
    @Override
    public void setHeight(int height) {
        this.width = height;  // Must keep square!
        this.height = height;
    }
}

// This code works for Rectangle but BREAKS for Square
public class AreaCalculatorTest {
    
    public void testRectangleArea(Rectangle rect) {
        rect.setWidth(5);
        rect.setHeight(4);
        
        // For Rectangle: 5 * 4 = 20 ✓
        // For Square: setHeight(4) also sets width to 4, so 4 * 4 = 16 ✗
        assert rect.getArea() == 20;  // FAILS for Square!
    }
}
```

**The problem**: Code that works with Rectangle breaks when given a Square. Square is NOT substitutable for Rectangle, violating LSP.

### The Solution (Following LSP)

```java
// SOLUTION: Don't use inheritance for this relationship

// Common interface for shapes with area
public interface Shape {
    int getArea();
}

// Rectangle is a shape
public class Rectangle implements Shape {
    private final int width;
    private final int height;
    
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    public int getWidth() { return width; }
    public int getHeight() { return height; }
    
    @Override
    public int getArea() {
        return width * height;
    }
}

// Square is a shape (not a Rectangle!)
public class Square implements Shape {
    private final int side;
    
    public Square(int side) {
        this.side = side;
    }
    
    public int getSide() { return side; }
    
    @Override
    public int getArea() {
        return side * side;
    }
}

// Now code that uses Shape works with both
public class AreaCalculator {
    public int calculateTotalArea(List<Shape> shapes) {
        return shapes.stream()
                     .mapToInt(Shape::getArea)
                     .sum();
    }
}
```

### Real-World Example: Bird Hierarchy

```java
// VIOLATION: Not all birds can fly!

public class Bird {
    public void fly() {
        System.out.println("Flying high!");
    }
}

public class Sparrow extends Bird {
    // Inherits fly() - works fine
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
}

// This breaks!
public class BirdMigration {
    public void migrate(List<Bird> birds) {
        for (Bird bird : birds) {
            bird.fly();  // Throws exception for Penguin!
        }
    }
}
```

```java
// SOLUTION: Separate flying capability

public interface Bird {
    void eat();
    void sleep();
}

public interface FlyingBird extends Bird {
    void fly();
}

public interface SwimmingBird extends Bird {
    void swim();
}

public class Sparrow implements FlyingBird {
    @Override
    public void eat() { /* ... */ }
    
    @Override
    public void sleep() { /* ... */ }
    
    @Override
    public void fly() {
        System.out.println("Sparrow flying!");
    }
}

public class Penguin implements SwimmingBird {
    @Override
    public void eat() { /* ... */ }
    
    @Override
    public void sleep() { /* ... */ }
    
    @Override
    public void swim() {
        System.out.println("Penguin swimming!");
    }
}

// Now migration only accepts birds that CAN fly
public class BirdMigration {
    public void migrate(List<FlyingBird> birds) {
        for (FlyingBird bird : birds) {
            bird.fly();  // Safe - all FlyingBirds can fly
        }
    }
}
```

### LSP Rules to Follow

1. **Preconditions cannot be strengthened**: Subclass can't require MORE than parent
2. **Postconditions cannot be weakened**: Subclass must deliver AT LEAST what parent promises
3. **Invariants must be preserved**: Subclass must maintain parent's rules
4. **No new exceptions**: Subclass shouldn't throw exceptions parent doesn't

---

## I - Interface Segregation Principle (ISP)

### Definition

**Clients should not be forced to depend on interfaces they don't use.**

Many specific interfaces are better than one general-purpose interface.

### Why It Exists

Fat interfaces force classes to implement methods they don't need. This creates:

- Empty or throwing implementations
- Unnecessary dependencies
- Harder testing

### The Problem (Violating ISP)

```java
// VIOLATION: Fat interface forces unnecessary implementations

public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeReport();
    void reviewCode();
}

// Human worker - implements all
public class Developer implements Worker {
    @Override
    public void work() { /* coding */ }
    
    @Override
    public void eat() { /* lunch break */ }
    
    @Override
    public void sleep() { /* go home and sleep */ }
    
    @Override
    public void attendMeeting() { /* daily standup */ }
    
    @Override
    public void writeReport() { /* sprint report */ }
    
    @Override
    public void reviewCode() { /* PR reviews */ }
}

// Robot worker - forced to implement things it can't do!
public class RobotWorker implements Worker {
    @Override
    public void work() { /* automated tasks */ }
    
    @Override
    public void eat() {
        throw new UnsupportedOperationException("Robots don't eat!");
    }
    
    @Override
    public void sleep() {
        throw new UnsupportedOperationException("Robots don't sleep!");
    }
    
    @Override
    public void attendMeeting() {
        throw new UnsupportedOperationException("Robots can't attend meetings!");
    }
    
    @Override
    public void writeReport() { /* generate report */ }
    
    @Override
    public void reviewCode() {
        throw new UnsupportedOperationException("Robots can't review code!");
    }
}
```

**Problems**:

- RobotWorker has 4 methods that throw exceptions
- Any code using Worker might call methods that don't work
- Testing requires mocking unused methods
- Interface changes affect all implementers

### The Solution (Following ISP)

```java
// SOLUTION: Segregate into focused interfaces

public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public interface MeetingAttendee {
    void attendMeeting();
}

public interface ReportWriter {
    void writeReport();
}

public interface CodeReviewer {
    void reviewCode();
}

// Human developer implements what humans do
public class Developer implements Workable, Eatable, Sleepable, 
                                 MeetingAttendee, ReportWriter, CodeReviewer {
    @Override
    public void work() { /* coding */ }
    
    @Override
    public void eat() { /* lunch */ }
    
    @Override
    public void sleep() { /* rest */ }
    
    @Override
    public void attendMeeting() { /* standup */ }
    
    @Override
    public void writeReport() { /* report */ }
    
    @Override
    public void reviewCode() { /* PR review */ }
}

// Robot only implements what robots can do
public class RobotWorker implements Workable, ReportWriter {
    @Override
    public void work() { /* automated tasks */ }
    
    @Override
    public void writeReport() { /* generate report */ }
    
    // No need to implement eat(), sleep(), etc.!
}

// Services depend only on what they need
public class WorkScheduler {
    public void scheduleWork(List<Workable> workers) {
        workers.forEach(Workable::work);  // Works for humans AND robots
    }
}

public class LunchScheduler {
    public void scheduleLunch(List<Eatable> eaters) {
        eaters.forEach(Eatable::eat);  // Only humans, robots not included
    }
}
```

### Real-World Example: Repository Pattern

```java
// VIOLATION: Fat repository interface

public interface Repository<T> {
    T findById(Long id);
    List<T> findAll();
    T save(T entity);
    void delete(T entity);
    void deleteById(Long id);
    List<T> findByField(String field, Object value);
    Page<T> findAll(Pageable pageable);
    List<T> findAllSorted(Sort sort);
    long count();
    boolean existsById(Long id);
    void flush();
    T saveAndFlush(T entity);
    void deleteAllInBatch();
    // 20 more methods...
}

// Simple service only needs read operations
public class ReportService {
    private final Repository<Order> orderRepository;
    
    // Depends on 25 methods but only uses 2!
    public Report generateReport() {
        List<Order> orders = orderRepository.findAll();
        long count = orderRepository.count();
        // ...
    }
}
```

```java
// SOLUTION: Segregated repository interfaces

public interface ReadRepository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    long count();
    boolean existsById(ID id);
}

public interface WriteRepository<T, ID> {
    T save(T entity);
    void delete(T entity);
    void deleteById(ID id);
}

public interface PagingRepository<T, ID> extends ReadRepository<T, ID> {
    Page<T> findAll(Pageable pageable);
    List<T> findAll(Sort sort);
}

// Full repository composes all interfaces
public interface CrudRepository<T, ID> extends ReadRepository<T, ID>, WriteRepository<T, ID> {
}

// Report service only depends on what it needs
public class ReportService {
    private final ReadRepository<Order, Long> orderRepository;
    
    public Report generateReport() {
        List<Order> orders = orderRepository.findAll();
        long count = orderRepository.count();
        // ...
    }
}

// Admin service needs write operations
public class AdminService {
    private final CrudRepository<Order, Long> orderRepository;
    
    public void deleteOldOrders() {
        // Can read AND write
    }
}
```

---

## D - Dependency Inversion Principle (DIP)

### Definition

**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

**Abstractions should not depend on details. Details should depend on abstractions.**

### Why It Exists

When high-level business logic depends directly on low-level implementation details:

- Changes in low-level code break high-level code
- Can't swap implementations (e.g., different databases)
- Testing requires real dependencies
- Tight coupling makes reuse impossible

### The Problem (Violating DIP)

```java
// VIOLATION: High-level depends on low-level

// Low-level module (implementation detail)
public class MySQLDatabase {
    public void save(String data) {
        // MySQL-specific code
        System.out.println("Saving to MySQL: " + data);
    }
    
    public String query(String sql) {
        // MySQL-specific code
        return "MySQL result";
    }
}

// Low-level module (implementation detail)
public class SmtpEmailSender {
    public void send(String to, String message) {
        // SMTP-specific code
        System.out.println("Sending via SMTP to " + to);
    }
}

// High-level module (business logic)
// PROBLEM: Directly depends on MySQL and SMTP!
public class OrderService {
    private MySQLDatabase database = new MySQLDatabase();  // Tight coupling!
    private SmtpEmailSender emailSender = new SmtpEmailSender();  // Tight coupling!
    
    public void createOrder(Order order) {
        // Business logic
        database.save(order.toString());  // Tied to MySQL
        emailSender.send(order.getCustomerEmail(), "Order created!");  // Tied to SMTP
    }
}
```

**Problems**:

- Can't switch to PostgreSQL without changing OrderService
- Can't switch to SendGrid without changing OrderService
- Can't unit test without real MySQL and SMTP
- OrderService knows too much about implementation details

### The Solution (Following DIP)

```java
// SOLUTION: Depend on abstractions

// Abstraction for data storage
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String id);
}

// Abstraction for notifications
public interface NotificationService {
    void sendOrderConfirmation(Order order);
}

// Low-level implementation #1: MySQL
public class MySQLOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) {
        // MySQL-specific code
        System.out.println("Saving to MySQL: " + order);
    }
    
    @Override
    public Optional<Order> findById(String id) {
        // MySQL-specific code
        return Optional.empty();
    }
}

// Low-level implementation #2: PostgreSQL
public class PostgreSQLOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) {
        // PostgreSQL-specific code
        System.out.println("Saving to PostgreSQL: " + order);
    }
    
    @Override
    public Optional<Order> findById(String id) {
        // PostgreSQL-specific code
        return Optional.empty();
    }
}

// Low-level implementation: Email
public class EmailNotificationService implements NotificationService {
    @Override
    public void sendOrderConfirmation(Order order) {
        System.out.println("Sending email to " + order.getCustomerEmail());
    }
}

// Low-level implementation: SMS
public class SmsNotificationService implements NotificationService {
    @Override
    public void sendOrderConfirmation(Order order) {
        System.out.println("Sending SMS to " + order.getCustomerPhone());
    }
}

// High-level module: depends on ABSTRACTIONS, not implementations
public class OrderService {
    private final OrderRepository repository;  // Interface!
    private final NotificationService notificationService;  // Interface!
    
    // Dependencies injected through constructor
    public OrderService(OrderRepository repository, 
                       NotificationService notificationService) {
        this.repository = repository;
        this.notificationService = notificationService;
    }
    
    public void createOrder(Order order) {
        // Business logic - doesn't know or care about MySQL vs PostgreSQL
        repository.save(order);
        notificationService.sendOrderConfirmation(order);
    }
}

// Usage - swap implementations easily
public class Application {
    public static void main(String[] args) {
        // Production: MySQL + Email
        OrderService prodService = new OrderService(
            new MySQLOrderRepository(),
            new EmailNotificationService()
        );
        
        // Development: PostgreSQL + SMS
        OrderService devService = new OrderService(
            new PostgreSQLOrderRepository(),
            new SmsNotificationService()
        );
        
        // Testing: Mock implementations
        OrderService testService = new OrderService(
            new MockOrderRepository(),  // In-memory for tests
            new MockNotificationService()  // No-op for tests
        );
    }
}
```

### The Dependency Inversion Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WITHOUT DIP (Wrong)                                   │
│                                                                          │
│   High-Level                                                            │
│   ┌──────────────────┐                                                  │
│   │   OrderService   │                                                  │
│   └────────┬─────────┘                                                  │
│            │ depends on                                                  │
│            ▼                                                             │
│   Low-Level                                                             │
│   ┌──────────────────┐  ┌──────────────────┐                           │
│   │  MySQLDatabase   │  │  SmtpEmailSender │                           │
│   └──────────────────┘  └──────────────────┘                           │
│                                                                          │
│   High-level directly depends on low-level = TIGHT COUPLING            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    WITH DIP (Correct)                                    │
│                                                                          │
│   High-Level                                                            │
│   ┌──────────────────┐                                                  │
│   │   OrderService   │                                                  │
│   └────────┬─────────┘                                                  │
│            │ depends on                                                  │
│            ▼                                                             │
│   Abstractions                                                          │
│   ┌──────────────────┐  ┌──────────────────┐                           │
│   │ OrderRepository  │  │NotificationService│  ← Interfaces            │
│   └────────▲─────────┘  └────────▲─────────┘                           │
│            │ implements          │ implements                           │
│            │                     │                                       │
│   Low-Level                                                             │
│   ┌──────────────────┐  ┌──────────────────┐                           │
│   │MySQLOrderRepo    │  │EmailNotification │                           │
│   │PostgreSQLOrderRepo│  │SmsNotification   │                           │
│   └──────────────────┘  └──────────────────┘                           │
│                                                                          │
│   Both high-level and low-level depend on abstractions                  │
│   Arrows point INWARD toward abstractions = INVERTED                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

Let's trace how SOLID principles work together in a real scenario.

### Scenario: Building a Notification System

**Requirements**:

1. Send notifications when orders are placed
2. Support email, SMS, and push notifications
3. Different customers prefer different channels
4. Must be testable and extensible

### Without SOLID (The Mess)

```java
public class OrderProcessor {
    public void processOrder(Order order) {
        // Validate
        if (order.getItems().isEmpty()) {
            throw new RuntimeException("Empty order");
        }
        
        // Save to database (hardcoded MySQL)
        Connection conn = DriverManager.getConnection("jdbc:mysql://...");
        PreparedStatement stmt = conn.prepareStatement("INSERT...");
        // ... save order
        
        // Send notification (all types hardcoded)
        Customer customer = order.getCustomer();
        
        if (customer.getPreference().equals("EMAIL")) {
            // SMTP code here
            Properties props = new Properties();
            props.put("mail.smtp.host", "smtp.gmail.com");
            Session session = Session.getInstance(props);
            MimeMessage message = new MimeMessage(session);
            // ... 30 lines of email code
        } 
        else if (customer.getPreference().equals("SMS")) {
            // Twilio code here
            Twilio.init("account", "token");
            Message.creator(new PhoneNumber(customer.getPhone()), 
                           new PhoneNumber("+1234567890"),
                           "Order placed!").create();
        }
        else if (customer.getPreference().equals("PUSH")) {
            // Firebase code here
            FirebaseMessaging.getInstance()
                .send(Message.builder()
                    .setToken(customer.getDeviceToken())
                    .build());
        }
        
        // Log (hardcoded file logging)
        FileWriter fw = new FileWriter("orders.log", true);
        fw.write("Order processed: " + order.getId());
        fw.close();
    }
}
```

**Problems**:

- 200+ lines in one method
- Can't test without real MySQL, SMTP, Twilio, Firebase
- Adding Slack notification = modify this class
- Database change = modify this class
- Everything breaks everything

### With SOLID (Clean Architecture)

```java
// === STEP 1: Define Abstractions (DIP) ===

// Notification abstraction
public interface NotificationChannel {
    void send(Customer customer, String message);
    boolean supports(String preference);
}

// Repository abstraction
public interface OrderRepository {
    void save(Order order);
}

// Logger abstraction
public interface OrderLogger {
    void log(String message);
}

// === STEP 2: Implement Each Channel (SRP + OCP) ===

// Each class has ONE responsibility
public class EmailChannel implements NotificationChannel {
    private final EmailClient emailClient;
    
    public EmailChannel(EmailClient emailClient) {
        this.emailClient = emailClient;
    }
    
    @Override
    public void send(Customer customer, String message) {
        emailClient.send(customer.getEmail(), "Order Update", message);
    }
    
    @Override
    public boolean supports(String preference) {
        return "EMAIL".equals(preference);
    }
}

public class SmsChannel implements NotificationChannel {
    private final SmsClient smsClient;
    
    public SmsChannel(SmsClient smsClient) {
        this.smsClient = smsClient;
    }
    
    @Override
    public void send(Customer customer, String message) {
        smsClient.send(customer.getPhone(), message);
    }
    
    @Override
    public boolean supports(String preference) {
        return "SMS".equals(preference);
    }
}

public class PushChannel implements NotificationChannel {
    private final PushClient pushClient;
    
    public PushChannel(PushClient pushClient) {
        this.pushClient = pushClient;
    }
    
    @Override
    public void send(Customer customer, String message) {
        pushClient.send(customer.getDeviceToken(), message);
    }
    
    @Override
    public boolean supports(String preference) {
        return "PUSH".equals(preference);
    }
}

// === STEP 3: Notification Service (SRP) ===

public class NotificationService {
    private final List<NotificationChannel> channels;
    
    public NotificationService(List<NotificationChannel> channels) {
        this.channels = channels;
    }
    
    public void notifyCustomer(Customer customer, String message) {
        channels.stream()
            .filter(ch -> ch.supports(customer.getPreference()))
            .findFirst()
            .ifPresent(ch -> ch.send(customer, message));
    }
}

// === STEP 4: Order Validator (SRP) ===

public class OrderValidator {
    public void validate(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("Order cannot be null");
        }
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have items");
        }
        if (order.getCustomer() == null) {
            throw new IllegalArgumentException("Order must have customer");
        }
    }
}

// === STEP 5: Order Processor (Orchestration) ===

public class OrderProcessor {
    private final OrderValidator validator;
    private final OrderRepository repository;
    private final NotificationService notificationService;
    private final OrderLogger logger;
    
    // Constructor injection (DIP)
    public OrderProcessor(OrderValidator validator,
                         OrderRepository repository,
                         NotificationService notificationService,
                         OrderLogger logger) {
        this.validator = validator;
        this.repository = repository;
        this.notificationService = notificationService;
        this.logger = logger;
    }
    
    public void processOrder(Order order) {
        // Each step is delegated to a focused class
        validator.validate(order);
        repository.save(order);
        notificationService.notifyCustomer(
            order.getCustomer(), 
            "Your order " + order.getId() + " has been placed!"
        );
        logger.log("Order processed: " + order.getId());
    }
}

// === STEP 6: Easy Testing ===

public class OrderProcessorTest {
    
    @Test
    void shouldProcessValidOrder() {
        // Create mock implementations
        OrderValidator validator = new OrderValidator();
        OrderRepository mockRepo = mock(OrderRepository.class);
        NotificationService mockNotification = mock(NotificationService.class);
        OrderLogger mockLogger = mock(OrderLogger.class);
        
        // Inject mocks
        OrderProcessor processor = new OrderProcessor(
            validator, mockRepo, mockNotification, mockLogger
        );
        
        // Test
        Order order = createTestOrder();
        processor.processOrder(order);
        
        // Verify
        verify(mockRepo).save(order);
        verify(mockNotification).notifyCustomer(any(), anyString());
        verify(mockLogger).log(anyString());
    }
}

// === STEP 7: Adding Slack (OCP - No modification!) ===

// Just add a new class - existing code unchanged!
public class SlackChannel implements NotificationChannel {
    private final SlackClient slackClient;
    
    public SlackChannel(SlackClient slackClient) {
        this.slackClient = slackClient;
    }
    
    @Override
    public void send(Customer customer, String message) {
        slackClient.postMessage(customer.getSlackId(), message);
    }
    
    @Override
    public boolean supports(String preference) {
        return "SLACK".equals(preference);
    }
}

// Configuration adds it to the list - that's it!
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Netflix's Microservices**:

Netflix applies SOLID at the service level:

```java
// Each microservice has single responsibility
// - User Service: user data only
// - Recommendation Service: recommendations only
// - Streaming Service: video delivery only

// Services depend on abstractions (interfaces/contracts)
public interface RecommendationClient {
    List<Movie> getRecommendations(String userId);
}

// Multiple implementations possible
public class MLRecommendationClient implements RecommendationClient { }
public class RuleBasedRecommendationClient implements RecommendationClient { }
public class FallbackRecommendationClient implements RecommendationClient { }
```

**Spring Framework**:

Spring is built on SOLID principles:

```java
// DIP: Depend on abstractions
@Service
public class UserService {
    private final UserRepository userRepository;  // Interface
    
    @Autowired  // Spring injects implementation
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// OCP: Add new functionality via configuration
@Configuration
public class SecurityConfig {
    @Bean
    @ConditionalOnProperty(name = "auth.type", havingValue = "oauth")
    public AuthenticationProvider oauthProvider() {
        return new OAuthProvider();
    }
    
    @Bean
    @ConditionalOnProperty(name = "auth.type", havingValue = "ldap")
    public AuthenticationProvider ldapProvider() {
        return new LdapProvider();
    }
}
```

### Real Workflows and Tooling

**Dependency Injection Frameworks**:

| Framework     | Language | DIP Support                    |
| ------------- | -------- | ------------------------------ |
| Spring        | Java     | @Autowired, @Inject            |
| Guice         | Java     | @Inject, Modules               |
| Dagger        | Java     | Compile-time DI                |
| .NET Core DI  | C#       | IServiceCollection             |
| Angular       | TS       | Providers, Injectors           |

**Code Analysis Tools**:

```bash
# SonarQube rules for SOLID violations
sonar.issue.ignore.multicriteria=e1,e2

# ArchUnit for architecture tests
@Test
void servicesShouldNotDependOnControllers() {
    noClasses()
        .that().resideInAPackage("..service..")
        .should().dependOnClassesThat().resideInAPackage("..controller..")
        .check(importedClasses);
}
```

### Production War Stories

**The Monolithic Payment Processor**:

A fintech company had a `PaymentService` class with 5,000 lines:

```java
public class PaymentService {
    public void processPayment(Payment payment) {
        // Validation (500 lines)
        // Fraud detection (800 lines)
        // Payment routing (600 lines)
        // Transaction logging (400 lines)
        // Notification (300 lines)
        // Reconciliation (500 lines)
        // Error handling (900 lines)
        // Retry logic (500 lines)
        // Metrics (500 lines)
    }
}
```

**Problems**:

- 45-minute build times (everything recompiled)
- 2-week testing cycles (couldn't isolate tests)
- 3 teams blocked on each other
- Average bug fix introduced 2 new bugs

**Solution** (Applied SOLID):

```java
// Separated into focused services
PaymentValidator validator;
FraudDetector fraudDetector;
PaymentRouter router;
TransactionLogger logger;
NotificationService notifier;
ReconciliationService reconciler;
RetryHandler retryHandler;
MetricsCollector metrics;

// Orchestrator brings them together
public class PaymentOrchestrator {
    // Inject all dependencies
    
    public void processPayment(Payment payment) {
        validator.validate(payment);
        fraudDetector.check(payment);
        router.route(payment);
        // etc.
    }
}
```

**Results**:

- Build time: 45 min → 3 min
- Test isolation: Each service tested independently
- Team autonomy: Teams own their services
- Bug rate: 70% reduction

---

## 6️⃣ How to Implement: Complete Example

Let's build a complete order processing system following all SOLID principles.

### Project Structure

```
src/main/java/com/example/orders/
├── domain/
│   ├── Order.java
│   ├── OrderItem.java
│   └── Customer.java
├── validation/
│   ├── OrderValidator.java
│   └── ValidationResult.java
├── repository/
│   ├── OrderRepository.java
│   └── JpaOrderRepository.java
├── notification/
│   ├── NotificationChannel.java
│   ├── EmailChannel.java
│   ├── SmsChannel.java
│   └── NotificationService.java
├── pricing/
│   ├── PricingStrategy.java
│   ├── StandardPricing.java
│   └── DiscountPricing.java
├── service/
│   └── OrderService.java
└── config/
    └── OrderConfig.java
```

### Domain Classes

```java
// Order.java
package com.example.orders.domain;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Order entity - follows SRP (only holds order data)
 */
public class Order {
    private final String id;
    private final Customer customer;
    private final List<OrderItem> items;
    private final LocalDateTime createdAt;
    private OrderStatus status;
    
    public Order(String id, Customer customer) {
        this.id = id;
        this.customer = customer;
        this.items = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
        this.status = OrderStatus.PENDING;
    }
    
    public void addItem(OrderItem item) {
        items.add(item);
    }
    
    public String getId() { return id; }
    public Customer getCustomer() { return customer; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public OrderStatus getStatus() { return status; }
    
    // Return unmodifiable list to protect encapsulation
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
    
    public void setStatus(OrderStatus status) {
        this.status = status;
    }
    
    public BigDecimal getSubtotal() {
        return items.stream()
            .map(OrderItem::getTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// OrderItem.java
public record OrderItem(
    String productId,
    String productName,
    int quantity,
    BigDecimal unitPrice
) {
    public BigDecimal getTotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}

// Customer.java
public record Customer(
    String id,
    String name,
    String email,
    String phone,
    NotificationPreference notificationPreference
) {}

// OrderStatus.java
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}

// NotificationPreference.java
public enum NotificationPreference {
    EMAIL, SMS, BOTH, NONE
}
```

### Validation (SRP)

```java
// ValidationResult.java
package com.example.orders.validation;

import java.util.ArrayList;
import java.util.List;

public class ValidationResult {
    private final List<String> errors = new ArrayList<>();
    
    public void addError(String error) {
        errors.add(error);
    }
    
    public boolean isValid() {
        return errors.isEmpty();
    }
    
    public List<String> getErrors() {
        return List.copyOf(errors);
    }
}

// OrderValidator.java
package com.example.orders.validation;

import com.example.orders.domain.Order;

/**
 * Single Responsibility: Only validates orders
 */
public class OrderValidator {
    
    private static final int MAX_ITEMS = 100;
    
    public ValidationResult validate(Order order) {
        ValidationResult result = new ValidationResult();
        
        if (order == null) {
            result.addError("Order cannot be null");
            return result;
        }
        
        if (order.getCustomer() == null) {
            result.addError("Customer is required");
        }
        
        if (order.getItems().isEmpty()) {
            result.addError("Order must contain at least one item");
        }
        
        if (order.getItems().size() > MAX_ITEMS) {
            result.addError("Order cannot contain more than " + MAX_ITEMS + " items");
        }
        
        // Validate each item
        order.getItems().forEach(item -> {
            if (item.quantity() <= 0) {
                result.addError("Item quantity must be positive: " + item.productName());
            }
            if (item.unitPrice().compareTo(java.math.BigDecimal.ZERO) <= 0) {
                result.addError("Item price must be positive: " + item.productName());
            }
        });
        
        return result;
    }
}
```

### Repository (DIP)

```java
// OrderRepository.java - Abstraction
package com.example.orders.repository;

import com.example.orders.domain.Order;
import java.util.Optional;
import java.util.List;

/**
 * Abstraction for order persistence.
 * High-level code depends on this interface, not implementations.
 */
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String id);
    List<Order> findByCustomerId(String customerId);
}

// JpaOrderRepository.java - Implementation
package com.example.orders.repository;

import com.example.orders.domain.Order;
import org.springframework.stereotype.Repository;
import java.util.*;

/**
 * JPA implementation of OrderRepository.
 * Can be swapped for MongoOrderRepository, etc.
 */
@Repository
public class JpaOrderRepository implements OrderRepository {
    
    // In real app, would use EntityManager or Spring Data JPA
    private final Map<String, Order> storage = new HashMap<>();
    
    @Override
    public void save(Order order) {
        storage.put(order.getId(), order);
    }
    
    @Override
    public Optional<Order> findById(String id) {
        return Optional.ofNullable(storage.get(id));
    }
    
    @Override
    public List<Order> findByCustomerId(String customerId) {
        return storage.values().stream()
            .filter(o -> o.getCustomer().id().equals(customerId))
            .toList();
    }
}
```

### Notification System (OCP + ISP + DIP)

```java
// NotificationChannel.java - Abstraction (ISP: focused interface)
package com.example.orders.notification;

import com.example.orders.domain.Customer;
import com.example.orders.domain.Order;

/**
 * Interface Segregation: Only notification-related methods.
 * Open/Closed: New channels implement this without modifying existing code.
 */
public interface NotificationChannel {
    void sendOrderConfirmation(Customer customer, Order order);
    boolean supports(Customer customer);
}

// EmailChannel.java
package com.example.orders.notification;

import com.example.orders.domain.*;
import org.springframework.stereotype.Component;

@Component
public class EmailChannel implements NotificationChannel {
    
    @Override
    public void sendOrderConfirmation(Customer customer, Order order) {
        String subject = "Order Confirmation - " + order.getId();
        String body = buildEmailBody(customer, order);
        
        // In real app, use JavaMailSender or SendGrid
        System.out.println("Sending email to: " + customer.email());
        System.out.println("Subject: " + subject);
        System.out.println("Body: " + body);
    }
    
    @Override
    public boolean supports(Customer customer) {
        NotificationPreference pref = customer.notificationPreference();
        return pref == NotificationPreference.EMAIL || 
               pref == NotificationPreference.BOTH;
    }
    
    private String buildEmailBody(Customer customer, Order order) {
        return String.format("""
            Dear %s,
            
            Thank you for your order!
            
            Order ID: %s
            Items: %d
            Total: $%.2f
            
            Best regards,
            The Team
            """, 
            customer.name(),
            order.getId(),
            order.getItems().size(),
            order.getSubtotal()
        );
    }
}

// SmsChannel.java
package com.example.orders.notification;

import com.example.orders.domain.*;
import org.springframework.stereotype.Component;

@Component
public class SmsChannel implements NotificationChannel {
    
    @Override
    public void sendOrderConfirmation(Customer customer, Order order) {
        String message = String.format(
            "Order %s confirmed! Total: $%.2f",
            order.getId(),
            order.getSubtotal()
        );
        
        // In real app, use Twilio
        System.out.println("Sending SMS to: " + customer.phone());
        System.out.println("Message: " + message);
    }
    
    @Override
    public boolean supports(Customer customer) {
        NotificationPreference pref = customer.notificationPreference();
        return pref == NotificationPreference.SMS || 
               pref == NotificationPreference.BOTH;
    }
}

// NotificationService.java
package com.example.orders.notification;

import com.example.orders.domain.*;
import org.springframework.stereotype.Service;
import java.util.List;

/**
 * Coordinates notification channels.
 * Open/Closed: Adding new channel doesn't require modification here.
 */
@Service
public class NotificationService {
    
    private final List<NotificationChannel> channels;
    
    // DIP: Depends on abstraction (List of interfaces)
    public NotificationService(List<NotificationChannel> channels) {
        this.channels = channels;
    }
    
    public void notifyOrderConfirmation(Order order) {
        Customer customer = order.getCustomer();
        
        if (customer.notificationPreference() == NotificationPreference.NONE) {
            return;
        }
        
        channels.stream()
            .filter(channel -> channel.supports(customer))
            .forEach(channel -> channel.sendOrderConfirmation(customer, order));
    }
}
```

### Pricing Strategy (OCP + LSP)

```java
// PricingStrategy.java - Abstraction
package com.example.orders.pricing;

import com.example.orders.domain.Order;
import java.math.BigDecimal;

/**
 * Strategy pattern for pricing.
 * Open/Closed: New pricing strategies don't modify existing code.
 * Liskov: All strategies are interchangeable.
 */
public interface PricingStrategy {
    BigDecimal calculateTotal(Order order);
    String getDescription();
}

// StandardPricing.java
package com.example.orders.pricing;

import com.example.orders.domain.Order;
import java.math.BigDecimal;

public class StandardPricing implements PricingStrategy {
    
    private static final BigDecimal TAX_RATE = new BigDecimal("0.08");
    
    @Override
    public BigDecimal calculateTotal(Order order) {
        BigDecimal subtotal = order.getSubtotal();
        BigDecimal tax = subtotal.multiply(TAX_RATE);
        return subtotal.add(tax);
    }
    
    @Override
    public String getDescription() {
        return "Standard pricing with 8% tax";
    }
}

// DiscountPricing.java
package com.example.orders.pricing;

import com.example.orders.domain.Order;
import java.math.BigDecimal;

public class DiscountPricing implements PricingStrategy {
    
    private final BigDecimal discountPercent;
    private final String promoCode;
    private final PricingStrategy basePricing;
    
    public DiscountPricing(BigDecimal discountPercent, String promoCode, 
                          PricingStrategy basePricing) {
        this.discountPercent = discountPercent;
        this.promoCode = promoCode;
        this.basePricing = basePricing;
    }
    
    @Override
    public BigDecimal calculateTotal(Order order) {
        BigDecimal baseTotal = basePricing.calculateTotal(order);
        BigDecimal discount = baseTotal.multiply(discountPercent)
                                       .divide(new BigDecimal("100"));
        return baseTotal.subtract(discount);
    }
    
    @Override
    public String getDescription() {
        return discountPercent + "% discount with code: " + promoCode;
    }
}

// VIPPricing.java - Easy to add new strategy!
package com.example.orders.pricing;

import com.example.orders.domain.Order;
import java.math.BigDecimal;

public class VIPPricing implements PricingStrategy {
    
    private static final BigDecimal VIP_TAX_RATE = new BigDecimal("0.05");
    private static final BigDecimal FREE_SHIPPING_THRESHOLD = new BigDecimal("100");
    
    @Override
    public BigDecimal calculateTotal(Order order) {
        BigDecimal subtotal = order.getSubtotal();
        BigDecimal tax = subtotal.multiply(VIP_TAX_RATE);
        // VIPs get free shipping over $100
        return subtotal.add(tax);
    }
    
    @Override
    public String getDescription() {
        return "VIP pricing: 5% tax, free shipping over $100";
    }
}
```

### Order Service (Orchestration)

```java
// OrderService.java
package com.example.orders.service;

import com.example.orders.domain.*;
import com.example.orders.notification.NotificationService;
import com.example.orders.pricing.PricingStrategy;
import com.example.orders.repository.OrderRepository;
import com.example.orders.validation.*;
import org.springframework.stereotype.Service;
import java.math.BigDecimal;

/**
 * Orchestrates order processing.
 * Follows all SOLID principles:
 * - SRP: Only orchestrates, doesn't implement details
 * - OCP: New features via new strategies/channels
 * - LSP: All injected implementations are interchangeable
 * - ISP: Depends on focused interfaces
 * - DIP: Depends on abstractions, not concretions
 */
@Service
public class OrderService {
    
    private final OrderValidator validator;
    private final OrderRepository repository;
    private final NotificationService notificationService;
    private final PricingStrategy pricingStrategy;
    
    public OrderService(OrderValidator validator,
                       OrderRepository repository,
                       NotificationService notificationService,
                       PricingStrategy pricingStrategy) {
        this.validator = validator;
        this.repository = repository;
        this.notificationService = notificationService;
        this.pricingStrategy = pricingStrategy;
    }
    
    public OrderResult processOrder(Order order) {
        // Step 1: Validate
        ValidationResult validation = validator.validate(order);
        if (!validation.isValid()) {
            return OrderResult.failure(validation.getErrors());
        }
        
        // Step 2: Calculate price
        BigDecimal total = pricingStrategy.calculateTotal(order);
        
        // Step 3: Save
        order.setStatus(OrderStatus.CONFIRMED);
        repository.save(order);
        
        // Step 4: Notify
        notificationService.notifyOrderConfirmation(order);
        
        return OrderResult.success(order.getId(), total);
    }
}

// OrderResult.java
package com.example.orders.service;

import java.math.BigDecimal;
import java.util.List;

public record OrderResult(
    boolean success,
    String orderId,
    BigDecimal total,
    List<String> errors
) {
    public static OrderResult success(String orderId, BigDecimal total) {
        return new OrderResult(true, orderId, total, List.of());
    }
    
    public static OrderResult failure(List<String> errors) {
        return new OrderResult(false, null, null, errors);
    }
}
```

### Configuration (Wiring It Together)

```java
// OrderConfig.java
package com.example.orders.config;

import com.example.orders.pricing.*;
import com.example.orders.validation.OrderValidator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.math.BigDecimal;

@Configuration
public class OrderConfig {
    
    @Bean
    public OrderValidator orderValidator() {
        return new OrderValidator();
    }
    
    @Bean
    public PricingStrategy pricingStrategy() {
        // Easy to change pricing strategy
        // return new StandardPricing();
        return new DiscountPricing(
            new BigDecimal("10"),
            "HOLIDAY10",
            new StandardPricing()
        );
    }
}
```

### Unit Tests

```java
// OrderServiceTest.java
package com.example.orders.service;

import com.example.orders.domain.*;
import com.example.orders.notification.NotificationService;
import com.example.orders.pricing.PricingStrategy;
import com.example.orders.repository.OrderRepository;
import com.example.orders.validation.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository repository;
    
    @Mock
    private NotificationService notificationService;
    
    @Mock
    private PricingStrategy pricingStrategy;
    
    private OrderValidator validator;
    private OrderService orderService;
    
    @BeforeEach
    void setUp() {
        validator = new OrderValidator();  // Use real validator
        orderService = new OrderService(
            validator, repository, notificationService, pricingStrategy
        );
    }
    
    @Test
    void shouldProcessValidOrder() {
        // Arrange
        Customer customer = new Customer(
            "C1", "John", "john@example.com", "+1234567890",
            NotificationPreference.EMAIL
        );
        Order order = new Order("O1", customer);
        order.addItem(new OrderItem("P1", "Widget", 2, new BigDecimal("25.00")));
        
        when(pricingStrategy.calculateTotal(order))
            .thenReturn(new BigDecimal("54.00"));
        
        // Act
        OrderResult result = orderService.processOrder(order);
        
        // Assert
        assertThat(result.success()).isTrue();
        assertThat(result.orderId()).isEqualTo("O1");
        assertThat(result.total()).isEqualByComparingTo("54.00");
        
        verify(repository).save(order);
        verify(notificationService).notifyOrderConfirmation(order);
    }
    
    @Test
    void shouldRejectEmptyOrder() {
        // Arrange
        Customer customer = new Customer(
            "C1", "John", "john@example.com", "+1234567890",
            NotificationPreference.EMAIL
        );
        Order order = new Order("O1", customer);
        // No items added!
        
        // Act
        OrderResult result = orderService.processOrder(order);
        
        // Assert
        assertThat(result.success()).isFalse();
        assertThat(result.errors())
            .contains("Order must contain at least one item");
        
        verify(repository, never()).save(any());
        verify(notificationService, never()).notifyOrderConfirmation(any());
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Over-Engineering with SOLID

**The Problem**: Applying SOLID to everything, even simple code.

```java
// OVER-ENGINEERED: Simple utility doesn't need all this
public interface StringFormatter {
    String format(String input);
}

public interface StringFormatterFactory {
    StringFormatter createFormatter();
}

public class UpperCaseFormatter implements StringFormatter {
    @Override
    public String format(String input) {
        return input.toUpperCase();
    }
}

public class UpperCaseFormatterFactory implements StringFormatterFactory {
    @Override
    public StringFormatter createFormatter() {
        return new UpperCaseFormatter();
    }
}

// Just do this instead!
public class StringUtils {
    public static String toUpperCase(String input) {
        return input.toUpperCase();
    }
}
```

### When to Apply SOLID

| Scenario                      | Apply SOLID? | Reason                                      |
| ----------------------------- | ------------ | ------------------------------------------- |
| Core business logic           | ✅ Yes       | Changes frequently, needs flexibility       |
| Infrastructure (DB, HTTP)     | ✅ Yes       | May need to swap implementations            |
| Simple utilities              | ❌ No        | Over-engineering adds complexity            |
| One-off scripts               | ❌ No        | Won't be maintained                         |
| Prototype/MVP                 | ⚠️ Maybe    | Speed matters, but consider future          |
| Code that will never change   | ❌ No        | YAGNI (You Aren't Gonna Need It)            |

### Common Mistakes

**1. Interface for Everything**

```java
// WRONG: Interface with single implementation that will never change
public interface UserService {
    User findById(Long id);
}

public class UserServiceImpl implements UserService {
    // Only implementation, ever
}

// RIGHT: Just use the class directly until you need abstraction
public class UserService {
    public User findById(Long id) { ... }
}
```

**2. Violating SRP by Splitting Too Much**

```java
// WRONG: Over-split, too many tiny classes
public class UserFirstNameValidator { }
public class UserLastNameValidator { }
public class UserEmailValidator { }
public class UserPhoneValidator { }
public class UserAddressValidator { }

// RIGHT: One cohesive validator
public class UserValidator {
    public ValidationResult validate(User user) {
        // All user validation in one place
    }
}
```

**3. Misunderstanding OCP**

```java
// WRONG: Thinking OCP means never modify any code
// Reality: OCP is about design, not a rule against all changes

// You CAN modify code to:
// - Fix bugs
// - Improve performance
// - Refactor for clarity

// OCP means: Design so NEW FEATURES don't require modifying existing code
```

### SOLID Anti-Patterns

| Anti-Pattern                | SOLID Violation | Fix                                    |
| --------------------------- | --------------- | -------------------------------------- |
| God Class                   | SRP             | Split into focused classes             |
| Switch on Type              | OCP             | Use polymorphism                       |
| Refused Bequest             | LSP             | Don't inherit if can't substitute      |
| Fat Interface               | ISP             | Split into smaller interfaces          |
| Service Locator             | DIP             | Use constructor injection              |

---

## 8️⃣ When NOT to Use SOLID Strictly

### Situations Where Pragmatism Wins

**1. Performance-Critical Code**

```java
// SOLID version: Many small objects, virtual calls
public interface Calculator {
    int calculate(int a, int b);
}

public class Adder implements Calculator { ... }
public class Multiplier implements Calculator { ... }

// Performance-critical version: Direct, no indirection
public class MathOps {
    public static int add(int a, int b) {
        return a + b;
    }
}
```

**2. Prototypes and Experiments**

```java
// Prototype: Just make it work
public class QuickExperiment {
    public void doEverything() {
        // Validate, process, save, notify
        // All in one method is FINE for prototypes
    }
}
```

**3. Simple CRUD Applications**

```java
// For simple CRUD, Spring Data JPA is enough
// No need for custom repository interfaces
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring provides implementation
}
```

### The Pragmatic Approach

1. **Start simple**: Don't add abstractions until needed
2. **Refactor when pain appears**: If changing one thing breaks others, apply SOLID
3. **Consider team size**: Solo project needs less structure than 50-person team
4. **Consider lifespan**: Throwaway code needs less design
5. **Consider change frequency**: Stable code needs less flexibility

---

## 9️⃣ Comparison with Alternatives

### SOLID vs Other Design Principles

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DESIGN PRINCIPLES COMPARISON                          │
│                                                                          │
│   SOLID                          DRY (Don't Repeat Yourself)            │
│   ─────                          ───                                     │
│   Focus: Class design            Focus: Code duplication                │
│   Goal: Maintainability          Goal: Single source of truth           │
│   When: OOP design               When: Any code                         │
│                                                                          │
│   SOLID                          KISS (Keep It Simple, Stupid)          │
│   ─────                          ────                                    │
│   Can add complexity             Minimize complexity                     │
│   Worth it for flexibility       Simplest solution that works           │
│   Balance needed!                Sometimes conflicts with SOLID          │
│                                                                          │
│   SOLID                          YAGNI (You Aren't Gonna Need It)       │
│   ─────                          ─────                                   │
│   Design for extension           Don't build what you don't need        │
│   Prepare for change             Implement when needed                  │
│   Balance needed!                Sometimes conflicts with OCP            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### When Each Principle Applies

| Principle | Apply When                          | Skip When                        |
| --------- | ----------------------------------- | -------------------------------- |
| SRP       | Class has multiple reasons to change | Class is cohesive and focused    |
| OCP       | Feature changes are frequent        | Code is stable and simple        |
| LSP       | Using inheritance                   | Using composition                |
| ISP       | Interfaces have many methods        | Interfaces are already focused   |
| DIP       | Need to swap implementations        | Single implementation forever    |
| DRY       | Same logic appears multiple times   | Similar but not same logic       |
| KISS      | Always                              | Never skip simplicity            |
| YAGNI     | Speculative features                | Clear requirements exist         |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What does SOLID stand for?**

A: SOLID is an acronym for five object-oriented design principles:
- **S**ingle Responsibility: A class should have only one reason to change
- **O**pen/Closed: Open for extension, closed for modification
- **L**iskov Substitution: Subtypes must be substitutable for their base types
- **I**nterface Segregation: Many specific interfaces beat one general interface
- **D**ependency Inversion: Depend on abstractions, not concretions

These principles help create maintainable, flexible, and testable code.

**Q: Give an example of Single Responsibility Principle.**

A: Consider an `Employee` class that handles employee data, calculates payroll, generates reports, and sends emails. This violates SRP because it has four reasons to change: data structure changes, payroll rules change, report format changes, or email provider changes.

The fix is to split into focused classes: `Employee` (data), `PayrollCalculator` (calculations), `ReportGenerator` (reports), and `EmailService` (notifications). Now each class has one reason to change.

### L5 (Mid-Level) Questions

**Q: How would you refactor code that violates the Open/Closed Principle?**

A: I'd look for switch statements or if-else chains that check types:

```java
// Before: Violates OCP
if (shape instanceof Circle) { ... }
else if (shape instanceof Rectangle) { ... }
```

I'd refactor to use polymorphism:

```java
// After: Follows OCP
interface Shape { double area(); }
class Circle implements Shape { double area() { ... } }
class Rectangle implements Shape { double area() { ... } }
```

Now adding Triangle doesn't require modifying existing code, just creating a new class.

**Q: Explain Dependency Inversion with a real example.**

A: In a payment system, instead of `OrderService` directly depending on `StripePaymentProcessor`:

```java
// Wrong: High-level depends on low-level
class OrderService {
    private StripePaymentProcessor processor = new StripePaymentProcessor();
}
```

I'd introduce an abstraction:

```java
// Right: Both depend on abstraction
interface PaymentProcessor { void process(Payment p); }
class StripePaymentProcessor implements PaymentProcessor { ... }
class OrderService {
    private PaymentProcessor processor; // Interface!
    OrderService(PaymentProcessor processor) { this.processor = processor; }
}
```

Now OrderService doesn't know about Stripe. We can swap to PayPal, use a mock for testing, or add new processors without changing OrderService.

### L6 (Senior) Questions

**Q: When would you intentionally violate SOLID principles?**

A: I'd violate SOLID when:

1. **Performance is critical**: Virtual dispatch has overhead. In hot paths, direct calls may be necessary.

2. **Prototyping**: Speed of development matters more than maintainability.

3. **Simple code**: A 20-line utility doesn't need interfaces and dependency injection.

4. **YAGNI**: If I'm certain requirements won't change, adding flexibility is waste.

The key is being intentional. I document why I'm not following SOLID and ensure the team agrees. We can always refactor when the pain of not having SOLID exceeds the cost of adding it.

**Q: How do you balance SOLID with KISS and YAGNI?**

A: These principles can conflict. My approach:

1. **Start simple** (KISS): Don't add abstractions until needed.

2. **Watch for pain points**: When changing one thing breaks another, or testing requires complex setup, that's when SOLID helps.

3. **Apply incrementally**: Don't redesign everything at once. Apply SOLID to the parts that hurt.

4. **Consider context**: Core business logic needs more SOLID than utility code. High-change areas need more than stable areas.

5. **Team agreement**: The team should agree on where to apply SOLID. Inconsistency is worse than imperfection.

Real example: We had a `ReportGenerator` class. Initially, it was one class doing everything. When we needed PDF and Excel formats, we applied OCP and created a `ReportFormatter` interface. We didn't do this upfront because YAGNI, but we did it when the need was clear.

---

## 1️⃣1️⃣ One Clean Mental Summary

SOLID principles are guidelines for writing code that's easy to change without breaking. Think of them as building with LEGO instead of glue: **S**ingle Responsibility means each piece does one thing, **O**pen/Closed means you can add new pieces without modifying existing ones, **L**iskov Substitution means any piece of the same type works the same way, **I**nterface Segregation means pieces have only the connectors they need, and **D**ependency Inversion means pieces connect through standard interfaces, not custom glue. Apply SOLID to code that changes frequently and needs testing. Don't over-apply to simple utilities or prototypes. The goal isn't perfect adherence. It's code that's easy to understand, test, and modify. When you feel pain changing code, that's when SOLID principles help the most.

