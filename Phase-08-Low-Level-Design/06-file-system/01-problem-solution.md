# 📁 File System - Complete Solution

## Problem Statement

Design an in-memory file system that can:
- Support directory tree structure
- Perform file operations (create, delete, move, copy)
- Search files by name and extension
- Calculate directory sizes
- Handle file/directory permissions
- Support path navigation (absolute and relative)

---

## STEP 1: Complete Reference Solution (Answer Key)

### Class Diagram Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              FILE SYSTEM                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                         FileSystem                                        │   │
│  │                                                                           │   │
│  │  - root: Directory                                                        │   │
│  │  - currentDirectory: Directory                                            │   │
│  │                                                                           │   │
│  │  + mkdir(path): Directory                                                 │   │
│  │  + touch(path): File                                                      │   │
│  │  + rm(path): boolean                                                      │   │
│  │  + mv(src, dest): boolean                                                 │   │
│  │  + cp(src, dest): boolean                                                 │   │
│  │  + ls(path): List<Entry>                                                  │   │
│  │  + cd(path): boolean                                                      │   │
│  │  + find(name): List<Entry>                                                │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          │                                                       │
│                          │ manages                                               │
│                          ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                      Entry (abstract)                                     │   │
│  │                                                                           │   │
│  │  - name: String                                                           │   │
│  │  - parent: Directory                                                      │   │
│  │  - createdAt: LocalDateTime                                               │   │
│  │  - modifiedAt: LocalDateTime                                              │   │
│  │  - permissions: Permissions                                               │   │
│  │                                                                           │   │
│  │  + getPath(): String                                                      │   │
│  │  + getSize(): long (abstract)                                             │   │
│  │  + isDirectory(): boolean (abstract)                                      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                          △                                                       │
│           ┌──────────────┴──────────────┐                                       │
│           │                             │                                       │
│  ┌────────┴────────┐          ┌────────┴────────┐                              │
│  │      File       │          │    Directory    │                              │
│  │                 │          │                 │                              │
│  │  - content: byte[]         │  - children: Map│                              │
│  │  - extension: String       │                 │                              │
│  │                 │          │  + addChild()   │                              │
│  │  + read(): byte[]          │  + removeChild()│                              │
│  │  + write(byte[])│          │  + getChild()   │                              │
│  │  + getSize()    │          │  + getSize()    │                              │
│  └─────────────────┘          └─────────────────┘                              │
│                                                                                  │
│  ┌──────────────────┐         ┌──────────────────┐                              │
│  │   Permissions    │         │   SearchCriteria │                              │
│  │                  │         │                  │                              │
│  │  - read: boolean │         │  - name: String  │                              │
│  │  - write: boolean│         │  - extension: String                            │
│  │  - execute: bool │         │  - minSize: long │                              │
│  └──────────────────┘         └──────────────────┘                              │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Directory Tree Visualization

```
/ (root)
├── home/
│   ├── user1/
│   │   ├── documents/
│   │   │   ├── report.pdf
│   │   │   └── notes.txt
│   │   └── pictures/
│   │       └── photo.jpg
│   └── user2/
│       └── data.csv
├── etc/
│   └── config.ini
└── var/
    └── log/
        └── system.log
```

---

## STEP 2: Complete Java Implementation

### 2.1 Permissions Class

```java
// Permissions.java
package com.filesystem;

/**
 * Represents file/directory permissions.
 * 
 * Similar to Unix permissions (rwx).
 */
public class Permissions {
    
    private boolean readable;
    private boolean writable;
    private boolean executable;
    
    public Permissions() {
        // Default: read and write, no execute
        this(true, true, false);
    }
    
    public Permissions(boolean readable, boolean writable, boolean executable) {
        this.readable = readable;
        this.writable = writable;
        this.executable = executable;
    }
    
    // Factory methods for common permission sets
    public static Permissions readOnly() {
        return new Permissions(true, false, false);
    }
    
    public static Permissions readWrite() {
        return new Permissions(true, true, false);
    }
    
    public static Permissions all() {
        return new Permissions(true, true, true);
    }
    
    public static Permissions none() {
        return new Permissions(false, false, false);
    }
    
    // Getters and setters
    public boolean isReadable() { return readable; }
    public boolean isWritable() { return writable; }
    public boolean isExecutable() { return executable; }
    
    public void setReadable(boolean readable) { this.readable = readable; }
    public void setWritable(boolean writable) { this.writable = writable; }
    public void setExecutable(boolean executable) { this.executable = executable; }
    
    /**
     * Returns Unix-style permission string (e.g., "rwx", "r--", "rw-").
     */
    @Override
    public String toString() {
        return (readable ? "r" : "-") +
               (writable ? "w" : "-") +
               (executable ? "x" : "-");
    }
    
    /**
     * Creates a copy of these permissions.
     */
    public Permissions copy() {
        return new Permissions(readable, writable, executable);
    }
}
```

