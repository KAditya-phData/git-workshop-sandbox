# **The Git Fear-Free Workshop: A Comprehensive Facilitator’s Guide and Technical Curriculum**

## **1. Introduction**

Git is the industry standard for version control, yet for many developers, commands like `merge`, `rebase`, and `reset` can induce significant hesitation. The fear of "breaking the build," losing work, or entering a "Detached HEAD" state often stifles experimentation and confidence.

This workshop provides a comprehensive, expert-level curriculum designed to replace that hesitation with mastery. By utilizing a sandboxed environment where failure is encouraged, we enable developers to practice high-stakes scenarios safely.

### **1.1 Workshop Methodology**

We utilize a scripted, automated setup to generate a complex, conflicted Git repository. This "sandbox" approach allows participants to encounter and resolve real-world crises without risk to production code.

The workshop is detailed in eight core modules:

1. **Safe Merging:** verifying changes before committing.
2. **Recovery:** Demystifying conflicts and utilizing the `reflog`, `ORIG_HEAD`, and reset strategies.
3. **History Navigation:** Understanding "Detached HEAD" as an exploratory tool.
4. **Reverting:** Non-destructive undoing, specifically regarding merge commits.
5. **Surgical Extraction:** extracting specific files without full branch merges, including deep file history, patches, and cherry-pick porting.
6. **Branch Renaming:** safely renaming branches locally and on the remote.
7. **Advanced Rebasing:** understanding `git rebase`, interactive mode, and the Golden Rule.
8. **Bulk Cherry-Picking:** extracting commit ranges and squashing via cherry-pick.

The following sections detail the technical infrastructure and theoretical underpinnings of each module.

## ---

**2\. Infrastructure: The "Holodeck" Simulation Script**

The foundation of this workshop is the automated setup script. This Bash script acts as the "Director," creating a consistent, reproducible state of chaos for every participant. It constructs a repository with a linear history, creates divergent branches, introduces conflicting changes to the same file on different branches, and simulates a "bad merge" that will require complex reverting later.

### **2.1 The Topology of Chaos**

The script creates a repository named `git-workshop-sandbox`. Inside, it constructs a Directed Acyclic Graph (DAG) designed to facilitate specific learning outcomes:

* **The Main Trunk:** Represents the "production" stability we are afraid to break.  
* **The Feature Branch (`feature/update-header`):** Contains a conflicting change to `header.html`, designed to trigger a merge conflict.
* **The "Bad" Feature (`feature/experimental-analytics`):** Contains a buggy file that is merged into `main` using `--no-ff`, creating a merge commit that must be reverted later.
* **The Utility Branch (`feature/cool-utils`):** Contains a useful file that we will want to "steal" without merging the branch, teaching the difference between `checkout` and `restore`.
* **The Renamed File:** A file created as `old_utils.js` and later renamed to `renamed_utils.js`, designed for the `--follow` file history exercise.
* **The Hotfix Branch (`hotfix/v1.0-security`):** Branched from the `v1.0` tag with a small security patch, providing a pre-built target for the Branch Renaming module.
* **The Multi-Commit Feature (`feature/my-work`):** A branch with four sequential commits (form component, layout, styling, validation), providing targets for the Advanced Rebasing and Bulk Cherry-Picking modules.

### **2.2 The Setup Script (setup_git_workshop.sh)**

Facilitators should distribute this script to all participants at the start of the session. It is idempotent (it cleans up previous runs) and self-contained.

**Prerequisites:** Participants must have Git installed and configured in their VS Code integrated terminal. The script assumes your global `user.name` and `user.email` are already set (i.e., you have committed at least once before). To run this script on Windows, use the **Git Bash** terminal in VS Code (select it from the terminal dropdown: `Terminal > New Terminal > Git Bash`).

```bash
#!/bin/bash

# ==============================================================================
# Git Fear-Free Workshop: The Holodeck Generator
# ==============================================================================
# This script automates the creation of a complex git history.
# It simulates the work of a team, creating conflicts, bad merges, and
# history that allows for time travel exercises.
#
# PREREQUISITES:
#   - Git is installed and globally configured (user.name and user.email set)
#   - Run from the VS Code integrated terminal (Git Bash on Windows)
# ==============================================================================

PROJECT_DIR="git-workshop-sandbox"

# --- 1. CLEANUP PHASE ---
# Remove any existing sandbox to ensure a fresh start for every run.
if [ -d "$PROJECT_DIR" ]; then
    echo "⚠️  Detected previous sandbox. Cleaning up..."
    rm -rf "$PROJECT_DIR"
fi

# --- 2. INITIALIZATION PHASE ---
echo "🚀 Initializing the Simulation Environment..."
mkdir "$PROJECT_DIR"
cd "$PROJECT_DIR" || exit

git init
# Ensure default branch is named 'main' for consistency
git branch -m main

# --- 3. SCENARIO: The Foundation (Modules 1-4) ---
echo "📝 Building commit history..."

# Initial Commit
echo "# Project Phoenix" > README.md
echo "This is the documentation for the critical infrastructure project." >> README.md
git add README.md
git commit -m "Initial commit: Project Phoenix setup"

# Add a config file (Base state)
echo "db_host=localhost" > config.env
echo "db_port=5432" >> config.env
git add config.env
git commit -m "Add configuration file"

# --- 4. SCENARIO: Time Travel Targets (Module 3) ---
# We create a series of commits to serve as "save points" for the Time Travel module.
# We sleep 1 second to ensure timestamps are distinct.
for i in {1..3}; do
    echo "Log entry $i: System stable" >> server.log
    git add server.log
    git commit -m "Update server logs: Entry $i"
    sleep 1
done

# Create a tag to act as a clear signpost in history
git tag -a v1.0 -m "Version 1.0 Release"

# --- 5. SCENARIO: File Rename History (Module 5: --follow exercise) ---
# Create a file under one name, then rename it. This sets up the exercise
# on tracking file history across renames using 'git log --follow'.
echo "function oldHelper() { return 'original'; }" > old_utils.js
git add old_utils.js
git commit -m "Add original utility file as old_utils.js"

git mv old_utils.js renamed_utils.js
echo "function renamedHelper() { return 'renamed and improved'; }" > renamed_utils.js
git add renamed_utils.js
git commit -m "Rename old_utils.js to renamed_utils.js and update"

# --- 6. SCENARIO: The Bad Merge & Revert Trap (Module 4) ---
# We create a feature branch that introduces a 'bug'.
# This sets up the exercise on reverting merge commits.
git checkout -b feature/experimental-analytics
echo "calculating_analytics = true" > analytics.js
echo "WARN: This function has a known memory leak" >> analytics.js
git add analytics.js
git commit -m "Add experimental analytics module"

# Switch back to main and create a divergence so the merge isn't a fast-forward
git checkout main
echo "security_level=high" >> config.env
git add config.env
git commit -m "Harden security settings"

# Merge the buggy feature into main.
# We use --no-ff (No Fast Forward) to force a merge commit object.
# This is crucial for the 'Revert -m 1' exercise.
git merge --no-ff feature/experimental-analytics -m "Merge feature/experimental-analytics"

# --- 7. SCENARIO: The Conflict Trap (Module 2) ---
# We create a branch that modifies a file, then modify the SAME line in main.
git checkout -b feature/update-header
echo "<h1>Welcome to Project Phoenix</h1>" > header.html
git add header.html
git commit -m "Create header component"

# Go back to main and edit the SAME file in a DIFFERENT way
git checkout main
echo "<h1>Project Phoenix Dashboard</h1>" > header.html
git add header.html
git commit -m "Create dashboard header"

# --- 8. SCENARIO: Surgical Extraction Target (Module 5) ---
# Create a branch with a file we want to 'restore' later without merging.
git checkout -b feature/cool-utils
echo "function helpful() { return 'I am helpful'; }" > utils.js
git add utils.js
git commit -m "Add helpful utility function"

# --- 9. SCENARIO: Branch Renaming Target (Module 6) ---
# Create a hotfix branch from the v1.0 tag so the Branch Renaming module
# can be run independently without relying on the Time Travel exercise.
git checkout v1.0
git switch -c hotfix/v1.0-security
echo "security_patch=true" >> config.env
git add config.env
git commit -m "Apply v1.0 security hotfix"

# --- 10. SCENARIO: Rebase & Cherry-Pick Target (Modules 7 & 8) ---
# Create a feature branch with multiple sequential commits that diverges
# from main. This supports both Advanced Rebasing and Bulk Cherry-Picking.
git checkout main
git checkout -b feature/my-work

echo "<form id='login'>" > login.html
echo "  <input type='text' name='user' />" >> login.html
echo "</form>" >> login.html
git add login.html
git commit -m "Add form component"

echo "  <div class='form-layout'>" >> login.html
git add login.html
git commit -m "Add form layout"

echo ".form { color: blue; }" > form-style.css
git add form-style.css
git commit -m "Add form styling"

echo "  <script>validate()</script>" >> login.html
git add login.html
git commit -m "Add final validation"

# --- 11. RETURN TO MAIN ---
git checkout main

# --- 12. VERIFICATION ---
echo ""
echo "========================================================="
echo "✅ SANDBOX READY: $PROJECT_DIR"
echo "========================================================="
echo "  Current Graph Topology:"
git log --oneline --graph --all --decorate
echo "========================================================="
echo "👉 Next Steps:"
echo "   1. cd $PROJECT_DIR"
echo "   2. Open this folder in VS Code: code ."
echo "========================================================="
```

