# 📁 File System - Code Walkthrough

## Building From Scratch: Step-by-Step

### Phase 1: Design the Entry Hierarchy

```java
// Step 1: Entry abstract class
public abstract class Entry {
    protected String name;
    protected Directory parent;
    protected LocalDateTime createdAt;
    protected LocalDateTime modifiedAt;
    protected Permissions permissions;
    
    public abstract long getSize();
    public abstract boolean isDirectory();
}
```

**Why abstract Entry?**
- Common properties for all entries (name, parent, timestamps)
- Polymorphism: treat files and directories uniformly
- Template for path calculation

```java
// Step 2: getPath() implementation
public String getPath() {
    if (parent == null) {
        return "/";  // Root directory
    }
    
    String parentPath = parent.getPath();
    if (parentPath.equals("/")) {
        return "/" + name;
    }
    return parentPath + "/" + name;
}
```

**Path calculation trace:**

```
For file at: /home/user/file.txt

file.getPath()
  └── parent.getPath()  (parent = user)
        └── parent.getPath()  (parent = home)
              └── parent.getPath()  (parent = root)
                    └── return "/"
              └── return "/" + "home" = "/home"
        └── return "/home" + "/" + "user" = "/home/user"
  └── return "/home/user" + "/" + "file.txt" = "/home/user/file.txt"
```

---

### Phase 2: Implement File Class

```java
// Step 3: File class
public class File extends Entry {
    private byte[] content;
    private String extension;
    
    public File(String name, Directory parent) {
        super(name, parent);
        this.content = new byte[0];
        this.extension = extractExtension(name);
    }
```

**Why store extension separately?**
- Frequent search by extension
- Avoid parsing name every time
- Extracted once at creation

```java
    public byte[] read() {
        if (!permissions.isReadable()) {
            throw new SecurityException("Permission denied");
        }
        return content.clone();  // Return copy for safety
    }
    
    public void write(byte[] newContent) {
        if (!permissions.isWritable()) {
            throw new SecurityException("Permission denied");
        }
        this.content = newContent != null ? newContent.clone() : new byte[0];
        touch();  // Update modified time
    }
```

**Why clone content?**
- Prevents external modification of internal state
- Caller can't corrupt file by modifying returned array
- Defensive programming

---

### Phase 3: Implement Directory Class

```java
// Step 4: Directory class
public class Directory extends Entry {
    private final Map<String, Entry> children;
    
    public Directory(String name, Directory parent) {
        super(name, parent);
        this.children = new LinkedHashMap<>();  // Maintains insertion order
    }
```

**Why LinkedHashMap?**
- O(1) lookup by name
- Maintains insertion order for ls() output
- Predictable iteration

```java
    public void addChild(Entry entry) {
        if (!permissions.isWritable()) {
            throw new SecurityException("Permission denied");
        }
        
        if (children.containsKey(entry.getName())) {
            throw new IllegalArgumentException("Entry already exists");
        }
        
        children.put(entry.getName(), entry);
        entry.setParent(this);
        touch();
    }
```

**addChild flow:**

```
addChild(file)
    │
    ├── Check permissions
    │       │ Not writable
    │       └──► SecurityException
    │
    ├── Check name conflict
    │       │ Name exists
    │       └──► IllegalArgumentException
    │
    ├── Add to children map
    │
    ├── Set file's parent to this
    │
    └── Update modified time
```

```java
    @Override
    public long getSize() {
        return children.values().stream()
            .mapToLong(Entry::getSize)
            .sum();
    }
```

**Recursive size calculation:**

```
/home (getSize())
├── user1/ (getSize() = 150)
│   ├── file1.txt (50 bytes)
│   └── file2.txt (100 bytes)
└── user2/ (getSize() = 200)
    └── data.csv (200 bytes)

/home.getSize() = 150 + 200 = 350 bytes
```

---

### Phase 4: Implement FileSystem

```java
// Step 5: FileSystem class
public class FileSystem {
    private final Directory root;
    private Directory currentDirectory;
    
    public FileSystem() {
        this.root = Directory.createRoot();
        this.currentDirectory = root;
    }
```

**Root directory special case:**

