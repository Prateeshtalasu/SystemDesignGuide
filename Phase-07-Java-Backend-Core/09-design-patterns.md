# 🏗️ Design Patterns (Gang of Four)

---

## 0️⃣ Prerequisites

Before diving into Design Patterns, you need to understand:

- **OOP Fundamentals**: Encapsulation, inheritance, polymorphism, abstraction (covered in `01-oop-fundamentals.md`)
- **SOLID Principles**: Single Responsibility, Open/Closed, etc. (covered in `02-solid-principles.md`)
- **Interfaces and Abstract Classes**: When to use each
- **Composition vs Inheritance**: Why "favor composition over inheritance"

Quick mental model:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHAT ARE DESIGN PATTERNS?                             │
│                                                                          │
│   Design patterns are REUSABLE SOLUTIONS to COMMON PROBLEMS             │
│                                                                          │
│   They are NOT:                                                         │
│   - Libraries or frameworks you import                                  │
│   - Algorithms with specific implementations                            │
│   - Rules you must always follow                                        │
│                                                                          │
│   They ARE:                                                             │
│   - Templates for solving recurring design problems                     │
│   - A shared vocabulary for developers                                  │
│   - Proven approaches refined over decades                              │
│                                                                          │
│   Categories:                                                           │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │
│   │ CREATIONAL  │  │ STRUCTURAL  │  │ BEHAVIORAL  │                    │
│   │             │  │             │  │             │                    │
│   │ How objects │  │ How objects │  │ How objects │                    │
│   │ are created │  │ are composed│  │ communicate │                    │
│   └─────────────┘  └─────────────┘  └─────────────┘                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Without design patterns, developers repeatedly solve the same problems in different ways:

```java
// Developer A creates objects like this:
public class OrderService {
    private Database db = new MySQLDatabase();  // Hardcoded!
    private EmailService email = new SmtpEmailService();  // Hardcoded!
}

// Developer B creates objects like this:
public class UserService {
    private Database db;

    public UserService() {
        this.db = DatabaseFactory.create("mysql");  // Different approach
    }
}

// Developer C creates objects like this:
public class ProductService {
    public void process() {
        Database db = Database.getInstance();  // Yet another approach
    }
}

// Result: Inconsistent codebase, hard to maintain, hard to test
```

### What Systems Looked Like Before Patterns

Before the Gang of Four (GoF) book in 1994:

- Each developer invented their own solutions
- No common vocabulary ("I used a thing that creates objects" vs "I used a Factory")
- Knowledge wasn't easily transferred between projects
- Same mistakes were repeated across the industry

### Real Examples of the Problem

**The Notification System Problem**:

```java
// Without patterns: Rigid, hard to extend
public class NotificationService {
    public void notify(User user, String message, String type) {
        if (type.equals("email")) {
            // Send email
        } else if (type.equals("sms")) {
            // Send SMS
        } else if (type.equals("push")) {
            // Send push notification
        }
        // Adding Slack? WhatsApp? Modify this class every time!
    }
}
```

---

## 2️⃣ Creational Patterns

### Factory Method

**Problem**: You need to create objects, but the exact type depends on runtime conditions.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FACTORY METHOD                                        │
│                                                                          │
│   Instead of:  new ConcreteProduct()                                    │
│   Use:         factory.createProduct()                                  │
│                                                                          │
│   ┌────────────────────┐                                                │
│   │   ProductFactory   │ (abstract)                                     │
│   │   ───────────────  │                                                │
│   │ + createProduct()  │ ◄───── Subclasses decide what to create       │
│   └─────────┬──────────┘                                                │
│             │                                                            │
│   ┌─────────┴─────────┬─────────────────────┐                          │
│   ▼                   ▼                     ▼                           │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                     │
│ │EmailFactory  │ │ SMSFactory   │ │ PushFactory  │                     │
│ │──────────────│ │──────────────│ │──────────────│                     │
│ │createProduct │ │createProduct │ │createProduct │                     │
│ │returns Email │ │returns SMS   │ │returns Push  │                     │
│ └──────────────┘ └──────────────┘ └──────────────┘                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