### 2.2 Entry Abstract Class

```java
// Entry.java
package com.filesystem;

import java.time.LocalDateTime;

/**
 * Abstract base class for file system entries (files and directories).
 * 
 * COMPOSITE PATTERN: Entry is the component interface.
 * File is a leaf, Directory is a composite.
 */
public abstract class Entry {
    
    protected String name;
    protected Directory parent;
    protected LocalDateTime createdAt;
    protected LocalDateTime modifiedAt;
    protected Permissions permissions;
    
    protected Entry(String name, Directory parent) {
        validateName(name);
        this.name = name;
        this.parent = parent;
        this.createdAt = LocalDateTime.now();
        this.modifiedAt = LocalDateTime.now();
        this.permissions = new Permissions();
    }
    
    /**
     * Validates entry name.
     */
    private void validateName(String name) {
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException("Name cannot be null or empty");
        }
        if (name.contains("/") || name.contains("\\")) {
            throw new IllegalArgumentException("Name cannot contain path separators");
        }
        if (name.equals(".") || name.equals("..")) {
            throw new IllegalArgumentException("Name cannot be '.' or '..'");
        }
    }
    
    /**
     * Gets the full path from root.
     */
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
    
    /**
     * Gets the size of this entry.
     * Files return content size, directories return total size of contents.
     */
    public abstract long getSize();
    
    /**
     * Checks if this entry is a directory.
     */
    public abstract boolean isDirectory();
    
    /**
     * Checks if this entry is a file.
     */
    public boolean isFile() {
        return !isDirectory();
    }
    
    /**
     * Updates the modified timestamp.
     */
    protected void touch() {
        this.modifiedAt = LocalDateTime.now();
    }
    
    /**
     * Renames this entry.
     */
    public void rename(String newName) {
        validateName(newName);
        
        if (parent != null && parent.hasChild(newName)) {
            throw new IllegalArgumentException("Entry with name '" + newName + "' already exists");
        }
        
        String oldName = this.name;
        this.name = newName;
        
        // Update parent's children map
        if (parent != null) {
            parent.updateChildName(oldName, newName, this);
        }
        
        touch();
    }
    
    // Getters
    public String getName() { return name; }
    public Directory getParent() { return parent; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getModifiedAt() { return modifiedAt; }
    public Permissions getPermissions() { return permissions; }
    
    // Setters
    public void setPermissions(Permissions permissions) {
        this.permissions = permissions;
    }
    
    protected void setParent(Directory parent) {
        this.parent = parent;
    }
    
    @Override
    public String toString() {
        return String.format("%s %s %s",
                permissions,
                isDirectory() ? "d" : "-",
                name);
    }
}
```

### 2.3 File Class