### **2.3 Facilitator Notes on Infrastructure**

The script output provides the first opportunity for reassurance. When the participants run this, they will see a text-based representation of the commit graph. The facilitator should use this visualization to ground the abstract concepts.

**VS Code Note:** After running the script, participants should open the sandbox folder in VS Code (`code git-workshop-sandbox`). This gives them access to the built-in Source Control panel, the Git Graph extension (recommended), and the integrated terminal—all of which reinforce the concepts visually. Encourage participants to keep the Source Control panel open during exercises so they can see staging, conflicts, and diffs in real time.

**Analogy:** "Think of this script as building a movie set. We have constructed the buildings, the props, and the timeline. We are now standing on the main street. Down one alleyway is a conflict waiting to happen; down another is a feature we want to copy. We can walk down any of these paths safely because this is *our* set. If you burn it down, we just run the script again."

**Branch Inventory:** After setup, the following branches and tags are available:

| Branch / Tag | Purpose | Module(s) |
| :---- | :---- | :---- |
| `main` | The production trunk; starting point for all exercises | All |
| `feature/update-header` | Conflicting `header.html` change | 2 (Conflicts) |
| `feature/experimental-analytics` | Buggy feature already merged into `main` | 4 (Reverting) |
| `feature/cool-utils` | Contains `utils.js` for surgical extraction | 1 (Safe Merge), 5 (Extraction) |
| `hotfix/v1.0-security` | Pre-built hotfix from `v1.0` for renaming | 6 (Branch Renaming) |
| `feature/my-work` | Four sequential commits for rebase and cherry-pick | 7 (Rebasing), 8 (Cherry-Picking) |
| `v1.0` (tag) | Signpost for time travel exercises | 3 (Time Travel) |

## ---

**3. Module 1: Safe Merging (The "Try-Before-You-Buy" Approach)**

**Objective:** To dismantle the anxiety surrounding the `git merge` command by introducing non-committing merges and the psychological buffer of the staging area.

### **3.1 The Anxiety of the Enter Key**

For a novice, `git merge` feels like a leap of faith. The command is opaque; you type it, press enter, and the system either works silently or explodes in conflict markers. There is no intermediate step to verify what is about to happen. This binary outcome—success or failure—triggers performance anxiety. To combat this, we introduce the concept of "Safe Merging" using the `--no-commit` flag.

### **3.2 Theory: The Merge Commit vs. The Staged State**

In a standard merge, Git performs two actions:

1. **Integration:** It calculates the delta between the two branches and applies it to the working directory.  
2. **Commit:** It automatically creates a new commit object sealing the integration.

By separating these two actions, we give the developer control. Using `--no-commit` stops the process after step 1. The changes are placed in the Staging Area (Index), allowing the developer to inspect them using `git status` and `git diff` before finalizing the transaction. This is the "Fear-Free" equivalent of a fitting room—trying on the changes before purchasing them.

### **3.3 Interactive Exercise: The Safe Merge**

In this exercise, we will merge the `feature/cool-utils` branch into `main` using the safe approach.

**Step 1: Verify Position**

Ensure everyone is on the `main` branch.

```bash
git checkout main
```

**Step 2: The Safe Merge Command**

Instruct participants to run the merge with safety flags.

```bash
git merge --no-commit --no-ff feature/cool-utils
```

* **`--no-commit`**: Tells Git to perform the merge but stop before creating the commit.
* **`--no-ff`**: Forces a merge commit even if a fast-forward is possible. This is crucial for visualization. It ensures that the "bubble" of the feature branch remains visible in the history, preserving the context that this code came from a specific line of development.

**Step 3: The Inspection (Psychological Relief)**

Now, instead of a finished (and potentially wrong) merge, the shell returns control to the user.

```bash
git status
```

**Output Analysis:** The output will show new file: `utils.js` ready to be committed.

**Facilitator Analogy:** "Notice that nothing is permanent yet. You are holding the new file in your hands. You can look at it (`cat utils.js`), you can test it, or you can decide you don't want it (`git merge --abort`). You are in total control."

**Step 4: Finalizing the Merge (Accepting a Paused Merge)**

Once confident, the participant completes the process manually. There are two approaches:

*   **Use the default generated merge message:** Simply run `git commit` without the `-m` flag. Git will open your editor with a pre-populated message (e.g., `Merge branch 'feature/cool-utils'`). Save and close to accept it.
    ```bash
    git commit
    ```

*   **Write a custom commit message:** Provide your own description of the merge.
    ```bash
    git commit -m "Safely merged cool-utils with inspection"
    ```

**Facilitator Note:** Either approach is valid. The default message is useful because it automatically records the branch name—a detail that becomes important when investigating history later (see Section 3.5).

### **3.5 Identifying the Merged Branch After the Fact**

Branch names are *not* permanently encoded in Git's data model. Once a branch is deleted, its name is gone. However, you can still forensically determine what was merged into a merge commit using three techniques.

**Technique 1: Check the Default Commit Message**

If the developer accepted Git's default message, it will contain the branch name.

```bash
git log -1 <merge-commit-SHA>
```

Look for a message like: `Merge branch 'feature/cool-utils'`.

**Technique 2: Inspect the Second Parent Hash**

Every merge commit has two parents. Parent 1 is the branch you were *on* (e.g., `main`). Parent 2 is the branch you merged *in*.

```bash
git show <merge-commit-SHA>
```