```java
// Product interface
public interface Notification {
    void send(String message);
}

// Concrete products
public class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending email: " + message);
    }
}

public class SMSNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending SMS: " + message);
    }
}

public class PushNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending push: " + message);
    }
}

// Factory
public class NotificationFactory {
    public static Notification createNotification(String channel) {
        return switch (channel.toLowerCase()) {
            case "email" -> new EmailNotification();
            case "sms" -> new SMSNotification();
            case "push" -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown channel: " + channel);
        };
    }
}

// Usage
Notification notification = NotificationFactory.createNotification("email");
notification.send("Hello!");
```

**Real-world usage**: `Calendar.getInstance()`, `NumberFormat.getInstance()`, Spring's `BeanFactory`

---

### Abstract Factory

**Problem**: You need to create families of related objects without specifying their concrete classes.

```java
// Abstract products
public interface Button {
    void render();
}

public interface Checkbox {
    void render();
}

// Concrete products - Windows family
public class WindowsButton implements Button {
    @Override
    public void render() {
        System.out.println("Rendering Windows button");
    }
}

public class WindowsCheckbox implements Checkbox {
    @Override
    public void render() {
        System.out.println("Rendering Windows checkbox");
    }
}

// Concrete products - Mac family
public class MacButton implements Button {
    @Override
    public void render() {
        System.out.println("Rendering Mac button");
    }
}

public class MacCheckbox implements Checkbox {
    @Override
    public void render() {
        System.out.println("Rendering Mac checkbox");
    }
}

// Abstract factory
public interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete factories
public class WindowsFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }

    @Override
    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
}

public class MacFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }

    @Override
    public Checkbox createCheckbox() {
        return new MacCheckbox();
    }
}

// Usage
public class Application {
    private Button button;
    private Checkbox checkbox;

    public Application(GUIFactory factory) {
        button = factory.createButton();
        checkbox = factory.createCheckbox();
    }

    public void render() {
        button.render();
        checkbox.render();
    }
}

// Client code
GUIFactory factory = System.getProperty("os.name").contains("Windows")
    ? new WindowsFactory()
    : new MacFactory();
Application app = new Application(factory);
app.render();
```

**Real-world usage**: JDBC's `Connection`, `Statement`, `ResultSet` families

---

### Builder

**Problem**: You need to construct complex objects step by step, with many optional parameters.

```java
// The product
public class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeout;
    private final boolean followRedirects;

    private HttpRequest(Builder builder) {
        this.method = builder.method;
        this.url = builder.url;
        this.headers = builder.headers;
        this.body = builder.body;
        this.timeout = builder.timeout;
        this.followRedirects = builder.followRedirects;
    }

    // Getters...

    public static class Builder {
        // Required parameters
        private final String method;
        private final String url;

        // Optional parameters with defaults
        private Map<String, String> headers = new HashMap<>();
        private String body = "";
        private int timeout = 30000;
        private boolean followRedirects = true;

        public Builder(String method, String url) {
            this.method = method;
            this.url = url;
        }

        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }

        public Builder followRedirects(boolean follow) {
            this.followRedirects = follow;
            return this;
        }

        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }
}

// Usage - fluent, readable
HttpRequest request = new HttpRequest.Builder("POST", "https://api.example.com/users")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\": \"John\"}")
    .timeout(5000)
    .build();
```

**Real-world usage**: `StringBuilder`, `Stream.Builder`, Lombok's `@Builder`, OkHttp's `Request.Builder`

---

### Singleton

**Problem**: Ensure a class has only one instance and provide global access to it.

```java
// Thread-safe Singleton using holder pattern (recommended)
public class ConfigurationManager {

    private final Properties config;

    private ConfigurationManager() {
        config = new Properties();
        // Load configuration
    }

    // Holder class - loaded only when getInstance() is called
    private static class Holder {
        static final ConfigurationManager INSTANCE = new ConfigurationManager();
    }

    public static ConfigurationManager getInstance() {
        return Holder.INSTANCE;
    }

    public String get(String key) {
        return config.getProperty(key);
    }
}

// Enum Singleton (Effective Java recommended)
public enum DatabaseConnection {
    INSTANCE;

    private Connection connection;

    DatabaseConnection() {
        // Initialize connection
    }

    public Connection getConnection() {
        return connection;
    }
}

// Usage
ConfigurationManager config = ConfigurationManager.getInstance();
DatabaseConnection db = DatabaseConnection.INSTANCE;
```

**Warning**: Singletons can make testing difficult. Prefer dependency injection in modern applications.

**Real-world usage**: `Runtime.getRuntime()`, Spring beans (default scope is singleton)

---

### Prototype