```java
// File.java
package com.filesystem;

/**
 * Represents a file in the file system.
 * 
 * Files store content as byte array and have an extension.
 */
public class File extends Entry {
    
    private byte[] content;
    private String extension;
    
    public File(String name, Directory parent) {
        super(name, parent);
        this.content = new byte[0];
        this.extension = extractExtension(name);
    }
    
    public File(String name, Directory parent, byte[] content) {
        super(name, parent);
        this.content = content != null ? content.clone() : new byte[0];
        this.extension = extractExtension(name);
    }
    
    /**
     * Extracts file extension from name.
     */
    private String extractExtension(String name) {
        int lastDot = name.lastIndexOf('.');
        if (lastDot > 0 && lastDot < name.length() - 1) {
            return name.substring(lastDot + 1).toLowerCase();
        }
        return "";
    }
    
    /**
     * Reads the file content.
     */
    public byte[] read() {
        if (!permissions.isReadable()) {
            throw new SecurityException("Permission denied: cannot read " + getPath());
        }
        return content.clone();
    }
    
    /**
     * Reads the file content as string.
     */
    public String readAsString() {
        return new String(read());
    }
    
    /**
     * Writes content to the file.
     */
    public void write(byte[] newContent) {
        if (!permissions.isWritable()) {
            throw new SecurityException("Permission denied: cannot write to " + getPath());
        }
        this.content = newContent != null ? newContent.clone() : new byte[0];
        touch();
    }
    
    /**
     * Writes string content to the file.
     */
    public void write(String text) {
        write(text.getBytes());
    }
    
    /**
     * Appends content to the file.
     */
    public void append(byte[] additionalContent) {
        if (!permissions.isWritable()) {
            throw new SecurityException("Permission denied: cannot write to " + getPath());
        }
        
        byte[] newContent = new byte[content.length + additionalContent.length];
        System.arraycopy(content, 0, newContent, 0, content.length);
        System.arraycopy(additionalContent, 0, newContent, content.length, additionalContent.length);
        this.content = newContent;
        touch();
    }
    
    /**
     * Appends string content to the file.
     */
    public void append(String text) {
        append(text.getBytes());
    }
    
    @Override
    public long getSize() {
        return content.length;
    }
    
    @Override
    public boolean isDirectory() {
        return false;
    }
    
    public String getExtension() {
        return extension;
    }
    
    /**
     * Creates a copy of this file.
     */
    public File copy(String newName, Directory newParent) {
        File copy = new File(newName, newParent, this.content);
        copy.permissions = this.permissions.copy();
        return copy;
    }
    
    @Override
    public String toString() {
        return String.format("%s %s %8d %s",
                permissions,
                "-",
                getSize(),
                name);
    }
}
```

### 2.4 Directory Class

```java
// Directory.java
package com.filesystem;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Represents a directory in the file system.
 * 
 * Directories contain children (files and other directories).
 * COMPOSITE PATTERN: Directory is the composite.
 */
public class Directory extends Entry {
    
    private final Map<String, Entry> children;
    
    public Directory(String name, Directory parent) {
        super(name, parent);
        this.children = new LinkedHashMap<>();  // Maintains insertion order
    }
    
    /**
     * Creates the root directory.
     */
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
    
    /**
     * Adds a child entry to this directory.
     */
    public void addChild(Entry entry) {
        if (!permissions.isWritable()) {
            throw new SecurityException("Permission denied: cannot write to " + getPath());
        }
        
        if (children.containsKey(entry.getName())) {
            throw new IllegalArgumentException(
                "Entry '" + entry.getName() + "' already exists in " + getPath());
        }
        
        children.put(entry.getName(), entry);
        entry.setParent(this);
        touch();
    }
    
    /**
     * Removes a child entry from this directory.
     */
    public Entry removeChild(String name) {
        if (!permissions.isWritable()) {
            throw new SecurityException("Permission denied: cannot write to " + getPath());
        }
        
        Entry removed = children.remove(name);
        if (removed != null) {
            removed.setParent(null);
            touch();
        }
        return removed;
    }
    
    /**
     * Gets a child by name.
     */
    public Entry getChild(String name) {
        if (!permissions.isReadable()) {
            throw new SecurityException("Permission denied: cannot read " + getPath());
        }
        return children.get(name);
    }
    
    /**
     * Checks if a child with the given name exists.
     */
    public boolean hasChild(String name) {
        return children.containsKey(name);
    }
    
    /**
     * Gets all children.
     */
    public List<Entry> getChildren() {
        if (!permissions.isReadable()) {
            throw new SecurityException("Permission denied: cannot read " + getPath());
        }
        return new ArrayList<>(children.values());
    }
    
    /**
     * Gets only file children.
     */
    public List<File> getFiles() {
        return getChildren().stream()
                .filter(Entry::isFile)
                .map(e -> (File) e)
                .collect(Collectors.toList());
    }
    
    /**
     * Gets only directory children.
     */
    public List<Directory> getDirectories() {
        return getChildren().stream()
                .filter(Entry::isDirectory)
                .map(e -> (Directory) e)
                .collect(Collectors.toList());
    }
    
    /**
     * Gets the number of direct children.
     */
    public int getChildCount() {
        return children.size();
    }
    
    /**
     * Checks if directory is empty.
     */
    public boolean isEmpty() {
        return children.isEmpty();
    }
    
    /**
     * Updates child name in the map (called during rename).
     */
    protected void updateChildName(String oldName, String newName, Entry entry) {
        children.remove(oldName);
        children.put(newName, entry);
    }
    
    @Override
    public long getSize() {
        // Directory size is sum of all children sizes
        return children.values().stream()
                .mapToLong(Entry::getSize)
                .sum();
    }
    
    @Override
    public boolean isDirectory() {
        return true;
    }
    
    /**
     * Creates a copy of this directory (shallow - doesn't copy children).
     */
    public Directory copyShallow(String newName, Directory newParent) {
        Directory copy = new Directory(newName, newParent);
        copy.permissions = this.permissions.copy();
        return copy;
    }
    
    /**
     * Creates a deep copy of this directory (includes all children).
     */
    public Directory copyDeep(String newName, Directory newParent) {
        Directory copy = copyShallow(newName, newParent);
        
        for (Entry child : children.values()) {
            if (child.isFile()) {
                File fileCopy = ((File) child).copy(child.getName(), copy);
                copy.children.put(fileCopy.getName(), fileCopy);
            } else {
                Directory dirCopy = ((Directory) child).copyDeep(child.getName(), copy);
                copy.children.put(dirCopy.getName(), dirCopy);
            }
        }
        
        return copy;
    }
    
    @Override
    public String toString() {
        return String.format("%s %s %8d %s/",
                permissions,
                "d",
                getChildCount(),
                name.isEmpty() ? "/" : name);
    }
}
```