The output will include a line like: `Merge: a1b2c3d e5f6g7h`. The second hash (`e5f6g7h`) is the tip of the merged branch at the time of the merge.

**Technique 3: Find Which Branch Contains That Parent**

Using the second parent hash from Technique 2, ask Git which branches contain that commit.

```bash
git branch --contains <second-parent-hash>
```

If the branch still exists, its name will appear. If it has been deleted, this will return nothing—but you still have the SHA for investigation.

**Insight:** These three techniques form a forensic toolkit. Even months after a merge, you can reconstruct *what* was merged and *from where*, reducing the anxiety of "I don't remember what that merge brought in."

### **3.6 Insight: The "Fast-Forward" Confusion**

A significant source of confusion for beginners is the "Fast-Forward" merge. When Git simply moves the pointer forward without creating a commit bubble, beginners often feel like they've "lost" the branch. They ask, "Where did my branch go?" By teaching `--no-ff` (No Fast Forward) as a default for beginners, we prioritize **explicitness over efficiency**. The resulting history clearly shows: "Here is where the work started, and here is where it finished." This visual confirmation reduces the anxiety of "losing" work in the timeline.

| Flag | Effect on History | Psychological Benefit |
| :---- | :---- | :---- |
| `git merge` (default) | May flatten history (Fast-Forward). | Efficient, but confusing. "Where did my branch go?" |
| `git merge --no-ff` | Always creates a merge commit bubble. | Explicit. "I can see exactly what I merged and when." |
| `git merge --no-commit` | Stages changes, does not commit. | Safe. "I can check the work before I sign off on it." |
| `git log -1 <merge>` | Shows the merge commit message. | Forensic. "I can trace what was merged, even later." |

## ---

**4. Module 2: Oops Scenarios (The Safety Net)**

**Objective:** To replace the fear of "breaking" the repository with the confidence of recovery, utilizing `git reflog` and conflict resolution strategies.

### **4.1 The Myth of the "Undo" Button**

New developers often search for a Ctrl+Z equivalent in Git. Finding none, they assume mistakes are permanent. In reality, Git has a far more powerful mechanism: the **Reflog** (Reference Log). The Reflog is a local recording of every time the HEAD pointer moves—whether by `commit`, `checkout`, `reset`, or `merge`. It is the black box flight recorder of the repository. Even if a user "deletes" a commit using `reset`, the commit object remains in the database, and the Reflog retains a pointer to it for a configurable period (usually 30-90 days).

### **4.2 Interactive Exercise: The Conflict Simulator**

We pre-seeded a conflict between `main` and `feature/update-header` in the `header.html` file. We will trigger it intentionally.

**Step 1: Initiating the Collision**

```bash
git merge feature/update-header
```

**Output:** `CONFLICT (add/add): Merge conflict in header.html` followed by `Automatic merge failed; fix conflicts and then commit the result.` **Facilitator Note:** This red text is a common panic trigger. Reframe this: "Git is not crashing. Git is being polite. It is saying, 'I have two valid options here, and I respect you too much to guess which one you want.' It is pausing for your expertise."

**Step 2: The Eject Button (`--abort`)**

Before fixing it, teach the escape route.

```bash
git merge --abort
```

This command returns the repository to the exact state before the merge command was typed. It is the ultimate "Never Mind" button. Knowing this button exists allows users to enter merges with lower stakes.

**Step 3: Resolving the Conflict**

Trigger the merge again. Open `header.html`.

```html
<<<<<<< HEAD
<h1>Project Phoenix Dashboard</h1>
=======
<h1>Welcome to Project Phoenix</h1>
>>>>>>> feature/update-header
```

**Anatomy of a Marker:**

* `<<<<<<< HEAD`: "This is what you have currently."  
* `=======`: "The dividing wall."  
* `>>>>>>> feature...`: "This is what is coming in."

**Action:** Edit the file to keep both or one (e.g., `<h1 class="dashboard">Welcome to Project Phoenix Dashboard</h1>`). Remove the markers. Save.

```bash
git add header.html
git commit -m "Resolved header conflict"
```

### **4.3 Interactive Exercise: The Catastrophic Reset**

Now, we will simulate the worst-case scenario: a developer accidentally resets their branch, apparently losing all their work.

**Step 1: The Destructive Act**

Instruction: "We are going to destroy the work we just did. We will reset the main branch back 3 commits, wiping out our merge and recent history."

```bash
git reset --hard HEAD~3
```

**Verification:** Run `git log`. The "Resolved header conflict" commit is gone. The file `header.html` is reverted. The panic is real.

**Step 2: The Safety Net (Reflog)**

Instruction: "Take a deep breath. We have the flight recorder."

```bash
git reflog
```

**Output Analysis:** The output will look like this:

```
a1b2c3d HEAD@{0}: reset: moving to HEAD~3  
e5f6g7h HEAD@{1}: commit (merge): Resolved header conflict  
...
```

The entry `HEAD@{1}` (or the one identifying the merge) is our lost state. The SHA `e5f6g7h` is the key to the time machine.

**Step 3: The Restoration**

```bash
git reset --hard HEAD@{1}
```

**Verification:** Run `git log`. The commit is back. The file is back. **Insight:** This moment—recovering "lost" data—is pivotal. It shifts the developer's mindset from "Git is fragile" to "Git is resilient." The realization that `reset --hard` is reversible via `reflog` eliminates the vast majority of Git-related anxiety.

### **4.4 The ORIG_HEAD Bookmark**

Before we leave the topic of recovery, there is an even simpler undo mechanism for the most common case: undoing the *last* dangerous operation.

**Theory:** Every time Git performs a potentially destructive operation—such as a `merge`, `rebase`, or `reset`—it automatically saves a temporary bookmark called `ORIG_HEAD`. This bookmark points to where HEAD was *just before* the operation. Think of it as Git dropping a pin on the map before you start hiking into unknown territory.

**Interactive Exercise: Undo via ORIG_HEAD**

**Step 1:** Perform a merge (or any operation).

```bash
git merge feature/update-header
```

**Step 2:** Realize you want to undo it immediately.

```bash
git reset --hard ORIG_HEAD
```

This moves the branch pointer back to the bookmark and forcefully overwrites the staging area and working directory to match.

**⚠️ WARNING:** `reset --hard` destroys uncommitted changes in your working directory. Always commit or stash before using it.

**When ORIG_HEAD breaks down:** If you perform *additional commits* after a dangerous operation and then run `git reset --hard ORIG_HEAD`, those new commits become **orphaned**—detached from any branch. They still exist in the reflog, but they are no longer reachable from the branch. For anything beyond the most recent operation, use the full `reflog` approach from Section 4.3.

### **4.5 Resetting to Match the Remote (The Nuclear Option)**

Sometimes the local branch is so far gone that the simplest fix is to wipe it clean and mirror the remote server exactly. This is the "factory reset" for a branch.

**Step 1: Fetch the latest state from the remote**

```bash
git fetch origin
```

This downloads the remote's history without changing your local branch.

**Step 2: Hard reset to the remote branch**

```bash
git reset --hard origin/<branch-name>
```

This forces your local branch pointer to match the remote exactly, discarding all local commits and changes.

**Step 3: Clean untracked files and directories**

```bash
git clean -fd
```

* `-f`: Force (required by default).
* `-d`: Also remove untracked directories.

**⚠️ WARNING:** This three-command sequence is irreversible for uncommitted work. Any local changes, untracked files, and local-only commits are destroyed. Use this only when you are certain the remote state is the "source of truth."