**Problem**: Create new objects by copying existing ones, especially when creation is expensive.

```java
public interface Prototype<T> {
    T clone();
}

public class GameCharacter implements Prototype<GameCharacter> {
    private String name;
    private int health;
    private int level;
    private List<String> inventory;
    private Map<String, Integer> skills;

    public GameCharacter(String name) {
        this.name = name;
        this.health = 100;
        this.level = 1;
        this.inventory = new ArrayList<>();
        this.skills = new HashMap<>();
    }

    // Private constructor for cloning
    private GameCharacter(GameCharacter source) {
        this.name = source.name;
        this.health = source.health;
        this.level = source.level;
        this.inventory = new ArrayList<>(source.inventory);  // Deep copy
        this.skills = new HashMap<>(source.skills);          // Deep copy
    }

    @Override
    public GameCharacter clone() {
        return new GameCharacter(this);
    }

    // Setters for customization after cloning...
}

// Prototype registry
public class CharacterRegistry {
    private Map<String, GameCharacter> prototypes = new HashMap<>();

    public void register(String key, GameCharacter prototype) {
        prototypes.put(key, prototype);
    }

    public GameCharacter create(String key) {
        GameCharacter prototype = prototypes.get(key);
        return prototype != null ? prototype.clone() : null;
    }
}

// Usage
CharacterRegistry registry = new CharacterRegistry();

// Create and configure a template
GameCharacter warriorTemplate = new GameCharacter("Warrior");
warriorTemplate.setHealth(150);
warriorTemplate.addSkill("Sword", 10);
registry.register("warrior", warriorTemplate);

// Create instances from template
GameCharacter warrior1 = registry.create("warrior");
GameCharacter warrior2 = registry.create("warrior");
```

**Real-world usage**: `Object.clone()`, JavaScript's prototype chain

---

## 3️⃣ Structural Patterns

### Adapter

**Problem**: Make incompatible interfaces work together.

```java
// Target interface (what our code expects)
public interface PaymentProcessor {
    void processPayment(double amount, String currency);
}

// Adaptee (third-party library with different interface)
public class StripeAPI {
    public void charge(int amountInCents, String currencyCode) {
        System.out.println("Stripe charging " + amountInCents + " " + currencyCode);
    }
}

// Adapter
public class StripeAdapter implements PaymentProcessor {
    private final StripeAPI stripe;

    public StripeAdapter(StripeAPI stripe) {
        this.stripe = stripe;
    }

    @Override
    public void processPayment(double amount, String currency) {
        // Convert dollars to cents
        int amountInCents = (int) (amount * 100);
        // Convert currency format
        String currencyCode = currency.toUpperCase();
        stripe.charge(amountInCents, currencyCode);
    }
}

// Usage
PaymentProcessor processor = new StripeAdapter(new StripeAPI());
processor.processPayment(99.99, "usd");
```

**Real-world usage**: `Arrays.asList()`, `InputStreamReader`, Spring's `HandlerAdapter`

---

### Decorator

**Problem**: Add behavior to objects dynamically without affecting other objects of the same class.

```java
// Component interface
public interface Coffee {
    String getDescription();
    double getCost();
}

// Concrete component
public class Espresso implements Coffee {
    @Override
    public String getDescription() {
        return "Espresso";
    }

    @Override
    public double getCost() {
        return 2.00;
    }
}

// Base decorator
public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee coffee;

    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }

    @Override
    public String getDescription() {
        return coffee.getDescription();
    }

    @Override
    public double getCost() {
        return coffee.getCost();
    }
}

// Concrete decorators
public class Milk extends CoffeeDecorator {
    public Milk(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Milk";
    }

    @Override
    public double getCost() {
        return coffee.getCost() + 0.50;
    }
}

public class Whip extends CoffeeDecorator {
    public Whip(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Whip";
    }

    @Override
    public double getCost() {
        return coffee.getCost() + 0.75;
    }
}

// Usage - decorators can be stacked
Coffee order = new Whip(new Milk(new Espresso()));
System.out.println(order.getDescription());  // "Espresso, Milk, Whip"
System.out.println(order.getCost());         // 3.25
```

**Real-world usage**: `BufferedInputStream`, `Collections.synchronizedList()`, Spring Security filters

---

### Facade

**Problem**: Provide a simplified interface to a complex subsystem.

