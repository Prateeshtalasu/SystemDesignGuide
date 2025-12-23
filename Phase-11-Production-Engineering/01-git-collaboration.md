# 🔀 Git & Collaboration

## 0️⃣ Prerequisites

Before diving into Git and collaboration strategies, you should understand:

- **Command Line Basics**: Navigating directories, running commands. If you can `cd`, `ls`, and run programs from a terminal, you're ready.
- **Basic File System Concepts**: Files, directories, and how changes are saved to disk.
- **What "Version" Means**: The idea that a document can have different states over time (like saving multiple drafts of an essay).

No prior Git knowledge is assumed. We start from zero.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Before Version Control

Imagine you're writing code for a project. Without version control:

**Problem 1: The "Final_v2_REAL_FINAL.java" Nightmare**
```
MyProject/
├── Calculator.java
├── Calculator_backup.java
├── Calculator_old.java
├── Calculator_v2.java
├── Calculator_v2_fixed.java
├── Calculator_FINAL.java
├── Calculator_FINAL_v2.java
└── Calculator_FINAL_REAL.java   ← Which one is actually current?
```

You've lost track of which version works. You can't remember what changed between versions. You accidentally delete the wrong file and lose a day's work.

**Problem 2: Team Collaboration Chaos**

Two developers, Alice and Bob, are working on the same project:

1. Alice downloads the code on Monday
2. Bob downloads the same code on Monday
3. Alice changes `login()` function on Tuesday
4. Bob changes `login()` function on Wednesday (he doesn't have Alice's changes)
5. Bob uploads his version on Thursday
6. Alice's work is completely overwritten and lost

This is called the **"last writer wins"** problem, and it destroyed countless hours of work before version control existed.

**Problem 3: "It Worked Yesterday"**

Your code was working perfectly on Friday. On Monday, it's broken. What changed? Without version control, you have no way to:
- See what the code looked like on Friday
- Compare Friday's code with today's code
- Go back to Friday's working version

**Problem 4: Blame and Understanding**

A critical bug appears in production. The code looks intentional, but:
- Who wrote this line?
- When was it written?
- Why was it written this way?
- What was the context?

Without version control, these questions are unanswerable.

### What Breaks Without Version Control

| Scenario | Without Version Control | With Version Control |
|----------|------------------------|---------------------|
| Accidental deletion | Work is permanently lost | Restore from any previous version |
| Team conflicts | Last person to save wins | Merge changes intelligently |
| Finding bugs | "It broke sometime" | Pinpoint exact commit that broke it |
| Understanding code | Ask around and hope someone remembers | See full history with context |
| Experimentation | Too risky, might break everything | Create branches, experiment freely |
| Releases | Manual tracking in spreadsheets | Tagged versions with full history |

---

## 2️⃣ Intuition and Mental Model

### The Time Machine Analogy

Think of Git as a **time machine for your code**.

Imagine you're writing a novel:
- Every time you finish a chapter, you take a **snapshot** (photograph) of the entire manuscript
- Each snapshot has a **timestamp** and a **note** explaining what you wrote
- You store these snapshots in a **photo album** (repository)
- You can flip back to any snapshot to see exactly what the manuscript looked like at that moment
- If you want to try a risky new ending, you create a **parallel photo album** (branch) to experiment without risking your main story

**Key insight**: Git doesn't store "changes." It stores **complete snapshots** of your project at each point in time. (Internally, it's optimized to not duplicate unchanged files, but conceptually, each commit is a full snapshot.)

### The Three Areas Mental Model

Git has three "areas" where your files can exist:

```
┌─────────────────┐    git add     ┌─────────────────┐    git commit    ┌─────────────────┐
│  Working        │ ────────────▶  │  Staging Area   │ ───────────────▶ │   Repository    │
│  Directory      │                │  (Index)        │                  │   (History)     │
│                 │                │                 │                  │                 │
│  Your actual    │                │  "Ready for     │                  │  Permanent      │
│  files on disk  │                │  photo" area    │                  │  snapshots      │
└─────────────────┘                └─────────────────┘                  └─────────────────┘
        │                                                                       │
        │◀──────────────────── git checkout ────────────────────────────────────│
```

