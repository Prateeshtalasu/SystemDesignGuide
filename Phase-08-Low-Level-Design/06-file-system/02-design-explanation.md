# 📁 File System - Design Explanation

## SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `Entry` | Base file system entry behavior | Entry structure changes |
| `File` | File-specific operations (read/write) | File handling changes |
| `Directory` | Directory-specific operations (children) | Directory handling changes |
| `Permissions` | Permission management | Permission model changes |
| `SearchCriteria` | Define search parameters | Search features change |
| `FileSystem` | Coordinate file operations | Command interface changes |

**SRP in Action:**

```java
// File ONLY handles file operations
public class File extends Entry {
    public byte[] read() { }
    public void write(byte[] content) { }
    public void append(byte[] content) { }
}

// Directory ONLY handles directory operations
public class Directory extends Entry {
    public void addChild(Entry entry) { }
    public Entry removeChild(String name) { }
    public Entry getChild(String name) { }
}

// FileSystem coordinates but doesn't implement low-level operations
public class FileSystem {
    public File touch(String path) {
        Directory parent = resolveParentPath(path);
        File file = new File(name, parent);
        parent.addChild(file);  // Delegates to Directory
        return file;
    }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Entry Types:**

```java
// Add symbolic link without changing existing code
public class SymbolicLink extends Entry {
    private final Entry target;
    
    public SymbolicLink(String name, Directory parent, Entry target) {
        super(name, parent);
        this.target = target;
    }
    
    @Override
    public long getSize() {
        return 0;  // Links have no size
    }
    
    @Override
    public boolean isDirectory() {
        return target.isDirectory();
    }
    
    public Entry getTarget() {
        return target;
    }
}
```

**Adding New Search Criteria:**

```java
// SearchCriteria builder is extensible
SearchCriteria criteria = SearchCriteria.builder()
    .nameContains("log")
    .extension("txt")
    .minSize(100)
    .maxSize(10000)
    .filesOnly()
    .readable()
    .modifiedAfter(LocalDateTime.now().minusDays(7))  // New criterion
    .build();
```

**Adding New File Operations:**

```java
// Add compression without changing File class
public class FileCompressor {
    public byte[] compress(File file) {
        byte[] content = file.read();
        return GZIPUtils.compress(content);
    }
    
    public void decompress(File file, byte[] compressed) {
        byte[] content = GZIPUtils.decompress(compressed);
        file.write(content);
    }
}
```

---

### 3. Liskov Substitution Principle (LSP)

**Testing LSP:**

```java
// Any Entry subtype should work in these methods
public void processEntry(Entry entry) {
    String path = entry.getPath();        // Works for File and Directory
    long size = entry.getSize();          // Works for File and Directory
    boolean isDir = entry.isDirectory();  // Works for File and Directory
}

// Both work correctly
processEntry(new File("test.txt", parent));
processEntry(new Directory("testdir", parent));
```

**LSP with Size Calculation:**

```java
// getSize() contract: returns size in bytes
// File: returns content length
// Directory: returns sum of children sizes