```java
// Complex subsystem classes
public class VideoDecoder {
    public void decode(String filename) {
        System.out.println("Decoding video: " + filename);
    }
}

public class AudioDecoder {
    public void decode(String filename) {
        System.out.println("Decoding audio: " + filename);
    }
}

public class VideoRenderer {
    public void render(int width, int height) {
        System.out.println("Rendering video at " + width + "x" + height);
    }
}

public class AudioPlayer {
    public void play() {
        System.out.println("Playing audio");
    }
}

public class SubtitleLoader {
    public void load(String filename) {
        System.out.println("Loading subtitles: " + filename);
    }
}

// Facade - simple interface to complex subsystem
public class VideoPlayerFacade {
    private final VideoDecoder videoDecoder;
    private final AudioDecoder audioDecoder;
    private final VideoRenderer videoRenderer;
    private final AudioPlayer audioPlayer;
    private final SubtitleLoader subtitleLoader;

    public VideoPlayerFacade() {
        this.videoDecoder = new VideoDecoder();
        this.audioDecoder = new AudioDecoder();
        this.videoRenderer = new VideoRenderer();
        this.audioPlayer = new AudioPlayer();
        this.subtitleLoader = new SubtitleLoader();
    }

    // Simple method that orchestrates the complex subsystem
    public void play(String videoFile) {
        videoDecoder.decode(videoFile);
        audioDecoder.decode(videoFile);
        videoRenderer.render(1920, 1080);
        audioPlayer.play();
    }

    public void playWithSubtitles(String videoFile, String subtitleFile) {
        play(videoFile);
        subtitleLoader.load(subtitleFile);
    }
}

// Usage - client doesn't need to know about subsystem complexity
VideoPlayerFacade player = new VideoPlayerFacade();
player.play("movie.mp4");
```

**Real-world usage**: `javax.faces.context.FacesContext`, SLF4J logging facade, Spring's `JdbcTemplate`

---

### Proxy

**Problem**: Control access to an object by providing a surrogate or placeholder.

```java
// Subject interface
public interface Image {
    void display();
}

// Real subject (expensive to create)
public class HighResolutionImage implements Image {
    private final String filename;
    private byte[] imageData;

    public HighResolutionImage(String filename) {
        this.filename = filename;
        loadFromDisk();  // Expensive operation
    }

    private void loadFromDisk() {
        System.out.println("Loading image from disk: " + filename);
        // Simulate loading large file
        try { Thread.sleep(2000); } catch (InterruptedException e) {}
        imageData = new byte[10_000_000];  // 10MB image
    }

    @Override
    public void display() {
        System.out.println("Displaying: " + filename);
    }
}

// Proxy (lazy loading)
public class ImageProxy implements Image {
    private final String filename;
    private HighResolutionImage realImage;

    public ImageProxy(String filename) {
        this.filename = filename;
        // Don't load yet!
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new HighResolutionImage(filename);  // Load on first use
        }
        realImage.display();
    }
}

// Usage
List<Image> gallery = new ArrayList<>();
gallery.add(new ImageProxy("photo1.jpg"));  // Instant
gallery.add(new ImageProxy("photo2.jpg"));  // Instant
gallery.add(new ImageProxy("photo3.jpg"));  // Instant

// Images loaded only when displayed
gallery.get(0).display();  // Loads photo1.jpg now
```

**Types of proxies**:

- **Virtual Proxy**: Lazy loading (shown above)
- **Protection Proxy**: Access control
- **Remote Proxy**: Represent remote objects (RMI)
- **Caching Proxy**: Cache results

**Real-world usage**: Hibernate lazy loading, Spring AOP proxies, Java RMI

---

### Composite

**Problem**: Treat individual objects and compositions of objects uniformly.