**Analogy**: Think of taking a group photo:
1. **Working Directory**: Everyone at the party (all your files, changed or not)
2. **Staging Area**: People you've asked to line up for the photo (files you've selected to include in the next snapshot)
3. **Repository**: The photo album (permanent record of all snapshots taken)

You don't have to include everyone in every photo. You choose who lines up (staging), then take the photo (commit).

---

## 3️⃣ How It Works Internally

### Git's Data Model

Git stores everything as **objects** in a simple key-value database. There are four types of objects:

#### 1. Blob (Binary Large Object)
- Stores the **contents** of a single file
- Identified by SHA-1 hash of the contents
- Same content = same hash = stored only once

```
"Hello, World!" → SHA-1 → af5626b4a114abcb82d63db7c8082c3c4756e51b
```

#### 2. Tree
- Represents a **directory**
- Contains pointers to blobs (files) and other trees (subdirectories)
- Each entry has: mode, type, hash, name

```
tree 8a3f2...
├── 100644 blob a1b2c3... README.md
├── 100644 blob d4e5f6... Main.java
└── 040000 tree 7g8h9i... src/
```

#### 3. Commit
- A **snapshot** in time
- Contains:
  - Pointer to a tree (the root directory at this moment)
  - Pointer to parent commit(s)
  - Author name, email, timestamp
  - Committer name, email, timestamp
  - Commit message

```
commit 9f8e7d...
├── tree: 8a3f2...
├── parent: 5c4b3a...
├── author: Alice <alice@example.com> 1703347200
├── committer: Alice <alice@example.com> 1703347200
└── message: "Add user authentication"
```

#### 4. Tag
- A named pointer to a specific commit
- Used for marking releases (v1.0.0, v2.1.3)

### How Commits Form a Chain

Each commit points to its parent, forming a linked list:

```
[Commit A] ◀─── [Commit B] ◀─── [Commit C] ◀─── [Commit D]
 (initial)      (add login)     (fix bug)       (add logout)
                                                     ▲
                                                     │
                                                   HEAD
                                                   main
```

- **HEAD**: A pointer to your current position in history
- **Branch (main)**: A pointer to a specific commit (moves forward as you add commits)

### How Branching Works

A branch is just a **pointer to a commit**. Creating a branch is instant because Git only creates a 41-byte file containing the commit hash.

```
                        feature-login
                              │
                              ▼
[A] ◀─── [B] ◀─── [C] ◀─── [D]
                    │
                    ▼
                  main
                  HEAD
```

When you commit on `feature-login`:

```
                        feature-login
                              │
                              ▼
[A] ◀─── [B] ◀─── [C] ◀─── [D] ◀─── [E]
                    │
                    ▼
                  main
```

### How Merge Works