```java
public static Directory createRoot() {
    return new Directory("", null) {
        @Override
        public String getPath() {
            return "/";
        }
        
        @Override
        public String getName() {
            return "/";
        }
    };
}
```

---

### Phase 5: Path Resolution

```java
// Step 6: resolvePath method
public Entry resolvePath(String path) {
    if (path == null || path.isEmpty() || path.equals(".")) {
        return currentDirectory;
    }
    
    String[] parts = path.split("/");
    Directory current = path.startsWith("/") ? root : currentDirectory;
    
    for (String part : parts) {
        if (part.isEmpty() || part.equals(".")) {
            continue;
        }
        
        if (part.equals("..")) {
            if (current.getParent() != null) {
                current = current.getParent();
            }
            continue;
        }
        
        Entry child = current.getChild(part);
        if (child == null) {
            return null;
        }
        
        if (child.isDirectory()) {
            current = (Directory) child;
        } else {
            return child;  // File found, must be last part
        }
    }
    
    return current;
}
```

**Path resolution examples:**

```
Current directory: /home/user

resolvePath("/etc/config")
  parts = ["", "etc", "config"]
  start = root (absolute path)
  "" → skip
  "etc" → root.getChild("etc") → etc directory
  "config" → etc.getChild("config") → config file
  return config file

resolvePath("documents/file.txt")
  parts = ["documents", "file.txt"]
  start = currentDirectory (/home/user)
  "documents" → user.getChild("documents") → documents directory
  "file.txt" → documents.getChild("file.txt") → file
  return file

resolvePath("../user2")
  parts = ["..", "user2"]
  start = currentDirectory (/home/user)
  ".." → user.getParent() → home
  "user2" → home.getChild("user2") → user2 directory
  return user2 directory
```

---

### Phase 6: File Operations

```java
// Step 7: mkdir implementation
public Directory mkdir(String path) {
    String[] parts = parsePath(path);
    Directory current = getStartDirectory(path);
    
    for (String part : parts) {
        if (part.isEmpty()) continue;
        
        Entry child = current.getChild(part);
        
        if (child == null) {
            // Create new directory
            Directory newDir = new Directory(part, current);
            current.addChild(newDir);
            current = newDir;
        } else if (child.isDirectory()) {
            // Directory exists, continue
            current = (Directory) child;
        } else {
            // File exists with same name
            throw new IllegalArgumentException(part + " is a file");
        }
    }
    
    return current;
}
```

**mkdir trace for "/home/user/documents":**

```
Initial: root = /

Step 1: part = "home"
  current.getChild("home") → null
  Create Directory("home", root)
  root.addChild(home)
  current = home

Step 2: part = "user"
  current.getChild("user") → null
  Create Directory("user", home)
  home.addChild(user)
  current = user

Step 3: part = "documents"
  current.getChild("documents") → null
  Create Directory("documents", user)
  user.addChild(documents)
  current = documents

Return: documents

Tree:
/
└── home/
    └── user/
        └── documents/
```

```java
// Step 8: touch implementation
public File touch(String path) {
    int lastSlash = path.lastIndexOf('/');
    String dirPath = lastSlash > 0 ? path.substring(0, lastSlash) : 
                    (path.startsWith("/") ? "/" : ".");
    String fileName = lastSlash >= 0 ? path.substring(lastSlash + 1) : path;
    
    // Ensure parent directory exists
    Directory parent = mkdir(dirPath);
    
    // Check if file exists
    Entry existing = parent.getChild(fileName);
    if (existing != null) {
        if (existing.isFile()) {
            existing.touch();  // Update timestamp
            return (File) existing;
        } else {
            throw new IllegalArgumentException(fileName + " is a directory");
        }
    }
    
    // Create new file
    File file = new File(fileName, parent);
    parent.addChild(file);
    return file;
}
```

---

### Phase 7: Search Implementation

```java
// Step 9: find with criteria
public List<Entry> find(SearchCriteria criteria) {
    List<Entry> results = new ArrayList<>();
    findRecursive(root, criteria.toPredicate(), results);
    return results;
}

private void findRecursive(Directory dir, Predicate<Entry> predicate, 
                           List<Entry> results) {
    for (Entry child : dir.getChildren()) {
        // Check if entry matches criteria
        if (predicate.test(child)) {
            results.add(child);
        }
        
        // Recurse into subdirectories
        if (child.isDirectory()) {
            findRecursive((Directory) child, predicate, results);
        }
    }
}
```