```java
// Component
public interface FileSystemItem {
    String getName();
    long getSize();
    void print(String indent);
}

// Leaf
public class File implements FileSystemItem {
    private final String name;
    private final long size;

    public File(String name, long size) {
        this.name = name;
        this.size = size;
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public long getSize() {
        return size;
    }

    @Override
    public void print(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " bytes)");
    }
}

// Composite
public class Directory implements FileSystemItem {
    private final String name;
    private final List<FileSystemItem> children = new ArrayList<>();

    public Directory(String name) {
        this.name = name;
    }

    public void add(FileSystemItem item) {
        children.add(item);
    }

    public void remove(FileSystemItem item) {
        children.remove(item);
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public long getSize() {
        return children.stream()
            .mapToLong(FileSystemItem::getSize)
            .sum();
    }

    @Override
    public void print(String indent) {
        System.out.println(indent + "📁 " + name + " (" + getSize() + " bytes)");
        for (FileSystemItem child : children) {
            child.print(indent + "  ");
        }
    }
}

// Usage
Directory root = new Directory("root");
Directory docs = new Directory("documents");
docs.add(new File("resume.pdf", 50000));
docs.add(new File("cover_letter.docx", 25000));

Directory photos = new Directory("photos");
photos.add(new File("vacation.jpg", 2000000));

root.add(docs);
root.add(photos);
root.add(new File("readme.txt", 1000));

root.print("");
// Output:
// 📁 root (2076000 bytes)
//   📁 documents (75000 bytes)
//     📄 resume.pdf (50000 bytes)
//     📄 cover_letter.docx (25000 bytes)
//   📁 photos (2000000 bytes)
//     📄 vacation.jpg (2000000 bytes)
//   📄 readme.txt (1000 bytes)
```

**Real-world usage**: UI component trees, XML/JSON parsers, organization hierarchies

---

## 4️⃣ Behavioral Patterns

### Strategy

**Problem**: Define a family of algorithms, encapsulate each one, and make them interchangeable.

```java
// Strategy interface
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete strategies
public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;

    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " with credit card " +
            cardNumber.substring(cardNumber.length() - 4));
    }
}

public class PayPalPayment implements PaymentStrategy {
    private final String email;

    public PayPalPayment(String email) {
        this.email = email;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " via PayPal (" + email + ")");
    }
}

public class CryptoPayment implements PaymentStrategy {
    private final String walletAddress;

    public CryptoPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " in crypto to " +
            walletAddress.substring(0, 8) + "...");
    }
}

// Context
public class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    private double total = 0;

    public void addItem(double price) {
        total += price;
    }

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout() {
        if (paymentStrategy == null) {
            throw new IllegalStateException("Payment method not selected");
        }
        paymentStrategy.pay(total);
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.addItem(100);
cart.addItem(50);

// User selects payment method at runtime
cart.setPaymentStrategy(new CreditCardPayment("1234-5678-9012-3456"));
cart.checkout();

// Or switch strategy
cart.setPaymentStrategy(new PayPalPayment("user@example.com"));
cart.checkout();
```

**Real-world usage**: `Comparator`, `java.util.zip` compression strategies, Spring's `ResourceLoader`

---

### Observer

**Problem**: Define a one-to-many dependency so that when one object changes state, all dependents are notified.

```java
// Observer interface
public interface StockObserver {
    void update(String stockSymbol, double price);
}

// Subject
public class StockMarket {
    private final Map<String, Double> stocks = new HashMap<>();
    private final List<StockObserver> observers = new ArrayList<>();

    public void addObserver(StockObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(StockObserver observer) {
        observers.remove(observer);
    }

    public void setStockPrice(String symbol, double price) {
        stocks.put(symbol, price);
        notifyObservers(symbol, price);
    }

    private void notifyObservers(String symbol, double price) {
        for (StockObserver observer : observers) {
            observer.update(symbol, price);
        }
    }
}

// Concrete observers
public class StockDisplay implements StockObserver {
    private final String name;

    public StockDisplay(String name) {
        this.name = name;
    }

    @Override
    public void update(String stockSymbol, double price) {
        System.out.println("[" + name + "] " + stockSymbol + ": $" + price);
    }
}

public class PriceAlert implements StockObserver {
    private final String targetStock;
    private final double threshold;

    public PriceAlert(String targetStock, double threshold) {
        this.targetStock = targetStock;
        this.threshold = threshold;
    }

    @Override
    public void update(String stockSymbol, double price) {
        if (stockSymbol.equals(targetStock) && price > threshold) {
            System.out.println("🚨 ALERT: " + stockSymbol + " exceeded $" + threshold);
        }
    }
}

// Usage
StockMarket market = new StockMarket();
market.addObserver(new StockDisplay("Main Display"));
market.addObserver(new StockDisplay("Mobile App"));
market.addObserver(new PriceAlert("AAPL", 150.0));

market.setStockPrice("AAPL", 145.0);
market.setStockPrice("AAPL", 155.0);  // Triggers alert
```

**Real-world usage**: Event listeners in Swing/JavaFX, Spring's `ApplicationEventPublisher`, RxJava