**Fast-Forward Merge** (when main hasn't moved):

Before:
```
                  feature
                     │
                     ▼
[A] ◀─── [B] ◀─── [C]
          │
          ▼
        main
```

After `git merge feature`:
```
                  feature
                     │
                     ▼
[A] ◀─── [B] ◀─── [C]
                   │
                   ▼
                 main
```

Git just moves the `main` pointer forward. No new commit created.

**Three-Way Merge** (when both branches have new commits):

Before:
```
          feature
             │
             ▼
          [D] ◀─── [E]
         /
[A] ◀─── [B] ◀─── [C]
                   │
                   ▼
                 main
```

After `git merge feature`:
```
          feature
             │
             ▼
          [D] ◀─── [E]
         /           \
[A] ◀─── [B] ◀─── [C] ◀─── [M]  (merge commit)
                            │
                            ▼
                          main
```

Git creates a **merge commit** `[M]` with two parents.

### How Rebase Works

Rebase **replays** your commits on top of another branch:

Before:
```
          feature
             │
             ▼
          [D] ◀─── [E]
         /
[A] ◀─── [B] ◀─── [C]
                   │
                   ▼
                 main
```

After `git rebase main` (while on feature):
```
[A] ◀─── [B] ◀─── [C] ◀─── [D'] ◀─── [E']
                   │                   │
                   ▼                   ▼
                 main              feature
```

Note: `[D']` and `[E']` are **new commits** with different hashes. The original `[D]` and `[E]` still exist but are now orphaned.

---

## 4️⃣ Simulation: Step-by-Step Git Workflow

Let's trace through a complete workflow with one developer.

### Initial Setup

```bash
# Create a new project directory
mkdir my-project
cd my-project

# Initialize Git repository
git init
```

What happens internally:
```
my-project/
└── .git/                    ← Git's database
    ├── HEAD                 ← Points to current branch (refs/heads/main)
    ├── config               ← Repository configuration
    ├── objects/             ← Where blobs, trees, commits are stored
    │   ├── info/
    │   └── pack/
    └── refs/
        ├── heads/           ← Branch pointers (empty initially)
        └── tags/            ← Tag pointers
```

### First Commit

```bash
# Create a file
echo "# My Project" > README.md

# Check status
git status
```

Output:
```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        README.md

nothing added to commit but untracked files present
```

The file is in the **Working Directory** but not staged.

```bash
# Stage the file
git add README.md

# Check status again
git status
```

Output:
```
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   README.md
```

Now the file is in the **Staging Area**.

```bash
# Commit
git commit -m "Initial commit: Add README"
```

Output:
```
[main (root-commit) a1b2c3d] Initial commit: Add README
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

What happened internally:
1. Git created a **blob** object for README.md's contents
2. Git created a **tree** object pointing to that blob
3. Git created a **commit** object pointing to that tree
4. Git updated `refs/heads/main` to point to this commit
5. Git updated `HEAD` to point to `refs/heads/main`

### Making Changes

```bash
# Modify the file
echo "This is my awesome project." >> README.md

# Create another file
echo "public class Main {}" > Main.java

# Check status
git status
```

Output:
```
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
        modified:   README.md

Untracked files:
        Main.java
```

```bash
# Stage both files
git add README.md Main.java

# Commit
git commit -m "Add project description and Main class"
```

Now we have two commits:
```
[a1b2c3d] ◀─── [e4f5g6h]
 Initial         Add project...
                     │
                     ▼
                   main
                   HEAD
```

### Viewing History

```bash
git log --oneline
```

Output:
```
e4f5g6h (HEAD -> main) Add project description and Main class
a1b2c3d Initial commit: Add README
```

```bash
# See what changed in the last commit
git show
```

Output:
```
commit e4f5g6h
Author: You <you@example.com>
Date:   Mon Dec 23 2024 10:00:00

    Add project description and Main class

diff --git a/Main.java b/Main.java
new file mode 100644
index 0000000..1234567
--- /dev/null
+++ b/Main.java
@@ -0,0 +1 @@
+public class Main {}

diff --git a/README.md b/README.md
index abcdef1..2345678
--- a/README.md
+++ b/README.md
@@ -1 +1,2 @@
 # My Project
+This is my awesome project.
```

---

## 5️⃣ Branching Strategies in Production

### Strategy 1: Gitflow

**Used by**: Teams with scheduled releases (banks, enterprises, mobile apps with app store review cycles)

```
                    hotfix/critical-bug
                           │
    ┌──────────────────────┼──────────────────────────────────────┐
    │                      ▼                                      │
────●──────────────────────●──────────────────────────────────────●──── main (production)
    │                      │                                      │
    │   ┌──────────────────┴──────────────────────────────────────┤
    │   │                                                         │
────●───●─────────●────────●──────────●───────────●───────────────●──── develop
        │         │        │          │           │
        │         │        │          │           │
        │         │        └──────────┴───────────┘
        │         │              release/1.0
        │         │
        └─────────┴─────────────────────────────
              feature/login    feature/payment
```

**Branch Types**:

| Branch | Purpose | Created From | Merges Into |
|--------|---------|--------------|-------------|
| `main` | Production code | - | - |
| `develop` | Integration branch | main | release |
| `feature/*` | New features | develop | develop |
| `release/*` | Release preparation | develop | main + develop |
| `hotfix/*` | Emergency fixes | main | main + develop |

**Workflow**:

1. Developer creates `feature/user-auth` from `develop`
2. Developer works on feature, makes commits
3. Developer opens PR to merge into `develop`
4. Code review, tests pass, merge
5. When ready for release, create `release/1.0` from `develop`
6. QA tests release branch, bug fixes go directly to release branch
7. Merge `release/1.0` into `main` (tag as v1.0.0)
8. Merge `release/1.0` back into `develop`

**Pros**:
- Clear separation between stable and development code
- Supports parallel development of features
- Explicit release process

**Cons**:
- Complex, many branches to manage
- Merge conflicts between long-lived branches
- Slow for continuous deployment

### Strategy 2: Trunk-Based Development

**Used by**: Google, Facebook, Netflix, high-velocity teams with strong CI/CD

```
──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──── main (trunk)
  │     │        │              │        │
  └──●──┘        └──●──●──●─────┘        └──●──┘
    │                 │                      │
 short-lived      short-lived           short-lived
  feature          feature               feature
 (< 1 day)        (< 2 days)            (< 1 day)
```

**Rules**:
1. Everyone commits to `main` (trunk) frequently (at least daily)
2. Feature branches live for hours, not days
3. Features are hidden behind **feature flags** until ready
4. Broken main = stop everything and fix it

**Workflow**:

```bash
# Start work (from main)
git checkout main
git pull
git checkout -b feature/add-button

# Work for a few hours
# ... make changes ...
git commit -m "Add submit button"

# Before pushing, rebase on latest main
git fetch origin
git rebase origin/main

# Push and create PR
git push -u origin feature/add-button

# PR reviewed and merged within hours
# Delete feature branch
```

**Pros**:
- Simple, few branches
- Fast integration, fewer merge conflicts
- Continuous deployment friendly
- Forces small, incremental changes

**Cons**:
- Requires excellent CI/CD
- Requires feature flags for incomplete features
- Requires disciplined, experienced team
- Broken main affects everyone

### Strategy 3: GitHub Flow

**Used by**: GitHub, startups, teams wanting simplicity with PRs

```
──●──────────●──────────────────●────────────────●──────────── main
  │          │                  │                │
  └──●──●────┘                  └──●──●──●───────┘
      │                              │
   feature-1                     feature-2
```

**Rules**:
1. `main` is always deployable
2. Create descriptively named branches from `main`
3. Push to your branch regularly
4. Open PR when ready for feedback
5. Merge only after PR approval
6. Deploy immediately after merging

**Workflow**:

```bash
# Create branch with descriptive name
git checkout -b add-user-authentication

# Work on feature
git add .
git commit -m "Implement login form"
git push -u origin add-user-authentication

# Open PR on GitHub
# Discuss, review, iterate
# Merge when approved
# Deploy
```

**Pros**:
- Simple to understand
- Works well with GitHub's UI
- PR-centric workflow encourages code review

**Cons**:
- No explicit release process
- Assumes main is always deployable (requires good CI)

### Comparison Table

| Aspect | Gitflow | Trunk-Based | GitHub Flow |
|--------|---------|-------------|-------------|
| Complexity | High | Low | Medium |
| Branch lifespan | Days to weeks | Hours | Days |
| Release process | Explicit | Continuous | Continuous |
| Best for | Scheduled releases | High-velocity teams | General purpose |
| CI/CD requirement | Moderate | Critical | Important |
| Feature flags | Optional | Required | Optional |

---

## 6️⃣ Merge vs Rebase: When to Use Each

### The Fundamental Difference

**Merge**: Creates a new commit that combines two branches. Preserves history exactly as it happened.

**Rebase**: Rewrites history by replaying your commits on top of another branch. Creates a linear history.

### Visual Comparison

**Starting point**:
```
          feature
             │
             ▼
          [C] ◀─── [D]
         /
[A] ◀─── [B] ◀─── [E]
                   │
                   ▼
                 main
```

**After merge** (git checkout main && git merge feature):
```
          feature
             │
             ▼
          [C] ◀─── [D]
         /           \
[A] ◀─── [B] ◀─── [E] ◀─── [M]
                            │
                            ▼
                          main
```

**After rebase** (git checkout feature && git rebase main):
```
[A] ◀─── [B] ◀─── [E] ◀─── [C'] ◀─── [D']
                   │                   │
                   ▼                   ▼
                 main              feature
```

### When to Use Merge

Use merge when:

1. **Preserving history matters**: You want to see exactly when branches diverged and merged
2. **Shared branches**: The branch has been pushed and others are working on it
3. **Complex integrations**: Merging a long-lived feature branch where context matters
4. **Compliance requirements**: Audit trails require unmodified history

```bash
# Safe merge workflow
git checkout main
git pull origin main
git merge feature/payment --no-ff  # --no-ff forces a merge commit even if fast-forward possible
git push origin main
```

### When to Use Rebase

Use rebase when:

1. **Clean history**: You want a linear, easy-to-read history
2. **Before merging**: Cleaning up your feature branch before creating a PR
3. **Syncing with main**: Keeping your feature branch up-to-date with main
4. **Local commits only**: The commits haven't been pushed yet

```bash
# Safe rebase workflow
git checkout feature/login
git fetch origin
git rebase origin/main

# If conflicts occur:
# 1. Fix conflicts in files
# 2. git add <fixed-files>
# 3. git rebase --continue

git push --force-with-lease  # Safe force push (fails if remote has new commits)
```

### The Golden Rule of Rebasing

> **Never rebase commits that have been pushed to a shared branch that others are working on.**

Why? Rebase creates **new commits** with new hashes. If someone else has the old commits, their history diverges from yours, causing massive confusion.

**Safe**: Rebasing your local commits before pushing
**Safe**: Rebasing your feature branch that only you work on
**DANGEROUS**: Rebasing `main` or `develop` that others have pulled

### Interactive Rebase for Cleaning Up History

Before creating a PR, clean up your commits:

```bash
git rebase -i HEAD~5  # Interactively rebase last 5 commits
```

This opens an editor:
```
pick a1b2c3d Add login form
pick e4f5g6h Fix typo
pick h7i8j9k Add validation
pick l0m1n2o Fix another typo
pick p3q4r5s Add tests

# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# d, drop = remove commit
```

Clean it up:
```
pick a1b2c3d Add login form
fixup e4f5g6h Fix typo
pick h7i8j9k Add validation
fixup l0m1n2o Fix another typo
pick p3q4r5s Add tests
```

Result: 3 clean commits instead of 5 messy ones.

---

## 7️⃣ Conflict Resolution

### Why Conflicts Happen

A conflict occurs when Git can't automatically determine which changes to keep:

```
# Original (commit B):
function greet() {
    return "Hello";
}

# Alice's change (commit C):
function greet() {
    return "Hello, World!";
}

# Bob's change (commit D):
function greet() {
    return "Hi there!";
}
```

Both Alice and Bob changed the same line. Git doesn't know which to keep.

### Anatomy of a Conflict

When you encounter a conflict, Git marks the file:

```java
public class Greeting {
    public String greet() {
<<<<<<< HEAD
        return "Hello, World!";
=======
        return "Hi there!";
>>>>>>> feature/bobs-change
    }
}
```

- `<<<<<<< HEAD`: Start of your current branch's version
- `=======`: Separator
- `>>>>>>> feature/bobs-change`: End of incoming branch's version

### Step-by-Step Resolution

```bash
# 1. Attempt the merge
git merge feature/bobs-change

# Output:
# Auto-merging Greeting.java
# CONFLICT (content): Merge conflict in Greeting.java
# Automatic merge failed; fix conflicts and then commit the result.

# 2. See which files have conflicts
git status

# 3. Open the file and resolve
# Choose one version, combine them, or write something new:
public class Greeting {
    public String greet() {
        return "Hello there, World!";  // Combined both ideas
    }
}

# 4. Mark as resolved
git add Greeting.java

# 5. Complete the merge
git commit  # Git provides a default merge message
```

### Conflict Prevention Strategies

1. **Pull frequently**: Stay up-to-date with main
2. **Small PRs**: Less code = fewer conflicts
3. **Communicate**: Let teammates know which files you're modifying
4. **Use feature flags**: Merge incomplete features without conflicts
5. **Code ownership**: Reduce overlap in who modifies which files

### Tools for Conflict Resolution

```bash
# Use a visual merge tool
git mergetool

# Configure your preferred tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

---

## 8️⃣ Commit Quality and Best Practices

### Atomic Commits

An **atomic commit** is a single, self-contained change that:
- Does one thing
- Can be understood in isolation
- Can be reverted without side effects
- Leaves the codebase in a working state

**Bad**: One commit that adds a feature, fixes a bug, and refactors code
**Good**: Three separate commits, one for each change

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Example**:
```
feat(auth): Add password reset functionality

Implement password reset flow with email verification.
Users can now request a password reset link that expires
after 24 hours.

Closes #123
```

**Types**:
| Type | Description |
|------|-------------|
| feat | New feature |
| fix | Bug fix |
| docs | Documentation only |
| style | Formatting, no code change |
| refactor | Code change that neither fixes a bug nor adds a feature |
| test | Adding or fixing tests |
| chore | Build process, dependencies, etc. |

### The 50/72 Rule

- **Subject line**: Max 50 characters
- **Body lines**: Max 72 characters
- **Blank line** between subject and body

Why? Git log, GitHub, and many tools truncate at these lengths.

### What Makes a Good Commit Message

**Bad**:
```
fix bug
```

**Better**:
```
Fix null pointer in user login
```

**Best**:
```
fix(auth): Prevent NPE when user has no email

Users created before 2020 may not have an email address.
The login flow assumed email was always present, causing
a NullPointerException.

Added null check and fallback to username-based auth.

Fixes #456
```

---

## 9️⃣ Pull Request Best Practices

### PR Size

| Size | Lines Changed | Review Time | Defect Rate |
|------|---------------|-------------|-------------|
| Small | < 200 | 15 minutes | Low |
| Medium | 200-400 | 30 minutes | Medium |
| Large | 400+ | 1+ hours | High |

**Research shows**: PRs over 400 lines have significantly higher defect rates because reviewers get fatigued.

### PR Structure

```markdown
## Description
Brief explanation of what this PR does

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## How Has This Been Tested?
- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual testing

## Checklist
- [ ] My code follows the style guidelines
- [ ] I have performed a self-review
- [ ] I have commented hard-to-understand areas
- [ ] I have made corresponding changes to docs
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix/feature works
- [ ] New and existing unit tests pass locally

## Screenshots (if applicable)

## Related Issues
Closes #123
```

### Code Review Guidelines

**As a Reviewer**:

1. **Be kind**: Critique code, not people
2. **Be specific**: "This could cause a race condition because..." not "This is wrong"
3. **Ask questions**: "What happens if X is null?" not "Handle null case"
4. **Praise good work**: "Nice use of the builder pattern here!"
5. **Focus on important issues**: Don't nitpick style if there are logic bugs

**As an Author**:

1. **Self-review first**: Read your own diff before requesting review
2. **Provide context**: Explain why, not just what
3. **Respond to all comments**: Even if just "Done" or "Good point, fixed"
4. **Don't take it personally**: Reviews improve code, not judge you

### Review Comment Prefixes

| Prefix | Meaning |
|--------|---------|
| `nit:` | Minor style issue, optional to fix |
| `question:` | I don't understand, please explain |
| `suggestion:` | Consider this alternative |
| `issue:` | This needs to be fixed |
| `blocker:` | Cannot approve until fixed |

---

## 🔟 Monorepo vs Polyrepo

### Polyrepo (Multiple Repositories)

Each service/component has its own repository:

```
github.com/company/
├── user-service/
├── payment-service/
├── notification-service/
├── frontend/
└── shared-library/
```

**Pros**:
- Clear ownership boundaries
- Independent versioning
- Smaller clone size
- Fine-grained access control
- Independent CI/CD pipelines

**Cons**:
- Cross-repo changes are painful
- Dependency management is complex
- Code duplication across repos
- Harder to refactor across services

**Used by**: Netflix (historically), many microservices teams

### Monorepo (Single Repository)

All code in one repository:

```
github.com/company/monorepo/
├── services/
│   ├── user/
│   ├── payment/
│   └── notification/
├── frontend/
├── libs/
│   └── shared/
└── tools/
```

**Pros**:
- Atomic cross-service changes
- Easier code sharing
- Unified tooling and CI
- Single source of truth
- Easier refactoring

**Cons**:
- Large clone size
- Complex build systems needed
- All-or-nothing access control
- CI must be smart about what to build

**Used by**: Google, Facebook, Microsoft, Uber

### Tooling for Monorepos

| Tool | Purpose |
|------|---------|
| Bazel | Build system (Google) |
| Buck | Build system (Facebook) |
| Nx | JavaScript monorepo tool |
| Turborepo | JavaScript/TypeScript monorepo |
| Lerna | JavaScript monorepo (legacy) |

### Decision Framework

Choose **Polyrepo** when:
- Teams are truly independent
- Different languages/stacks per service
- Strict access control needed
- Services have different release cycles

Choose **Monorepo** when:
- Frequent cross-service changes
- Shared libraries are common
- Want unified tooling
- Team is small to medium
- Strong CI/CD infrastructure exists

---

## 1️⃣1️⃣ Interview Follow-Up Questions

### Q1: "What's the difference between git merge and git rebase?"

**Answer**:
Merge creates a new commit that combines two branches, preserving the exact history of when branches diverged and merged. Rebase replays your commits on top of another branch, creating a linear history but rewriting commit hashes.

Use merge for shared branches or when history preservation matters. Use rebase for cleaning up local commits before creating a PR, or for keeping a feature branch up-to-date with main.

The golden rule: never rebase commits that have been pushed to a branch others are working on, because it rewrites history and causes divergence.

### Q2: "How do you handle merge conflicts?"

**Answer**:
1. First, understand what caused the conflict by looking at the markers Git inserts
2. Open the file and examine both versions
3. Decide: keep one version, combine both, or write something new
4. Remove the conflict markers
5. Test that the code works
6. Stage the file with `git add`
7. Complete the merge with `git commit`

Prevention is better: pull frequently, make small PRs, communicate with teammates about which files you're modifying.

### Q3: "What branching strategy would you recommend for a team of 10 engineers?"

**Answer**:
For a team of 10, I'd recommend GitHub Flow or a simplified trunk-based development:

- Main branch is always deployable
- Short-lived feature branches (1-3 days max)
- Mandatory PR reviews before merging
- CI runs on every PR
- Deploy after merge

If the team has scheduled releases (mobile apps, enterprise software), Gitflow might be appropriate, but for most web services, simpler is better.

The key factors are: release frequency, team discipline, CI/CD maturity, and whether features can be hidden behind feature flags.

### Q4: "What makes a good commit message?"

**Answer**:
A good commit message:
1. Has a concise subject line (50 chars max) starting with a verb
2. Explains WHY the change was made, not just what
3. References related issues or tickets
4. Uses conventional commit format (feat/fix/docs/etc.) for consistency

Example:
```
fix(auth): Prevent session timeout during checkout

Users were losing their cart when sessions expired mid-checkout.
Extended session TTL during active checkout flow.

Fixes #789
```

### Q5: "How would you handle a situation where you accidentally committed sensitive data?"

**Answer**:
1. **Don't panic**, but act quickly
2. **If not pushed yet**: `git reset HEAD~1` to undo the commit, remove the sensitive data, recommit
3. **If pushed but not merged**: Force push after cleaning (coordinate with team)
4. **If merged to main**: Use `git filter-branch` or BFG Repo-Cleaner to remove from entire history, then force push all branches
5. **Rotate the credentials immediately**, regardless of whether you cleaned the history
6. **Prevent future incidents**: Add patterns to `.gitignore`, use pre-commit hooks, use secret scanning tools

The key insight: even if you clean Git history, the credentials should be considered compromised because they may exist in clones, forks, or backups.

---

## 1️⃣2️⃣ One Clean Mental Summary

Git is a distributed version control system that takes snapshots of your entire project at each commit, forming a chain of history that you can navigate, branch from, and merge. Branching is cheap (just a pointer), enabling parallel development. The three areas (working directory, staging area, repository) give you control over what goes into each snapshot.

For teams: choose a branching strategy that matches your release cadence. Trunk-based for continuous deployment, Gitflow for scheduled releases, GitHub Flow for a balance. Write atomic commits with meaningful messages. Keep PRs small. Review code kindly but thoroughly. And remember: never rebase shared history.