**Search trace for find("log"):**

```
findRecursive(root, nameContains("log"), results)
  │
  ├── Check /home → "home" contains "log"? No
  │   └── Recurse into /home
  │       ├── Check /home/user → "user" contains "log"? No
  │       │   └── Recurse into /home/user
  │       │       └── (files checked, none match)
  │       └── ...
  │
  └── Check /var → "var" contains "log"? No
      └── Recurse into /var
          └── Check /var/log → "log" contains "log"? Yes! Add to results
              └── Recurse into /var/log
                  └── Check system.log → "system.log" contains "log"? Yes! Add

Results: [/var/log, /var/log/system.log]
```

---

## Testing Approach

### Unit Tests

```java
// FileTest.java
public class FileTest {
    
    @Test
    void testReadWrite() {
        Directory parent = new Directory("test", null);
        File file = new File("test.txt", parent);
        
        file.write("Hello World".getBytes());
        
        assertEquals("Hello World", new String(file.read()));
        assertEquals(11, file.getSize());
    }
    
    @Test
    void testExtensionExtraction() {
        Directory parent = new Directory("test", null);
        
        assertEquals("txt", new File("file.txt", parent).getExtension());
        assertEquals("pdf", new File("report.pdf", parent).getExtension());
        assertEquals("", new File("noextension", parent).getExtension());
        assertEquals("gz", new File("archive.tar.gz", parent).getExtension());
    }
    
    @Test
    void testPermissions() {
        Directory parent = new Directory("test", null);
        File file = new File("readonly.txt", parent);
        file.setPermissions(Permissions.readOnly());
        
        assertDoesNotThrow(() -> file.read());
        assertThrows(SecurityException.class, () -> file.write("data".getBytes()));
    }
}
```

```java
// DirectoryTest.java
public class DirectoryTest {
    
    @Test
    void testAddChild() {
        Directory parent = new Directory("parent", null);
        File child = new File("child.txt", null);
        
        parent.addChild(child);
        
        assertEquals(1, parent.getChildCount());
        assertEquals(parent, child.getParent());
        assertSame(child, parent.getChild("child.txt"));
    }
    
    @Test
    void testDuplicateChildRejected() {
        Directory parent = new Directory("parent", null);
        parent.addChild(new File("file.txt", null));
        
        assertThrows(IllegalArgumentException.class, () -> 
            parent.addChild(new File("file.txt", null)));
    }
    
    @Test
    void testRecursiveSize() {
        Directory root = new Directory("root", null);
        Directory sub = new Directory("sub", root);
        root.addChild(sub);
        
        File file1 = new File("f1.txt", root);
        file1.write(new byte[100]);
        root.addChild(file1);
        
        File file2 = new File("f2.txt", sub);
        file2.write(new byte[50]);
        sub.addChild(file2);
        
        assertEquals(50, sub.getSize());
        assertEquals(150, root.getSize());
    }
}
```

```java
// FileSystemTest.java
public class FileSystemTest {
    
    private FileSystem fs;
    
    @BeforeEach
    void setUp() {
        fs = new FileSystem();
    }
    
    @Test
    void testMkdirCreatesPath() {
        fs.mkdir("/a/b/c");
        
        assertNotNull(fs.resolvePath("/a"));
        assertNotNull(fs.resolvePath("/a/b"));
        assertNotNull(fs.resolvePath("/a/b/c"));
    }
    
    @Test
    void testCdAndPwd() {
        fs.mkdir("/home/user");
        
        assertEquals("/", fs.pwd());
        
        fs.cd("/home/user");
        assertEquals("/home/user", fs.pwd());
        
        fs.cd("..");
        assertEquals("/home", fs.pwd());
    }
    
    @Test
    void testMoveFile() {
        fs.mkdir("/src");
        fs.mkdir("/dest");
        fs.createFile("/src/file.txt", "content");
        
        assertTrue(fs.mv("/src/file.txt", "/dest/file.txt"));
        
        assertNull(fs.resolvePath("/src/file.txt"));
        assertNotNull(fs.resolvePath("/dest/file.txt"));
    }
    
    @Test
    void testCopyFile() {
        fs.createFile("/original.txt", "content");
        
        assertTrue(fs.cp("/original.txt", "/copy.txt"));
        
        File original = (File) fs.resolvePath("/original.txt");
        File copy = (File) fs.resolvePath("/copy.txt");
        
        assertNotNull(original);
        assertNotNull(copy);
        assertEquals(original.readAsString(), copy.readAsString());
        assertNotSame(original, copy);
    }
    
    @Test
    void testFindByExtension() {
        fs.createFile("/a.txt", "");
        fs.createFile("/b.txt", "");
        fs.createFile("/c.pdf", "");
        fs.mkdir("/sub");
        fs.createFile("/sub/d.txt", "");
        
        List<File> txtFiles = fs.findByExtension("txt");
        
        assertEquals(3, txtFiles.size());
    }
}
```