### 2.5 Search Criteria

```java
// SearchCriteria.java
package com.filesystem;

import java.util.function.Predicate;

/**
 * Criteria for searching files and directories.
 * 
 * Uses builder pattern for flexible search configuration.
 */
public class SearchCriteria {
    
    private String namePattern;
    private String extension;
    private Long minSize;
    private Long maxSize;
    private Boolean isDirectory;
    private Boolean isReadable;
    private Boolean isWritable;
    
    private SearchCriteria() {}
    
    public static Builder builder() {
        return new Builder();
    }
    
    /**
     * Converts criteria to a predicate for filtering.
     */
    public Predicate<Entry> toPredicate() {
        return entry -> {
            // Name pattern match
            if (namePattern != null && !entry.getName().toLowerCase()
                    .contains(namePattern.toLowerCase())) {
                return false;
            }
            
            // Extension match (files only)
            if (extension != null) {
                if (!entry.isFile()) return false;
                if (!((File) entry).getExtension().equalsIgnoreCase(extension)) {
                    return false;
                }
            }
            
            // Size constraints
            long size = entry.getSize();
            if (minSize != null && size < minSize) return false;
            if (maxSize != null && size > maxSize) return false;
            
            // Type constraint
            if (isDirectory != null && entry.isDirectory() != isDirectory) {
                return false;
            }
            
            // Permission constraints
            if (isReadable != null && entry.getPermissions().isReadable() != isReadable) {
                return false;
            }
            if (isWritable != null && entry.getPermissions().isWritable() != isWritable) {
                return false;
            }
            
            return true;
        };
    }
    
    public static class Builder {
        private final SearchCriteria criteria = new SearchCriteria();
        
        public Builder nameContains(String pattern) {
            criteria.namePattern = pattern;
            return this;
        }
        
        public Builder extension(String ext) {
            criteria.extension = ext.startsWith(".") ? ext.substring(1) : ext;
            return this;
        }
        
        public Builder minSize(long bytes) {
            criteria.minSize = bytes;
            return this;
        }
        
        public Builder maxSize(long bytes) {
            criteria.maxSize = bytes;
            return this;
        }
        
        public Builder filesOnly() {
            criteria.isDirectory = false;
            return this;
        }
        
        public Builder directoriesOnly() {
            criteria.isDirectory = true;
            return this;
        }
        
        public Builder readable() {
            criteria.isReadable = true;
            return this;
        }
        
        public Builder writable() {
            criteria.isWritable = true;
            return this;
        }
        
        public SearchCriteria build() {
            return criteria;
        }
    }
}
```