**Facilitator Analogy:** "This is like calling IT and saying, 'Wipe my laptop and re-image it from the server.' Everything local is gone, but you get a guaranteed clean state."

### **4.6 Undoing a Commit Without Losing Work (Soft Reset)**

A common scenario: a developer accidentally commits to a protected branch (like `main`) instead of their feature branch. They need to remove the commit but keep the file changes so they can move them to the correct branch.

**Step 1: The Soft Reset**

```bash
git reset --soft HEAD~1
```

* `--soft`: Removes the commit from history but **keeps all file modifications staged** (in the index). Nothing in the working directory changes.
* `HEAD~1`: "One commit before the current HEAD."

**Step 2: Move to the Correct Branch**

The changes are still staged and ready to go. Switch to the correct branch and re-commit.

```bash
git switch -c feature/my-accidental-work
git commit -m "Move accidental commit to the correct branch"
```

**Insight:** This is the gentlest form of reset. The spectrum of reset modes is:

| Mode | Removes Commit? | Resets Staging Area? | Resets Working Directory? |
| :---- | :---- | :---- | :---- |
| `--soft` | Yes | No (changes stay staged) | No |
| `--mixed` (default) | Yes | Yes (changes become unstaged) | No |
| `--hard` | Yes | Yes | Yes (changes are **destroyed**) |

Understanding this spectrum gives developers precise control over how much they want to "undo."

## ---

**5. Module 3: Time Travel (Detached HEADs and Archaeology)**

**Objective:** To reframe "Detached HEAD" from an error state to a powerful tool for historical inspection and experimentation.

### **5.1 The "Museum" Analogy**

When a developer checks out a specific commit (e.g., `git checkout v1.0`), Git enters "Detached HEAD" state. The CLI warning is ominous: *"You are in 'detached HEAD' state. You can look around..."*. **Analogy:** A Detached HEAD is like taking a book off the library shelf. You can read it (run the code), but you cannot write new chapters into the book while holding it in the aisle. If you want to write new chapters (commit changes), you must check out the book and take it to a desk (create a branch).

### **5.2 Interactive Exercise: Visiting the Past**

We will travel back in time to investigate the state of the code.

**Step 1: finding a specific point in time**

Sometimes you don't have a clean tag like `v1.0`. You just know "it worked 3 commits ago".

```bash
git log --oneline
```

Identify a SHA (the random string of characters) from a few commits back.

**Step 2: Detaching the HEAD (The Time Jump)**

Execute the jump using the SHA you found (e.g., `a1b2c3d`).

```bash
git checkout <commit-hash>
```

Alternatively, if you have a tag, you can use that (which is just a human-readable label for a SHA):

```bash
git checkout v1.0
```

**Output:** `HEAD is now at...` **Facilitator Note:** Validate the "scary" warning message. "Git is warning you that any commits you make here are ephemeral—they have no branch name to hold onto them. They are like notes written on a loose piece of paper."

**Step 3: Archaeology**

Verify the state of `config.env`. It should lack the `security_level` line added later.

```bash
cat config.env
```

**Step 4: Branching from the Past**

Scenario: "We realized version 1.0 had a bug. We need to fix it *for version 1.0*, without including all the new stuff in main."

This is a common "hotfix" scenario. We can convert our Detached HEAD into a new branch.

```bash
git switch -c hotfix/v1.0-security
```

(Note: `git switch -c` is the modern equivalent of `git checkout -b`). **Insight:** The "loose paper" (Detached HEAD) has now been filed into a folder (Branch). The HEAD is now attached. We have successfully created a parallel timeline starting from the past.

### **5.3 Psychological Safety in Exploration**

By normalizing the Detached HEAD state, we encourage developers to use `git checkout` (or `git switch --detach`) to debug regressions. Instead of guessing when a bug appeared, they can confidently hop through history, run the app, and see exactly when behavior changed, knowing they can always return to the present with `git switch main`.

## ---

**6. Module 4: Reverting (The Anti-Matter Commit)**

**Objective:** To teach non-destructive undoing for shared branches, with a specific focus on the complexity of reverting merge commits (the "Linus Torvalds" scenario).

### **6.1 Revert vs. Reset**

While `reset` is a time machine that rewinds history (great for local work), `revert` is a forward-moving operation. It creates a *new* commit that is the mathematical inverse of a previous commit. **Analogy:** If `reset` is erasing a sentence, `revert` is writing a new sentence that says, "Ignore the previous sentence." This is crucial for shared branches (like `main`), where rewriting history with `reset` would break the repository for other team members.

### **6.2 Interactive Exercise: The Simple Revert**

We have a commit in our history: "Harden security settings". Let's assume this caused an issue.

**Step 1: Find the Target**

```bash
git log --oneline
```

Locate the SHA for "Harden security settings".

**Step 2: Apply Anti-Matter**

```bash
git revert <SHA>
```

Git will open the default editor for the commit message. Save and close. **Result:** A new commit appears: Revert "Harden security settings". The file `config.env` is restored to its previous state, but the history of the mistake—and its correction—is preserved. This transparency builds trust in teams; hiding mistakes with `reset` erodes it.

### **6.3 Interactive Exercise: The Revert Paradox (Merge Commits)**

This is the advanced "Fear-Free" maneuver. We merged `feature/experimental-analytics` earlier. It contains a "bug". We need to revert the *merge commit*, not the individual commits inside it.

**Step 1: The Trap**

Try to revert the merge commit SHA directly.

```bash
git revert <SHA-of-merge>
```

**Error:** `error: commit is a merge but no -m option was given.` **Theory:** A merge commit has two parents: the previous `main` (Parent 1) and the tip of the feature branch (Parent 2). Git doesn't know which side you want to revert *to*. Do you want to strip away the feature (keep Parent 1) or strip away the main line (keep Parent 2)?

**Step 2: The Solution (-m 1)**

We almost always want to keep the "mainline" (Parent 1).

```bash
git revert -m 1 <SHA-of-merge>
```

**Facilitator Note:** "This tells Git: 'Keep the timeline of parent 1 safe, and treat everything brought in from parent 2 as the mistake to be removed.'"

**Step 3: The Linus Torvalds Scenario (The Trap of Reverting a Revert)** The report must highlight a critical downstream consequence described famously by Linus Torvalds.

* **Situation:** We reverted the feature. The code is gone from `main`.  
* **Future Problem:** The developer fixes the bug in `feature/experimental-analytics` and tries to merge it again.  
* **Result:** Git sees the SHA history and says, "I already merged these commits." It will merge the *new* fix, but it will *not* bring back the original code, because the revert commit is still active in `main`, explicitly negating that code.

**Step 4: The Fix**

To re-introduce the feature, we must **Revert the Revert**.

```bash
git revert <SHA-of-the-revert-commit>
```

Then, we can merge the feature branch again. **Insight:** This double-negative ("I undo the undoing of the feature") clears the path for the feature to exist again. Understanding this topology prevents the "Why is my code ignored?" panic that plagues advanced Git usage.

## ---

**7. Module 5: Comparison and Extraction (Surgical Precision)**

**Objective:** To transition from "blunt force" tools like merging to "surgical" tools like `restore` and `diff`, teaching how to copy specific files across branches.

### **7.1 The "Spare Parts" Analogy**