---

## Complexity Analysis

### mkdir("/a/b/c/d")

```
Time: O(d) where d = depth
  - Split path: O(d)
  - For each part:
    - getChild(): O(1) HashMap lookup
    - addChild(): O(1) HashMap put

Space: O(d)
  - Path parts array: O(d)
  - New directories: O(d)
```

### find(criteria)

```
Time: O(n) where n = total entries
  - Visit every entry once
  - Predicate test: O(1) per entry

Space: O(h + m)
  - h = max tree height (recursion stack)
  - m = matching entries (results list)
```

### getSize() on directory

```
Time: O(n) where n = entries in subtree
  - Visit every descendant once
  - Sum sizes

Space: O(h)
  - h = max tree height (recursion stack)
```

---

## Interview Follow-ups with Answers

### Q1: How would you handle concurrent access?

```java
public class ConcurrentFileSystem {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final FileSystem fs = new FileSystem();
    
    public File touch(String path) {
        lock.writeLock().lock();
        try {
            return fs.touch(path);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public List<Entry> ls(String path) {
        lock.readLock().lock();
        try {
            return fs.ls(path);
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

### Q2: How would you implement undo/redo?

```java
public class UndoableFileSystem {
    private final FileSystem fs;
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();
    
    public void mkdir(String path) {
        fs.mkdir(path);
        undoStack.push(new DeleteCommand(path));
        redoStack.clear();
    }
    
    public void undo() {
        if (!undoStack.isEmpty()) {
            Command cmd = undoStack.pop();
            cmd.execute(fs);
            redoStack.push(cmd.inverse());
        }
    }
    
    public void redo() {
        if (!redoStack.isEmpty()) {
            Command cmd = redoStack.pop();
            cmd.execute(fs);
            undoStack.push(cmd.inverse());
        }
    }
}
```

### Q3: How would you implement watch/notify?

```java
public class WatchableFileSystem extends FileSystem {
    private final Map<String, List<FileWatcher>> watchers = new HashMap<>();
    
    public void watch(String path, FileWatcher watcher) {
        watchers.computeIfAbsent(path, k -> new ArrayList<>()).add(watcher);
    }
    
    @Override
    public File touch(String path) {
        File file = super.touch(path);
        notifyWatchers(path, FileEvent.CREATED);
        return file;
    }
    
    private void notifyWatchers(String path, FileEvent event) {
        // Notify watchers for this path and all parent paths
        String current = path;
        while (current != null && !current.isEmpty()) {
            List<FileWatcher> pathWatchers = watchers.get(current);
            if (pathWatchers != null) {
                for (FileWatcher watcher : pathWatchers) {
                    watcher.onEvent(path, event);
                }
            }
            int lastSlash = current.lastIndexOf('/');
            current = lastSlash > 0 ? current.substring(0, lastSlash) : null;
        }
    }
}

public interface FileWatcher {
    void onEvent(String path, FileEvent event);
}

public enum FileEvent {
    CREATED, MODIFIED, DELETED, MOVED
}
```

### Q4: What would you do differently with more time?

1. **Add hard/soft links** - Link entries pointing to other entries
2. **Add file attributes** - Custom metadata key-value pairs
3. **Add journaling** - Transaction log for crash recovery
4. **Add compression** - Transparent file compression
5. **Add encryption** - File-level encryption
6. **Add access control lists** - Fine-grained permissions per user