### 2.6 FileSystem Class

```java
// FileSystem.java
package com.filesystem;

import java.util.*;

/**
 * Main file system class that manages the directory tree.
 * 
 * Provides Unix-like commands: mkdir, touch, rm, mv, cp, ls, cd, find, pwd
 */
public class FileSystem {
    
    private final Directory root;
    private Directory currentDirectory;
    
    public FileSystem() {
        this.root = Directory.createRoot();
        this.currentDirectory = root;
    }
    
    // ==================== Navigation ====================
    
    /**
     * Gets the current working directory path.
     */
    public String pwd() {
        return currentDirectory.getPath();
    }
    
    /**
     * Changes the current directory.
     * 
     * @param path Absolute or relative path
     * @return true if successful
     */
    public boolean cd(String path) {
        Entry entry = resolvePath(path);
        
        if (entry == null) {
            System.out.println("cd: " + path + ": No such directory");
            return false;
        }
        
        if (!entry.isDirectory()) {
            System.out.println("cd: " + path + ": Not a directory");
            return false;
        }
        
        currentDirectory = (Directory) entry;
        return true;
    }
    
    // ==================== File/Directory Creation ====================
    
    /**
     * Creates a directory at the specified path.
     * Creates parent directories if they don't exist (like mkdir -p).
     */
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
                current = (Directory) child;
            } else {
                throw new IllegalArgumentException(
                    "Cannot create directory: " + part + " is a file");
            }
        }
        
        return current;
    }
    
    /**
     * Creates an empty file at the specified path.
     * Creates parent directories if needed.
     */
    public File touch(String path) {
        int lastSlash = path.lastIndexOf('/');
        String dirPath = lastSlash > 0 ? path.substring(0, lastSlash) : 
                        (path.startsWith("/") ? "/" : ".");
        String fileName = lastSlash >= 0 ? path.substring(lastSlash + 1) : path;
        
        // Ensure parent directory exists
        Directory parent;
        if (dirPath.equals(".")) {
            parent = currentDirectory;
        } else {
            parent = mkdir(dirPath);
        }
        
        // Check if file already exists
        Entry existing = parent.getChild(fileName);
        if (existing != null) {
            if (existing.isFile()) {
                existing.touch();
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
    
    /**
     * Creates a file with content.
     */
    public File createFile(String path, String content) {
        File file = touch(path);
        file.write(content);
        return file;
    }
    
    // ==================== File/Directory Deletion ====================
    
    /**
     * Removes a file or empty directory.
     */
    public boolean rm(String path) {
        Entry entry = resolvePath(path);
        
        if (entry == null) {
            System.out.println("rm: " + path + ": No such file or directory");
            return false;
        }
        
        if (entry == root) {
            System.out.println("rm: cannot remove root directory");
            return false;
        }
        
        if (entry.isDirectory() && !((Directory) entry).isEmpty()) {
            System.out.println("rm: " + path + ": Directory not empty (use rmdir -r)");
            return false;
        }
        
        entry.getParent().removeChild(entry.getName());
        return true;
    }
    
    /**
     * Removes a directory and all its contents recursively.
     */
    public boolean rmRecursive(String path) {
        Entry entry = resolvePath(path);
        
        if (entry == null) {
            System.out.println("rm: " + path + ": No such file or directory");
            return false;
        }
        
        if (entry == root) {
            System.out.println("rm: cannot remove root directory");
            return false;
        }
        
        // Check if trying to remove current directory or its parent
        Directory check = currentDirectory;
        while (check != null) {
            if (check == entry) {
                System.out.println("rm: cannot remove current directory or ancestor");
                return false;
            }
            check = check.getParent();
        }
        
        entry.getParent().removeChild(entry.getName());
        return true;
    }
    
    // ==================== Move and Copy ====================
    
    /**
     * Moves a file or directory to a new location.
     */
    public boolean mv(String sourcePath, String destPath) {
        Entry source = resolvePath(sourcePath);
        
        if (source == null) {
            System.out.println("mv: " + sourcePath + ": No such file or directory");
            return false;
        }
        
        if (source == root) {
            System.out.println("mv: cannot move root directory");
            return false;
        }
        
        // Determine destination
        Entry destEntry = resolvePath(destPath);
        Directory destDir;
        String newName;
        
        if (destEntry != null && destEntry.isDirectory()) {
            // Moving into existing directory
            destDir = (Directory) destEntry;
            newName = source.getName();
        } else {
            // Moving to new path (possibly with rename)
            int lastSlash = destPath.lastIndexOf('/');
            String destDirPath = lastSlash > 0 ? destPath.substring(0, lastSlash) : 
                                (destPath.startsWith("/") ? "/" : ".");
            newName = lastSlash >= 0 ? destPath.substring(lastSlash + 1) : destPath;
            
            Entry destDirEntry = resolvePath(destDirPath);
            if (destDirEntry == null || !destDirEntry.isDirectory()) {
                System.out.println("mv: " + destDirPath + ": No such directory");
                return false;
            }
            destDir = (Directory) destDirEntry;
        }
        
        // Check for name conflict
        if (destDir.hasChild(newName)) {
            System.out.println("mv: " + newName + ": already exists in destination");
            return false;
        }
        
        // Perform move
        source.getParent().removeChild(source.getName());
        if (!newName.equals(source.getName())) {
            // Rename during move
            source.name = newName;
        }
        destDir.addChild(source);
        
        return true;
    }
    
    /**
     * Copies a file or directory to a new location.
     */
    public boolean cp(String sourcePath, String destPath) {
        Entry source = resolvePath(sourcePath);
        
        if (source == null) {
            System.out.println("cp: " + sourcePath + ": No such file or directory");
            return false;
        }
        
        // Determine destination
        Entry destEntry = resolvePath(destPath);
        Directory destDir;
        String newName;
        
        if (destEntry != null && destEntry.isDirectory()) {
            destDir = (Directory) destEntry;
            newName = source.getName();
        } else {
            int lastSlash = destPath.lastIndexOf('/');
            String destDirPath = lastSlash > 0 ? destPath.substring(0, lastSlash) : 
                                (destPath.startsWith("/") ? "/" : ".");
            newName = lastSlash >= 0 ? destPath.substring(lastSlash + 1) : destPath;
            
            Entry destDirEntry = resolvePath(destDirPath);
            if (destDirEntry == null || !destDirEntry.isDirectory()) {
                System.out.println("cp: " + destDirPath + ": No such directory");
                return false;
            }
            destDir = (Directory) destDirEntry;
        }
        
        // Check for name conflict
        if (destDir.hasChild(newName)) {
            System.out.println("cp: " + newName + ": already exists in destination");
            return false;
        }
        
        // Perform copy
        Entry copy;
        if (source.isFile()) {
            copy = ((File) source).copy(newName, destDir);
        } else {
            copy = ((Directory) source).copyDeep(newName, destDir);
        }
        destDir.addChild(copy);
        
        return true;
    }
    
    // ==================== Listing ====================
    
    /**
     * Lists contents of a directory.
     */
    public List<Entry> ls(String path) {
        Entry entry = resolvePath(path);
        
        if (entry == null) {
            System.out.println("ls: " + path + ": No such file or directory");
            return Collections.emptyList();
        }
        
        if (entry.isFile()) {
            return Collections.singletonList(entry);
        }
        
        return ((Directory) entry).getChildren();
    }
    
    /**
     * Lists contents of current directory.
     */
    public List<Entry> ls() {
        return ls(".");
    }
    
    // ==================== Search ====================
    
    /**
     * Finds entries matching the given name pattern.
     */
    public List<Entry> find(String namePattern) {
        return find(SearchCriteria.builder().nameContains(namePattern).build());
    }
    
    /**
     * Finds entries matching the given criteria.
     */
    public List<Entry> find(SearchCriteria criteria) {
        return find(root, criteria);
    }
    
    /**
     * Finds entries matching criteria starting from a specific directory.
     */
    public List<Entry> find(String startPath, SearchCriteria criteria) {
        Entry start = resolvePath(startPath);
        if (start == null || !start.isDirectory()) {
            return Collections.emptyList();
        }
        return find((Directory) start, criteria);
    }
    
    private List<Entry> find(Directory dir, SearchCriteria criteria) {
        List<Entry> results = new ArrayList<>();
        findRecursive(dir, criteria.toPredicate(), results);
        return results;
    }
    
    private void findRecursive(Directory dir, java.util.function.Predicate<Entry> predicate, 
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
    
    /**
     * Finds files by extension.
     */
    public List<File> findByExtension(String extension) {
        List<Entry> entries = find(SearchCriteria.builder()
                .extension(extension)
                .filesOnly()
                .build());
        
        List<File> files = new ArrayList<>();
        for (Entry entry : entries) {
            files.add((File) entry);
        }
        return files;
    }
    
    // ==================== Utility ====================
    
    /**
     * Gets the size of a file or directory.
     */
    public long getSize(String path) {
        Entry entry = resolvePath(path);
        return entry != null ? entry.getSize() : -1;
    }
    
    /**
     * Resolves a path to an entry.
     */
    public Entry resolvePath(String path) {
        if (path == null || path.isEmpty() || path.equals(".")) {
            return currentDirectory;
        }
        
        String[] parts = parsePath(path);
        Directory current = getStartDirectory(path);
        
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
                // File found - must be the last part
                return child;
            }
        }
        
        return current;
    }
    
    private String[] parsePath(String path) {
        return path.split("/");
    }
    
    private Directory getStartDirectory(String path) {
        return path.startsWith("/") ? root : currentDirectory;
    }
    
    /**
     * Gets the root directory.
     */
    public Directory getRoot() {
        return root;
    }
    
    /**
     * Gets the current directory.
     */
    public Directory getCurrentDirectory() {
        return currentDirectory;
    }
    
    /**
     * Prints the directory tree.
     */
    public void tree() {
        tree(root, "");
    }
    
    public void tree(String path) {
        Entry entry = resolvePath(path);
        if (entry != null && entry.isDirectory()) {
            tree((Directory) entry, "");
        }
    }
    
    private void tree(Directory dir, String prefix) {
        System.out.println(prefix + dir.getName() + "/");
        
        List<Entry> children = dir.getChildren();
        for (int i = 0; i < children.size(); i++) {
            Entry child = children.get(i);
            boolean isLast = (i == children.size() - 1);
            String connector = isLast ? "└── " : "├── ";
            String newPrefix = prefix + (isLast ? "    " : "│   ");
            
            if (child.isDirectory()) {
                tree((Directory) child, prefix + connector.substring(0, 4));
            } else {
                System.out.println(prefix + connector + child.getName());
            }
        }
    }
}
```