---

### Template Method

**Problem**: Define the skeleton of an algorithm in a base class, letting subclasses override specific steps.

```java
// Abstract class with template method
public abstract class DataProcessor {

    // Template method - defines the algorithm skeleton
    public final void process() {
        readData();
        processData();
        writeData();

        // Hook - optional step
        if (shouldNotify()) {
            notifyComplete();
        }
    }

    // Abstract methods - must be implemented by subclasses
    protected abstract void readData();
    protected abstract void processData();
    protected abstract void writeData();

    // Hook method - can be overridden
    protected boolean shouldNotify() {
        return false;
    }

    // Concrete method - common implementation
    protected void notifyComplete() {
        System.out.println("Processing complete!");
    }
}

// Concrete implementations
public class CSVProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading CSV file");
    }

    @Override
    protected void processData() {
        System.out.println("Parsing CSV rows");
    }

    @Override
    protected void writeData() {
        System.out.println("Writing to database");
    }
}

public class JSONProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading JSON file");
    }

    @Override
    protected void processData() {
        System.out.println("Parsing JSON objects");
    }

    @Override
    protected void writeData() {
        System.out.println("Writing to Elasticsearch");
    }

    @Override
    protected boolean shouldNotify() {
        return true;  // Enable notification
    }
}

// Usage
DataProcessor csvProcessor = new CSVProcessor();
csvProcessor.process();

DataProcessor jsonProcessor = new JSONProcessor();
jsonProcessor.process();
```

**Real-world usage**: `HttpServlet.service()`, Spring's `JdbcTemplate`, JUnit test lifecycle

---

### State

**Problem**: Allow an object to alter its behavior when its internal state changes.

```java
// State interface
public interface OrderState {
    void next(Order order);
    void previous(Order order);
    void printStatus();
}

// Concrete states
public class NewState implements OrderState {
    @Override
    public void next(Order order) {
        order.setState(new PaidState());
    }

    @Override
    public void previous(Order order) {
        System.out.println("Already at initial state");
    }

    @Override
    public void printStatus() {
        System.out.println("Order is NEW - awaiting payment");
    }
}

public class PaidState implements OrderState {
    @Override
    public void next(Order order) {
        order.setState(new ShippedState());
    }

    @Override
    public void previous(Order order) {
        order.setState(new NewState());
    }

    @Override
    public void printStatus() {
        System.out.println("Order is PAID - preparing for shipment");
    }
}

public class ShippedState implements OrderState {
    @Override
    public void next(Order order) {
        order.setState(new DeliveredState());
    }

    @Override
    public void previous(Order order) {
        order.setState(new PaidState());
    }

    @Override
    public void printStatus() {
        System.out.println("Order is SHIPPED - in transit");
    }
}

public class DeliveredState implements OrderState {
    @Override
    public void next(Order order) {
        System.out.println("Order already delivered");
    }

    @Override
    public void previous(Order order) {
        order.setState(new ShippedState());
    }

    @Override
    public void printStatus() {
        System.out.println("Order is DELIVERED");
    }
}

// Context
public class Order {
    private OrderState state = new NewState();

    public void setState(OrderState state) {
        this.state = state;
    }

    public void nextState() {
        state.next(this);
    }

    public void previousState() {
        state.previous(this);
    }

    public void printStatus() {
        state.printStatus();
    }
}

// Usage
Order order = new Order();
order.printStatus();  // NEW
order.nextState();
order.printStatus();  // PAID
order.nextState();
order.printStatus();  // SHIPPED
order.nextState();
order.printStatus();  // DELIVERED
```

**Real-world usage**: TCP connection states, workflow engines, game character states

---

### Command

**Problem**: Encapsulate a request as an object, allowing parameterization and queuing of requests.