Sometimes a developer doesn't want to merge an entire branch; they just want one specific file or function from it. Merging is like buying a whole car just to get the tires. We need a way to just grab the tires. **Analogy:** "Imagine `main` is your house. `feature/cool-utils` is a neighbor's house. You like their front door. You don't want to merge their house into yours (that would be messy). You just want to copy the door."

### **7.2 Confusion: checkout vs. restore vs. switch**

Historically, `git checkout` was the "Swiss Army Knife"—it could switch branches *and* restore files. This overloading caused massive anxiety: "Will this command move me, or change my files?". Modern Git (2.23+) split these responsibilities to improve psychological safety:

* `git switch`: Only moves you (Safe).  
* `git restore`: Only changes files (Surgical).

### **7.3 Interactive Exercise: Surgical Extraction**

We want the `utils.js` file from `feature/cool-utils` without merging the whole branch.

**Step 1: Diagnosis (git diff)**

First, see what is different.

```bash
git diff feature/cool-utils -- utils.js
```

This shows the content of the file in the other dimension without affecting us.

**Step 2: The Old Way (checkout)**

Facilitators should mention this as it appears in legacy documentation.

```bash
git checkout feature/cool-utils -- utils.js
```

This command is syntactically confusing. "Checkout a branch... but just a file?"

**Step 3: The Fear-Free Way (restore)**

We use `restore` with a source.

```bash
git restore --source feature/cool-utils --worktree --staged utils.js
```

* `--source`: Where to grab it from.  
* `--worktree`: Put it in my folder.  
* `--staged`: Put it in my index (ready to commit).

**Step 4: Expanding the Scope (Multiple Files and Folders)**

The `restore` command is powerful enough to handle bulk operations.

*   **Multiple Files:** List them space-separated.
    ```bash
    git restore --source feature/cool-utils --worktree --staged utils.js config.env
    ```

*   **Entire Directories:** Use the path to the folder. Be careful, this acts recursively.
    ```bash
    git restore --source feature/cool-utils --worktree --staged src/components/
    ```

    **⚠️ CRITICAL WARNING:** unlike `checkout`, the `restore` command seeks to make your directory *exactly match* the source. This means **it will delete files** in your current folder if they do not exist in the source branch. Reference: `checkout` acts more like an overlay (adding/updating files), whereas `restore` acts like a reset (making the state identical).

**Legacy Equivalent (The Old Way):**

For completeness, here is how you used to do this with `checkout`. Note that `checkout` does not separate the staging area from the worktree as cleanly; it generally auto-stages the extracted files.

*   **Multiple Files:**
    ```bash
    git checkout feature/cool-utils -- utils.js config.env
    ```

*   **Entire Directories:**
    ```bash
    git checkout feature/cool-utils -- src/components/
    ```

**Insight:** Using `restore` aligns the command with the *intent*. "I am restoring this file from a version that exists elsewhere." This semantic clarity reduces the cognitive load required to parse the command's danger level.

### **7.4 Deep File History (Tracking Files Across Renames)**

Sometimes you need to investigate the full lifecycle of a file—including across renames. Standard `git log` stops at the point where the file was renamed, making it appear as if the file has no earlier history.

**The `--follow` Flag:**

```bash
git log --follow <file>
```

This tells Git to trace the file backwards through rename operations, using heuristic detection to find the original name. It reconstructs the full timeline of the file from its creation to the present.

**Adding Code Diffs:**

To see the actual code changes per commit (not just commit messages):

```bash
git log --follow -p <file>
```

* `-p` (or `--patch`): Shows the full diff for each commit that touched the file.

**Facilitator Analogy:** "Imagine a file was renamed from `old_utils.js` to `utils.js` six months ago. Without `--follow`, `git log utils.js` only shows the last six months. With `--follow`, you see the file's *entire* biography, all the way back to its birth under a different name."

### **7.5 Advanced Porting: Beyond Blunt Overwrites**

Section 7.3 taught the "blunt overwrite" approach—making a file exactly match another branch. But sometimes you need more surgical precision. Here are three porting strategies, ranked from blunt to precise.

**Strategy 1: Blunt Overwrite (Covered in 7.3)**

Makes the file *exactly* match the source branch. All local changes to that file are lost.

```bash
git checkout <source-branch> -- <file>
```

**Strategy 2: Apply Delta/Patch (Bring Only the Differences)**

Instead of overwriting, you can extract *just the differences* between your branch and the source, then apply them as a patch. This preserves your local changes to the file and layers the source's changes on top.

**Step 1:** Generate the patch. The three-dot syntax (`S...D`) computes the diff between the common ancestor and branch D.

```bash
git diff <your-branch>...<source-branch> -- <file> > changes.patch
```

**Step 2:** Apply the patch to your working directory.

```bash
git apply changes.patch
```

**Step 3:** Review and commit.

```bash
git add <file>
git commit -m "Applied targeted changes from <source-branch>"
```

**Insight:** This is like photocopying only the *annotations* from someone else's copy of a book, rather than replacing your entire book with theirs.

**Strategy 3: Copy the Commit (Cherry-Pick)**

If the change you want lives in a single commit on another branch, cherry-pick transplants that *exact commit* (with its message and author) onto your current branch.

```bash
git cherry-pick <commit-hash>
```

This creates a new commit on your branch with the same changes and message as the original (but a new SHA, since the parent is different). It preserves attribution and intent.

**When to Use Each Strategy:**

| Strategy | Use Case | Risk Level |
| :---- | :---- | :---- |
| Blunt Overwrite (`checkout --`) | "I want this file to be identical to the other branch." | Medium: overwrites local changes to the file. |
| Delta/Patch (`diff` + `apply`) | "I want to merge just the *differences* without losing my local edits." | Low: additive, may conflict. |
| Cherry-Pick | "I want to transplant one specific commit's changes." | Low: brings the exact commit with history. |

## ---

**8. Module 6: Branch Renaming (Local and Remote)**

**Objective:** To demystify the process of renaming branches, explain why local and remote renaming are separate operations, and establish the correct order of operations.

### **8.1 Why Renaming Feels Harder Than It Should**

Renaming a file on your computer is one click. Renaming a Git branch feels like defusing a bomb. The reason is architectural: Git is a *distributed* system. Your local repository and the remote (e.g., GitHub, GitLab) are independent copies of the data. There is no single "rename" command that updates both simultaneously. A local branch is just a pointer in your `.git/refs/heads/` directory—renaming it is instant and private. A remote branch, however, is a pointer on a completely separate server. Git has no native "rename remote branch" operation; instead, you must delete the old name and push the new one. Understanding this distinction eliminates the confusion.

### **8.2 Interactive Exercise: Renaming a Local Branch**

**Scenario:** We created `hotfix/v1.0-security` in the Time Travel module. The team has decided to standardize naming to `fix/v1.0-security-patch`.

**Option A: Rename the branch you are currently on**

If you are already on the branch you want to rename:

```bash
git branch -m fix/v1.0-security-patch
```

* `-m` stands for "move" (the same flag used by `mv` in Unix). It renames the branch in place.

**Option B: Rename a branch you are NOT on**

If you are on `main` and want to rename a different branch:

```bash
git branch -m hotfix/v1.0-security fix/v1.0-security-patch
```

The syntax is `git branch -m <old-name> <new-name>`.

**Verification:**

```bash
git branch
```

The old name is gone; the new name appears. All commit history is preserved—only the label changed.

### **8.3 Interactive Exercise: Renaming on the Remote**