### 2.7 Demo Application

```java
// FileSystemDemo.java
package com.filesystem;

import java.util.List;

public class FileSystemDemo {
    
    public static void main(String[] args) {
        System.out.println("=== FILE SYSTEM DEMO ===\n");
        
        FileSystem fs = new FileSystem();
        
        // ==================== Create Directory Structure ====================
        System.out.println("===== CREATING DIRECTORY STRUCTURE =====\n");
        
        fs.mkdir("/home/user1/documents");
        fs.mkdir("/home/user1/pictures");
        fs.mkdir("/home/user2");
        fs.mkdir("/etc");
        fs.mkdir("/var/log");
        
        System.out.println("Directory tree after mkdir:");
        fs.tree();
        
        // ==================== Create Files ====================
        System.out.println("\n===== CREATING FILES =====\n");
        
        fs.createFile("/home/user1/documents/report.pdf", "PDF content here");
        fs.createFile("/home/user1/documents/notes.txt", "Some notes...");
        fs.createFile("/home/user1/pictures/photo.jpg", "JPEG binary data");
        fs.createFile("/home/user2/data.csv", "col1,col2,col3\n1,2,3");
        fs.createFile("/etc/config.ini", "[settings]\nkey=value");
        fs.createFile("/var/log/system.log", "2024-01-01 System started");
        
        System.out.println("Directory tree after creating files:");
        fs.tree();
        
        // ==================== Navigation ====================
        System.out.println("\n===== NAVIGATION =====\n");
        
        System.out.println("Current directory: " + fs.pwd());
        
        fs.cd("/home/user1");
        System.out.println("After cd /home/user1: " + fs.pwd());
        
        fs.cd("documents");
        System.out.println("After cd documents: " + fs.pwd());
        
        fs.cd("..");
        System.out.println("After cd ..: " + fs.pwd());
        
        fs.cd("/");
        System.out.println("After cd /: " + fs.pwd());
        
        // ==================== Listing ====================
        System.out.println("\n===== LISTING =====\n");
        
        System.out.println("Contents of /home/user1:");
        for (Entry entry : fs.ls("/home/user1")) {
            System.out.println("  " + entry);
        }
        
        System.out.println("\nContents of /home/user1/documents:");
        for (Entry entry : fs.ls("/home/user1/documents")) {
            System.out.println("  " + entry);
        }
        
        // ==================== Search ====================
        System.out.println("\n===== SEARCH =====\n");
        
        System.out.println("Find files containing 'log':");
        for (Entry entry : fs.find("log")) {
            System.out.println("  " + entry.getPath());
        }
        
        System.out.println("\nFind all .txt files:");
        for (File file : fs.findByExtension("txt")) {
            System.out.println("  " + file.getPath());
        }
        
        System.out.println("\nFind files larger than 10 bytes:");
        List<Entry> largeFiles = fs.find(SearchCriteria.builder()
                .filesOnly()
                .minSize(10)
                .build());
        for (Entry entry : largeFiles) {
            System.out.println("  " + entry.getPath() + " (" + entry.getSize() + " bytes)");
        }
        
        // ==================== Size Calculation ====================
        System.out.println("\n===== SIZE CALCULATION =====\n");
        
        System.out.println("Size of /home: " + fs.getSize("/home") + " bytes");
        System.out.println("Size of /home/user1: " + fs.getSize("/home/user1") + " bytes");
        System.out.println("Size of /home/user1/documents/report.pdf: " + 
                          fs.getSize("/home/user1/documents/report.pdf") + " bytes");
        
        // ==================== Copy ====================
        System.out.println("\n===== COPY =====\n");
        
        fs.cp("/home/user1/documents/notes.txt", "/home/user2/notes_backup.txt");
        System.out.println("Copied notes.txt to user2/notes_backup.txt");
        
        System.out.println("Contents of /home/user2:");
        for (Entry entry : fs.ls("/home/user2")) {
            System.out.println("  " + entry);
        }
        
        // ==================== Move ====================
        System.out.println("\n===== MOVE =====\n");
        
        fs.mv("/home/user2/data.csv", "/home/user1/documents/data.csv");
        System.out.println("Moved data.csv to user1/documents");
        
        System.out.println("Contents of /home/user1/documents:");
        for (Entry entry : fs.ls("/home/user1/documents")) {
            System.out.println("  " + entry);
        }
        
        // ==================== Delete ====================
        System.out.println("\n===== DELETE =====\n");
        
        fs.rm("/home/user1/documents/notes.txt");
        System.out.println("Deleted notes.txt");
        
        System.out.println("Contents of /home/user1/documents:");
        for (Entry entry : fs.ls("/home/user1/documents")) {
            System.out.println("  " + entry);
        }
        
        // ==================== Final Tree ====================
        System.out.println("\n===== FINAL DIRECTORY TREE =====\n");
        fs.tree();
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
}
```