```java
// Command interface
public interface Command {
    void execute();
    void undo();
}

// Receiver
public class TextEditor {
    private StringBuilder text = new StringBuilder();

    public void write(String str) {
        text.append(str);
    }

    public void delete(int length) {
        int start = Math.max(0, text.length() - length);
        text.delete(start, text.length());
    }

    public String getText() {
        return text.toString();
    }
}

// Concrete commands
public class WriteCommand implements Command {
    private final TextEditor editor;
    private final String text;

    public WriteCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.text = text;
    }

    @Override
    public void execute() {
        editor.write(text);
    }

    @Override
    public void undo() {
        editor.delete(text.length());
    }
}

// Invoker with command history
public class CommandManager {
    private final Deque<Command> history = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    public void execute(Command command) {
        command.execute();
        history.push(command);
        redoStack.clear();  // Clear redo stack on new command
    }

    public void undo() {
        if (!history.isEmpty()) {
            Command command = history.pop();
            command.undo();
            redoStack.push(command);
        }
    }

    public void redo() {
        if (!redoStack.isEmpty()) {
            Command command = redoStack.pop();
            command.execute();
            history.push(command);
        }
    }
}

// Usage
TextEditor editor = new TextEditor();
CommandManager manager = new CommandManager();

manager.execute(new WriteCommand(editor, "Hello"));
manager.execute(new WriteCommand(editor, " World"));
System.out.println(editor.getText());  // "Hello World"

manager.undo();
System.out.println(editor.getText());  // "Hello"

manager.redo();
System.out.println(editor.getText());  // "Hello World"
```

**Real-world usage**: Swing `Action`, Spring's `CommandLineRunner`, undo/redo systems

---

### Chain of Responsibility

**Problem**: Pass a request along a chain of handlers. Each handler decides to process or pass it along.

```java
// Handler interface
public abstract class SupportHandler {
    protected SupportHandler nextHandler;

    public void setNext(SupportHandler handler) {
        this.nextHandler = handler;
    }

    public abstract void handle(SupportTicket ticket);
}

// Request
public class SupportTicket {
    public enum Priority { LOW, MEDIUM, HIGH, CRITICAL }

    private final String description;
    private final Priority priority;

    public SupportTicket(String description, Priority priority) {
        this.description = description;
        this.priority = priority;
    }

    public String getDescription() { return description; }
    public Priority getPriority() { return priority; }
}

// Concrete handlers
public class Level1Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == SupportTicket.Priority.LOW) {
            System.out.println("Level 1 handling: " + ticket.getDescription());
        } else if (nextHandler != null) {
            nextHandler.handle(ticket);
        }
    }
}

public class Level2Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == SupportTicket.Priority.MEDIUM) {
            System.out.println("Level 2 handling: " + ticket.getDescription());
        } else if (nextHandler != null) {
            nextHandler.handle(ticket);
        }
    }
}

public class Level3Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == SupportTicket.Priority.HIGH) {
            System.out.println("Level 3 handling: " + ticket.getDescription());
        } else if (nextHandler != null) {
            nextHandler.handle(ticket);
        }
    }
}

public class ManagerSupport extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        System.out.println("Manager handling CRITICAL: " + ticket.getDescription());
    }
}

// Usage
SupportHandler level1 = new Level1Support();
SupportHandler level2 = new Level2Support();
SupportHandler level3 = new Level3Support();
SupportHandler manager = new ManagerSupport();

level1.setNext(level2);
level2.setNext(level3);
level3.setNext(manager);

// Tickets flow through the chain
level1.handle(new SupportTicket("Password reset", SupportTicket.Priority.LOW));
level1.handle(new SupportTicket("System outage", SupportTicket.Priority.CRITICAL));
```

**Real-world usage**: Servlet filters, Spring Security filter chain, logging handlers

---

## 5️⃣ Repository Pattern (Spring Data)