If the old branch name has already been pushed to the remote, renaming locally is only half the job. The remote still has the old name.

**Step 1: Push the new name to the remote**

```bash
git push origin fix/v1.0-security-patch
```

This creates the branch under its new name on the remote.

**Step 2: Delete the old name from the remote**

```bash
git push origin --delete hotfix/v1.0-security
```

This removes the stale pointer. The remote now only knows the new name.

**Step 3: Reset the upstream tracking reference**

Your local branch may still be tracking the old remote name. Fix this:

```bash
git push origin -u fix/v1.0-security-patch
```

* `-u` (or `--set-upstream`) links your local branch to the new remote name so that future `git pull` and `git push` commands work without specifying the branch.

**Verification:**

```bash
git branch -vv
```

This shows each local branch and its upstream tracking reference. Confirm the new name appears.

### **8.4 Common Questions**

**Q: Should I rename locally first, then remote?**

Yes. Always rename locally first. The local rename (`git branch -m`) is instant, safe, and offline. Once confirmed, propagate the change to the remote. If you delete the remote branch first without renaming locally, your local branch still points to the old upstream—creating a confusing "gone" tracking state.

**Q: Why can't I just `git rename` the remote branch directly?**

Git's protocol has `push` and `fetch`, not `rename`. A remote branch is just a ref (a pointer to a commit SHA). The server doesn't support an atomic rename operation. The two-step "push new + delete old" pattern is the standard workflow used by GitHub, GitLab, and Bitbucket alike.

**Q: What happens to open Pull Requests / Merge Requests?**

If the old branch name is the source of an open PR/MR, deleting it from the remote will close or orphan that PR/MR (behavior varies by platform). Coordinate with your team before renaming branches that have active reviews. On GitHub, renaming the *default* branch (e.g., `master` → `main`) through the repository settings handles redirects automatically—but feature branches do not get this treatment.

**Q: Will other team members be affected?**

Yes. Anyone tracking the old branch name will see a "gone" upstream on their next `git fetch --prune`. They will need to run:

```bash
git fetch --prune
git switch fix/v1.0-security-patch
```

Communicate renames to the team before executing them on the remote.

### **8.5 Quick Reference**

| Task | Command |
| :---- | :---- |
| Rename current local branch | `git branch -m <new-name>` |
| Rename a different local branch | `git branch -m <old-name> <new-name>` |
| Push new name to remote | `git push origin <new-name>` |
| Delete old name from remote | `git push origin --delete <old-name>` |
| Update upstream tracking | `git push origin -u <new-name>` |
| Verify tracking references | `git branch -vv` |

## ---

**9. Module 7: Advanced Rebasing (The Timeline Rewriter)**

**Objective:** To demystify `git rebase` as a powerful history-editing tool, establish the Golden Rule for safe usage, and teach techniques for previewing a rebase before committing to it.

### **9.1 Merge vs. Rebase: Two Philosophies of Integration**

Both `merge` and `rebase` solve the same problem—integrating changes from one branch into another—but they produce fundamentally different histories.

* **Merge** preserves the timeline as it happened: a "diamond" shape showing divergence and convergence. It is honest but noisy.
* **Rebase** rewrites the timeline: it makes it *appear* as if the feature work happened sequentially after the latest `main`, producing a clean, linear history. It is tidy but revisionist.

**Analogy:** Merge is like stapling two documents together with a cover page that says "combined on this date." Rebase is like retyping the second document so its pages follow directly after the first, as if they were always one document.

### **9.2 How Rebase Works (Under the Hood)**

A rebase is *not* simply moving commits. It is an automated sequence of `cherry-pick` operations. When you run:

```bash
git checkout feature/my-work
git rebase main
```

Git performs these steps internally:

1. **Find the common ancestor** of `feature/my-work` and `main`.
2. **Save** each unique commit on `feature/my-work` (the commits since the divergence) as temporary patches.
3. **Reset** the `feature/my-work` branch pointer to the tip of `main`.
4. **Replay** each saved patch one by one on top of the new base, creating **new commits with new SHA-1 hashes**.

**Critical Insight:** Because the parent commit changes, every rebased commit gets a *new* SHA. The old SHAs become orphaned (but recoverable via `reflog`). This is why rebase is called "rewriting history."

### **9.3 The Golden Rule of Rebasing**

> **Never rebase a shared/public branch.**

If other developers have based their work on the commits you are about to rebase, their history will diverge from yours. They will have the old SHAs; you will have new ones. The next `push` or `pull` will produce a cascading mess of duplicate commits and conflicts.

**Safe to rebase:** Your local feature branch that only you work on, *before* pushing.
**Never rebase:** `main`, `develop`, or any branch that has been pushed and is shared with collaborators.

**Facilitator Analogy:** "Rebasing a shared branch is like editing a chapter in a book that other people are already reading and annotating. Their notes suddenly reference pages that no longer exist."

### **9.4 Interactive Exercise: A Simple Rebase**

**Scenario:** You have a feature branch that diverged from `main` a few commits ago. You want to "catch up" to the latest `main` without creating a merge commit.

**Step 1: Start the Rebase**

```bash
git checkout feature/my-work
git rebase main
```

If there are no conflicts, Git replays each commit silently and reports success.

**Step 2: If Conflicts Arise**

Rebase pauses on the conflicting commit. The process is similar to merge conflict resolution:

1. Edit the conflicting files to resolve the markers.
2. Stage the resolved files: `git add <file>`.
3. Continue the replay: `git rebase --continue`.

To abandon the rebase entirely and return to the pre-rebase state:

```bash
git rebase --abort
```

### **9.5 Previewing a Rebase Safely**

Unlike `merge`, there is no `--no-commit` equivalent for rebase. However, there are three techniques to preview the outcome before committing to it.

**Technique 1: The Sandbox Branch**

Create a disposable copy of your branch, rebase that, and inspect the result.

```bash
git checkout -b test-rebase feature/my-work
git rebase main
```

If the result looks good, delete the sandbox and rebase the real branch. If it looks bad, just delete the sandbox:

```bash
git checkout feature/my-work
git branch -D test-rebase
```

**Technique 2: Interactive Rebase with Breakpoints**

Interactive rebase (`git rebase -i`) opens an editor showing each commit as a line. You can insert `break` between lines to pause the rebase at specific points, or change `pick` to `edit` to pause after applying a specific commit.

```bash
git rebase -i main
```

The editor will show something like:

```
pick a1b2c3d Add login form
pick e5f6g7h Style the header
pick i9j0k1l Fix typo in README
```

Change it to:

```
pick a1b2c3d Add login form
break
pick e5f6g7h Style the header
edit i9j0k1l Fix typo in README
```

* `break`: Pauses the rebase at that point so you can inspect the state.
* `edit`: Applies the commit, then pauses so you can amend it or inspect before continuing.

Resume with `git rebase --continue` after each pause.

**Technique 3: The ORIG_HEAD Safety Net**

As covered in Section 4.4, Git sets the `ORIG_HEAD` bookmark before a rebase. If the result is unsatisfactory:

```bash
git reset --hard ORIG_HEAD
```

This instantly restores the branch to its pre-rebase state.

### **9.6 Quick Reference**

| Task | Command |
| :---- | :---- |
| Rebase onto main | `git rebase main` |
| Interactive rebase | `git rebase -i main` |
| Abort a rebase in progress | `git rebase --abort` |
| Continue after resolving conflict | `git rebase --continue` |
| Undo a completed rebase | `git reset --hard ORIG_HEAD` |
| Preview via sandbox branch | `git checkout -b test-rebase && git rebase main` |