public void calculateTotalSize(List<Entry> entries) {
    long total = 0;
    for (Entry entry : entries) {
        total += entry.getSize();  // Works for any Entry
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Current Design:**

Entry has methods used by both File and Directory. Could be improved:

```java
// Better with ISP
public interface Readable {
    byte[] read();
}

public interface Writable {
    void write(byte[] content);
}

public interface Container {
    void addChild(Entry entry);
    Entry removeChild(String name);
    List<Entry> getChildren();
}

// File implements Readable and Writable
public class File extends Entry implements Readable, Writable { }

// Directory implements Container
public class Directory extends Entry implements Container { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Current Implementation:**

```java
// FileSystem depends on concrete Directory
public class FileSystem {
    private final Directory root;
}
```

**Better with DIP:**

```java
// Define abstraction
public interface DirectoryOperations {
    void addChild(Entry entry);
    Entry removeChild(String name);
    Entry getChild(String name);
    List<Entry> getChildren();
}

// FileSystem depends on abstraction
public class FileSystem {
    private final DirectoryOperations root;
    
    public FileSystem(DirectoryOperations root) {
        this.root = root;
    }
}
```

---

## Design Patterns Used

### 1. Composite Pattern

**Where:** Entry, File, Directory hierarchy

```
       Entry (Component)
          △
    ┌─────┴─────┐
    │           │
  File       Directory
 (Leaf)    (Composite)
              │
              │ contains
              ▼
           Entry*
```

**Implementation:**

```java
public abstract class Entry {  // Component
    public abstract long getSize();
}

public class File extends Entry {  // Leaf
    @Override
    public long getSize() {
        return content.length;  // Simple size
    }
}

public class Directory extends Entry {  // Composite
    private Map<String, Entry> children;
    
    @Override
    public long getSize() {
        // Recursive size calculation
        return children.values().stream()
            .mapToLong(Entry::getSize)
            .sum();
    }
}
```

**Benefits:**
- Uniform treatment of files and directories
- Recursive operations (size, search) work naturally
- Easy to add new entry types

---

### 2. Builder Pattern

**Where:** SearchCriteria

```java
SearchCriteria criteria = SearchCriteria.builder()
    .nameContains("report")
    .extension("pdf")
    .minSize(1000)
    .filesOnly()
    .build();
```

**Benefits:**
- Flexible, readable construction
- Optional parameters without constructor overloading
- Immutable result

---

### 3. Template Method Pattern (Implicit)

**Where:** Entry base class

```java
public abstract class Entry {
    // Template method for getting path
    public String getPath() {
        if (parent == null) {
            return "/";
        }
        String parentPath = parent.getPath();
        if (parentPath.equals("/")) {
            return "/" + name;
        }
        return parentPath + "/" + name;
    }
    
    // Abstract methods for subclasses to implement
    public abstract long getSize();
    public abstract boolean isDirectory();
}
```

---

### 4. Iterator Pattern (Implicit)

**Where:** Directory traversal

```java
// External iteration
for (Entry child : directory.getChildren()) {
    process(child);
}

// Internal iteration with recursion
private void findRecursive(Directory dir, Predicate<Entry> predicate, 
                           List<Entry> results) {
    for (Entry child : dir.getChildren()) {
        if (predicate.test(child)) {
            results.add(child);
        }
        if (child.isDirectory()) {
            findRecursive((Directory) child, predicate, results);
        }
    }
}
```

---

## Why Alternatives Were Rejected

### Alternative 1: Single FileSystemEntry Class

```java
// Rejected
public class FileSystemEntry {
    private EntryType type;  // FILE or DIRECTORY
    private byte[] content;  // Only for files
    private Map<String, FileSystemEntry> children;  // Only for directories
}
```

**Why rejected:**
- Violates SRP (handles both file and directory logic)
- Wastes memory (files have unused children map)
- Complex conditionals everywhere

---

### Alternative 2: Path as String Throughout

```java
// Rejected
public class FileSystem {
    private Map<String, byte[]> files;  // path -> content
    
    public void write(String path, byte[] content) {
        files.put(path, content);
    }
}
```

**Why rejected:**
- No directory structure
- Can't list directory contents efficiently
- No metadata (permissions, timestamps)
- Path parsing on every operation

---

### Alternative 3: Separate File and Directory Collections

```java
// Rejected
public class FileSystem {
    private Map<String, File> files;
    private Map<String, Directory> directories;
}
```

**Why rejected:**
- Duplicate path tracking
- Hard to maintain consistency
- Can't treat files and directories uniformly

---

## What Would Break Without Each Class

| Class | What Breaks |
|-------|-------------|
| `Entry` | No common interface for files and directories |
| `File` | Can't store file content |
| `Directory` | Can't organize files hierarchically |
| `Permissions` | No access control |
| `SearchCriteria` | Hard to build flexible searches |
| `FileSystem` | No unified interface for operations |

---

## Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `mkdir(path)` | O(d) | d = depth of path |
| `touch(path)` | O(d) | d = depth of path |
| `rm(path)` | O(d) | d = depth of path |
| `mv(src, dest)` | O(d) | d = max path depth |
| `cp(src, dest)` | O(n) | n = total entries to copy |
| `ls(path)` | O(d + c) | d = depth, c = children count |
| `find(criteria)` | O(n) | n = total entries |
| `getSize(dir)` | O(n) | n = entries in subtree |

### Space Complexity

| Component | Space |
|-----------|-------|
| Directory | O(c) per directory | c = children count |
| File | O(s) per file | s = content size |
| Path resolution | O(d) | d = path depth |
| Find results | O(m) | m = matching entries |

---

## Interview Follow-ups

### Q1: How would you implement file versioning?

```java
public class VersionedFile extends File {
    private final List<Version> versions = new ArrayList<>();
    private int currentVersion = 0;
    
    @Override
    public void write(byte[] content) {
        // Save current content as version
        versions.add(new Version(currentVersion++, this.content, LocalDateTime.now()));
        super.write(content);
    }
    
    public byte[] getVersion(int versionNumber) {
        return versions.stream()
            .filter(v -> v.number == versionNumber)
            .findFirst()
            .map(v -> v.content)
            .orElse(null);
    }
    
    public void rollback(int versionNumber) {
        byte[] oldContent = getVersion(versionNumber);
        if (oldContent != null) {
            this.content = oldContent;
        }
    }
}
```

### Q2: How would you implement file locking?

```java
public class LockableFile extends File {
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private String lockedBy;
    
    public boolean acquireLock(String userId) {
        if (lock.writeLock().tryLock()) {
            lockedBy = userId;
            return true;
        }
        return false;
    }
    
    public void releaseLock(String userId) {
        if (userId.equals(lockedBy)) {
            lockedBy = null;
            lock.writeLock().unlock();
        }
    }
    
    @Override
    public void write(byte[] content) {
        if (!lock.isWriteLockedByCurrentThread()) {
            throw new IllegalStateException("Must acquire lock before writing");
        }
        super.write(content);
    }
}
```

### Q3: How would you implement disk quotas?

```java
public class QuotaManager {
    private final Map<String, Long> userQuotas;  // userId -> max bytes
    private final Map<String, Long> userUsage;   // userId -> current bytes
    
    public boolean canWrite(String userId, long bytes) {
        long quota = userQuotas.getOrDefault(userId, Long.MAX_VALUE);
        long usage = userUsage.getOrDefault(userId, 0L);
        return usage + bytes <= quota;
    }
    
    public void recordUsage(String userId, long bytes) {
        userUsage.merge(userId, bytes, Long::sum);
    }
}

public class QuotaAwareFileSystem extends FileSystem {
    private final QuotaManager quotaManager;
    
    @Override
    public File createFile(String path, String content, String userId) {
        long size = content.getBytes().length;
        if (!quotaManager.canWrite(userId, size)) {
            throw new QuotaExceededException("Disk quota exceeded");
        }
        File file = super.createFile(path, content);
        quotaManager.recordUsage(userId, size);
        return file;
    }
}
```

### Q4: How would you persist to actual disk?

```java
public class PersistentFileSystem extends FileSystem {
    private final Path rootPath;
    
    public PersistentFileSystem(Path rootPath) {
        this.rootPath = rootPath;
        loadFromDisk();
    }
    
    @Override
    public File touch(String virtualPath) {
        File file = super.touch(virtualPath);
        
        // Sync to disk
        Path physicalPath = rootPath.resolve(virtualPath.substring(1));
        try {
            Files.createDirectories(physicalPath.getParent());
            Files.createFile(physicalPath);
        } catch (IOException e) {
            throw new RuntimeException("Failed to create file on disk", e);
        }
        
        return file;
    }
    
    private void loadFromDisk() {
        // Walk physical directory and build in-memory structure
        try {
            Files.walk(rootPath).forEach(path -> {
                String virtualPath = "/" + rootPath.relativize(path).toString();
                if (Files.isDirectory(path)) {
                    mkdir(virtualPath);
                } else {
                    touch(virtualPath);
                }
            });
        } catch (IOException e) {
            throw new RuntimeException("Failed to load from disk", e);
        }
    }
}
```