```java
// Entity
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private BigDecimal price;
    private String category;

    // Getters, setters, constructors...
}

// Repository interface
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Query methods - Spring Data generates implementation
    List<Product> findByCategory(String category);

    List<Product> findByPriceLessThan(BigDecimal price);

    List<Product> findByNameContainingIgnoreCase(String name);

    // Custom query
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max")
    List<Product> findByPriceRange(@Param("min") BigDecimal min,
                                   @Param("max") BigDecimal max);

    // Native query
    @Query(value = "SELECT * FROM products WHERE category = ?1 ORDER BY price DESC LIMIT ?2",
           nativeQuery = true)
    List<Product> findTopByCategory(String category, int limit);
}

// Service using repository
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public Product createProduct(Product product) {
        return productRepository.save(product);
    }

    public Optional<Product> findById(Long id) {
        return productRepository.findById(id);
    }

    public List<Product> findByCategory(String category) {
        return productRepository.findByCategory(category);
    }

    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

---

## 6️⃣ When to Use Which Pattern

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PATTERN SELECTION GUIDE                               │
│                                                                          │
│   PROBLEM                          PATTERN                              │
│   ───────                          ───────                              │
│   Need to create objects           Factory Method                       │
│   without specifying class                                              │
│                                                                          │
│   Need families of related         Abstract Factory                     │
│   objects                                                               │
│                                                                          │
│   Complex object with many         Builder                              │
│   optional parameters                                                   │
│                                                                          │
│   Only one instance needed         Singleton (or DI)                    │
│                                                                          │
│   Need to copy complex objects     Prototype                            │
│                                                                          │
│   Incompatible interfaces          Adapter                              │
│                                                                          │
│   Add behavior dynamically         Decorator                            │
│                                                                          │
│   Simplify complex subsystem       Facade                               │
│                                                                          │
│   Control access to object         Proxy                                │
│                                                                          │
│   Tree structures                  Composite                            │
│                                                                          │
│   Interchangeable algorithms       Strategy                             │
│                                                                          │
│   Notify multiple objects          Observer                             │
│   of changes                                                            │
│                                                                          │
│   Algorithm skeleton with          Template Method                      │
│   customizable steps                                                    │
│                                                                          │
│   Object behavior depends          State                                │
│   on state                                                              │
│                                                                          │
│   Encapsulate requests,            Command                              │
│   support undo                                                          │
│                                                                          │
│   Pass request through             Chain of Responsibility             │
│   handlers                                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between Factory and Abstract Factory patterns?**

A: **Factory Method** creates one type of object. The factory has a single method that returns different implementations based on input.

**Abstract Factory** creates families of related objects. It has multiple factory methods, each creating a different type of object, but all objects in the family work together.

Example: Factory Method creates different `Button` types. Abstract Factory creates entire UI families (Windows: WindowsButton + WindowsCheckbox, Mac: MacButton + MacCheckbox).

**Q: When would you use the Builder pattern?**

A: Use Builder when:

1. Object has many constructor parameters (more than 4-5)
2. Many parameters are optional
3. Object should be immutable after construction
4. You want readable, fluent construction code

Example: `HttpRequest.Builder("GET", url).header("Auth", token).timeout(5000).build()`

### L5 (Mid-Level) Questions

**Q: Explain the difference between Strategy and State patterns.**

A: Both use composition and can change behavior at runtime, but:

**Strategy**: Client chooses and sets the algorithm. The context doesn't know which strategy it's using. Strategies are interchangeable and typically set once.

**State**: The object itself changes its state based on internal logic. State transitions are defined within states. The context knows its current state.

Example: Strategy - user picks payment method (Credit, PayPal). State - order automatically moves through states (New → Paid → Shipped → Delivered).

**Q: How would you implement undo/redo functionality?**

A: Use the **Command pattern**:

1. Each action is encapsulated as a Command object with `execute()` and `undo()` methods
2. Commands are stored in a history stack
3. Undo: Pop from history, call `undo()`, push to redo stack
4. Redo: Pop from redo stack, call `execute()`, push to history

The key is that each command stores enough state to reverse itself.

### L6 (Senior) Questions

**Q: How do design patterns apply in a microservices architecture?**

A: Several patterns are essential:

- **Facade**: API Gateway provides a unified interface to multiple services
- **Proxy**: Service mesh sidecars (Envoy) proxy all traffic
- **Chain of Responsibility**: Request filters, middleware pipelines
- **Observer**: Event-driven architecture, pub/sub messaging
- **Strategy**: Different implementations for different environments (cloud providers)
- **Circuit Breaker** (not GoF but related): Resilience patterns for service calls

The key insight is that patterns apply at different scales - within a service (class level) and between services (architecture level).

---

## 8️⃣ One Clean Mental Summary

Design patterns are reusable solutions to common software design problems. **Creational patterns** (Factory, Builder, Singleton) handle object creation. **Structural patterns** (Adapter, Decorator, Facade, Proxy) compose objects into larger structures. **Behavioral patterns** (Strategy, Observer, State, Command) manage object communication and responsibilities. Don't force patterns where they're not needed - they add complexity. Use them when you recognize the problem they solve. The most commonly used in Java backends are: Factory (object creation), Builder (complex objects), Strategy (interchangeable algorithms), Observer (event handling), and Decorator (adding behavior). Spring Framework heavily uses these patterns internally: Singleton (bean scope), Factory (BeanFactory), Proxy (AOP), Template Method (JdbcTemplate), and Observer (ApplicationEventPublisher).