## ---

**10. Module 8: Bulk Cherry-Picking (Transplanting Commit Ranges)**

**Objective:** To teach how to extract a specific range of commits from one branch and transplant them onto another, including how to squash multiple cherry-picked commits into a single clean commit.

### **10.1 When Merging is Too Much, and Single Cherry-Pick is Too Little**

Module 5 introduced cherry-picking a single commit (Section 7.5). But what if you need commits 4 through 8 from a feature branch, without taking commits 1 through 3 or commit 9? A full merge brings everything; a single cherry-pick brings one. Bulk cherry-picking fills the gap.

### **10.2 Cherry-Picking a Range of Commits**

**Syntax:**

```bash
git cherry-pick <oldest-hash>^..<newest-hash>
```

**The Caret (`^`) is Critical:** The `^` after the oldest hash means "start *from* this commit (inclusive)." Without it, Git interprets the range as "start *after* this commit (exclusive)," and the oldest commit is silently skipped.

| Syntax | Commits Included |
| :---- | :---- |
| `A^..D` | A, B, C, D (inclusive) |
| `A..D` | B, C, D (A is excluded) |

**Interactive Exercise:**

**Step 1:** Identify the commit range using `git log`.

```bash
git log --oneline feature/my-work
```

Suppose you see:

```
f1f1f1f (newest) Add final validation
e2e2e2e Add form styling
d3d3d3d Add form layout
c4c4c4c (oldest you want) Add form component
b5b5b5b Refactor utils (don't want this)
a6a6a6a Initial setup (don't want this)
```

You want commits `c4c4c4c` through `f1f1f1f`.

**Step 2:** Execute the cherry-pick range.

```bash
git cherry-pick c4c4c4c^..f1f1f1f
```

Git will replay each commit in order (`c4c4c4c`, `d3d3d3d`, `e2e2e2e`, `f1f1f1f`), creating new commits on your current branch with new SHAs.

**Step 3:** If a conflict occurs on any commit in the range, resolve it the same way as a rebase conflict:

1. Fix the conflict markers.
2. `git add <file>`.
3. `git cherry-pick --continue`.

To abort the entire operation: `git cherry-pick --abort`.

### **10.3 Squashing via Cherry-Pick**

Sometimes you want all the changes from a range of commits, but compressed into a *single* clean commit. This is useful for creating a tidy history on a release branch.

**Step 1:** Cherry-pick the range with `--no-commit`.

```bash
git cherry-pick --no-commit <oldest-hash>^..<newest-hash>
```

This grabs all the changes across the entire sequence and dumps them into your staging area at once, without creating individual commits.

**Step 2:** Verify the staged changes.

```bash
git status
git diff --cached
```

**Step 3:** Create one squashed commit.

```bash
git commit -m "Backport form feature (squashed from 4 commits)"
```

**Insight:** This achieves the same result as interactive rebase squashing, but without rewriting any existing branch history. It is purely additive—perfect for porting changes to a release branch without disturbing the source branch.

### **10.4 Quick Reference**

| Task | Command |
| :---- | :---- |
| Cherry-pick a single commit | `git cherry-pick <hash>` |
| Cherry-pick a range (inclusive) | `git cherry-pick <oldest>^..<newest>` |
| Squash a range into staging area | `git cherry-pick --no-commit <oldest>^..<newest>` |
| Continue after conflict | `git cherry-pick --continue` |
| Abort mid-operation | `git cherry-pick --abort` |

## ---

**11. Conclusion: Building a Culture of Resiliency**

This workshop curriculum is designed to do more than teach Git syntax; it is designed to rewire the emotional response to error. By simulating disaster in a controlled environment (the Holodeck), providing a guaranteed safety net (the Reflog), and using empathetic analogies (Museums, Fitting Rooms), we transform Git from a source of anxiety into a tool of empowerment.

The transition to a "Fear-Free" engineering culture requires consistent reinforcement. Facilitators should adopt the language of safety:

* Instead of "You broke the build," use "The state is currently unstable."  
* Instead of "Don't do that," use "Let's explore what happens if we do that in a sandbox."

As research in "Fear-Free" veterinary medicine has shown, reducing patient stress leads to faster healing and better cooperation. Similarly, reducing developer stress regarding version control leads to higher velocity, more frequent deployments, and a willingness to experiment with innovative code solutions. When the fear of the "Detached HEAD" is gone, the developer is free to explore the history of their project with the confidence of a time traveler who knows they can always find their way home.

### **11.1 Summary of Psychological Safety Tools**

| Concept | The Anxiety | The Fear-Free Reframe | The Technical Tool |
| :---- | :---- | :---- | :---- |
| **Merging** | "I will break the main branch forever." | "Try it on in the fitting room first." | `git merge --no-commit` |
| **Merge Forensics** | "I don't know what that old merge brought in." | "Check the parents and the commit message." | `git show <merge>`, `git branch --contains` |
| **Mistakes** | "I deleted everything and it's gone." | "We have a flight recorder." | `git reflog` |
| **Quick Undo** | "I need to undo what I just did, right now." | "Git already dropped a pin for you." | `git reset --hard ORIG_HEAD` |
| **Wrong Branch Commit** | "I committed to main by accident." | "Gently remove the commit, keep the work." | `git reset --soft HEAD~1` |
| **Remote Drift** | "My local branch is a mess compared to remote." | "Factory reset to the server's truth." | `git reset --hard origin/<branch>` |
| **Navigation** | "I am in a Detached HEAD limbo." | "I am visiting a museum exhibit." | `git switch --detach` |
| **Reverting** | "I can't undo a merge without breaking history." | "Apply anti-matter to the specific parent." | `git revert -m 1` |
| **Files** | "Checkout is confusing; I might switch branches." | "Surgically restore just the part you need." | `git restore --source` |
| **File History** | "The file was renamed; I can't see old history." | "Follow the file across its entire life." | `git log --follow -p` |
| **Porting Changes** | "I need *just* the differences, not a full overwrite." | "Photocopy only the annotations." | `git diff S...D \| git apply` |
| **Rebasing** | "Rebase will destroy my work." | "Test it on a sandbox branch first." | `git rebase -i`, `ORIG_HEAD` |
| **Cherry-Picking** | "I need commits 4-8 but not the rest." | "Transplant exactly the range you need." | `git cherry-pick <A>^..<B>` |

This guide serves as the foundational document for technical leads aiming to build high-trust, high-velocity teams through the mastery of their most fundamental tool.

#### **Works cited**