---

## STEP 5: Simulation / Dry Run

### Scenario: Creating and Navigating

```
Initial State:
/ (root)
currentDirectory = /

Step 1: mkdir("/home/user1/documents")
- Parse path: ["", "home", "user1", "documents"]
- Start at root (path starts with /)
- Create "home" directory under root
- Create "user1" directory under home
- Create "documents" directory under user1

Tree after:
/
└── home/
    └── user1/
        └── documents/

Step 2: touch("/home/user1/documents/report.pdf")
- Parse: dirPath="/home/user1/documents", fileName="report.pdf"
- Parent directory exists
- Create File("report.pdf", parent=documents)
- Add to documents.children

Step 3: cd("/home/user1")
- Resolve path: / → home → user1
- Set currentDirectory = user1

Step 4: ls()
- List currentDirectory.children
- Returns: [documents/]

Step 5: cd("documents")  (relative path)
- Start from currentDirectory (user1)
- Resolve: user1 → documents
- Set currentDirectory = documents

Step 6: pwd()
- Traverse parent chain: documents → user1 → home → root
- Build path: /home/user1/documents
```

---

## File Structure

```
com/filesystem/
├── Permissions.java
├── Entry.java
├── File.java
├── Directory.java
├── SearchCriteria.java
├── FileSystem.java
└── FileSystemDemo.java
```