1. Overcome Programmer's Anxiety: How to Face Your Fears | by Ali Akhtari | Level Up Coding, accessed February 16, 2026, [https://levelup.gitconnected.com/overcoming-programmers-anxiety-how-to-face-your-fears-7beaa7f0e30d](https://levelup.gitconnected.com/overcoming-programmers-anxiety-how-to-face-your-fears-7beaa7f0e30d)  
2. Recovering from the Git detached HEAD state - CircleCI, accessed February 16, 2026, [https://circleci.com/blog/git-detached-head-state/](https://circleci.com/blog/git-detached-head-state/)  
3. Git Detached Head: What This Means and How to Recover - CloudBees, accessed February 16, 2026, [https://www.cloudbees.com/blog/git-detached-head](https://www.cloudbees.com/blog/git-detached-head)  
4. Tools for the Approach of Fear, Anxiety, and Stress in the Domestic Feline: An Update - PMC, accessed February 16, 2026, [https://pmc.ncbi.nlm.nih.gov/articles/PMC12349988/](https://pmc.ncbi.nlm.nih.gov/articles/PMC12349988/)  
5. Moving toward Fear-Free Husbandry and Veterinary Care for Horses - PMC, accessed February 16, 2026, [https://pmc.ncbi.nlm.nih.gov/articles/PMC9653666/](https://pmc.ncbi.nlm.nih.gov/articles/PMC9653666/)  
6. How to Resolve Merge Conflicts in Git? | Atlassian Git Tutorial, accessed February 16, 2026, [https://www.atlassian.com/git/tutorials/using-branches/merge-conflicts](https://www.atlassian.com/git/tutorials/using-branches/merge-conflicts)  
7. Psychological Safety Short Course - The GitLab Handbook, accessed February 16, 2026, [https://handbook.gitlab.com/handbook/leadership/emotional-intelligence/psychological-safety-short-course/](https://handbook.gitlab.com/handbook/leadership/emotional-intelligence/psychological-safety-short-course/)  
8. Psychological Safety | Tech Fleet User Guide, accessed February 16, 2026, [https://guide.techfleet.org/agile-training-portal/agile-handbook/agile-teamwork/making-strong-agile-teams/self-actualized-agile-teams/psychological-safety](https://guide.techfleet.org/agile-training-portal/agile-handbook/agile-teamwork/making-strong-agile-teams/self-actualized-agile-teams/psychological-safety)  
9. Reflog, your safety net - Gitready, accessed February 16, 2026, [https://gitready.com/intermediate/2009/02/09/reflog-your-safety-net.html](https://gitready.com/intermediate/2009/02/09/reflog-your-safety-net.html)  
10. Introducing Reflogs in Git | CodeSignal Learn, accessed February 16, 2026, [https://codesignal.com/learn/courses/advanced-git-features/lessons/introducing-reflogs-in-git](https://codesignal.com/learn/courses/advanced-git-features/lessons/introducing-reflogs-in-git)  
11. Conflict Resolution: Git Merge practice - DEV Community, accessed February 16, 2026, [https://dev.to/cookrdan/conflict-resolution-git-merge-practice-3iab](https://dev.to/cookrdan/conflict-resolution-git-merge-practice-3iab)  
12. no-commit - Git - merge-options Documentation, accessed February 16, 2026, [https://git-scm.com/docs/merge-options/2.22.1](https://git-scm.com/docs/merge-options/2.22.1)  
13. in 2025 which one should we use "Git switch" or "Git checkout" : r/AskProgramming - Reddit, accessed February 16, 2026, [https://www.reddit.com/r/AskProgramming/comments/1lvnfpf/in_2025_which_one_should_we_use_git_switch_or_git/](https://www.reddit.com/r/AskProgramming/comments/1lvnfpf/in_2025_which_one_should_we_use_git_switch_or_git/)  
14. Resetting, Checking Out & Reverting | Atlassian Git Tutorial, accessed February 16, 2026, [https://www.atlassian.com/git/tutorials/resetting-checking-out-and-reverting](https://www.atlassian.com/git/tutorials/resetting-checking-out-and-reverting)  
15. Basic Branching and Merging - Git, accessed February 16, 2026, [https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)  
16. Using git merge --no-ff - David Golden, accessed February 16, 2026, [https://xdg.me/git-merge-no-ff/](https://xdg.me/git-merge-no-ff/)  
17. git merge --no-ff - Reddit, accessed February 16, 2026, [https://www.reddit.com/r/git/comments/ylyr70/git_merge_noff/](https://www.reddit.com/r/git/comments/ylyr70/git_merge_noff/)  
18. How to teach Git | Hacker News, accessed February 16, 2026, [https://news.ycombinator.com/item?id=18919599](https://news.ycombinator.com/item?id=18919599)  
19. Why did my Git repo enter a detached HEAD state? - Stack Overflow, accessed February 16, 2026, [https://stackoverflow.com/questions/3965676/why-did-my-git-repo-enter-a-detached-head-state](https://stackoverflow.com/questions/3965676/why-did-my-git-repo-enter-a-detached-head-state)  
20. Ditch Git Checkout: Use Git Switch and Git Restore Instead - DEV Community, accessed February 16, 2026, [https://dev.to/zacharylee/ditch-git-checkout-use-git-switch-and-git-restore-instead-51e6](https://dev.to/zacharylee/ditch-git-checkout-use-git-switch-and-git-restore-instead-51e6)  
21. Undoing Changes in Git | Atlassian Git Tutorial, accessed February 16, 2026, [https://www.atlassian.com/git/tutorials/undoing-changes](https://www.atlassian.com/git/tutorials/undoing-changes)  
22. 1.1 Getting Started - About Version Control - Git, accessed February 16, 2026, [https://git-scm.com/book/ms/v2/Getting-Started-About-Version-Control](https://git-scm.com/book/ms/v2/Getting-Started-About-Version-Control)  
23. Psychological Safety in Teams Best Practices | 2026 - GitScrum, accessed February 16, 2026, [https://docs.gitscrum.com/en/best-practices/building-psychological-safety-in-teams/](https://docs.gitscrum.com/en/best-practices/building-psychological-safety-in-teams/)  
24. Undo a Git merge that hasn't been pushed yet - Stack Overflow, accessed February 16, 2026, [https://stackoverflow.com/questions/2389361/undo-a-git-merge-that-hasnt-been-pushed-yet](https://stackoverflow.com/questions/2389361/undo-a-git-merge-that-hasnt-been-pushed-yet)  
25. Reverting a Faulty Merge and Fixing the Feature Branch | by Edward Ezekiel - Medium, accessed February 16, 2026, [https://edezekiel.medium.com/reverting-a-faulty-merge-and-fixing-the-feature-branch-4d2fb437a9b5](https://edezekiel.medium.com/reverting-a-faulty-merge-and-fixing-the-feature-branch-4d2fb437a9b5)  
26. revert-a-faulty-merge How-To, accessed February 16, 2026, [https://schacon.github.io/git/howto/revert-a-faulty-merge.txt](https://schacon.github.io/git/howto/revert-a-faulty-merge.txt)  
27. Git -- Merge then Revert then trying to Merge again - Stack Overflow, accessed February 16, 2026, [https://stackoverflow.com/questions/61414492/git-merge-then-revert-then-trying-to-merge-again](https://stackoverflow.com/questions/61414492/git-merge-then-revert-then-trying-to-merge-again)  
28. Reverting a reverted merge in Git - Stack Overflow, accessed February 16, 2026, [https://stackoverflow.com/questions/74637705/reverting-a-reverted-merge-in-git](https://stackoverflow.com/questions/74637705/reverting-a-reverted-merge-in-git)  
29. Git & GitHub for Total Beginners: An Analogies-First Guide | by Tech Media. - Medium, accessed February 16, 2026, [https://medium.com/@tech.media.unicorn/git-github-for-total-beginners-an-analogies-first-guide-44b014632571](https://medium.com/@tech.media.unicorn/git-github-for-total-beginners-an-analogies-first-guide-44b014632571)  
30. Git Checkout – How to Checkout a File from Another Branch - freeCodeCamp, accessed February 16, 2026, [https://www.freecodecamp.org/news/git-checkout-file-from-another-branch/](https://www.freecodecamp.org/news/git-checkout-file-from-another-branch/)