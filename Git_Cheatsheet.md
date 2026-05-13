# 🐙 Ultimate Git Cheatsheet

This cheatsheet is divided into two sections. The first section contains the core commands you will use every single day as a developer. The second section contains a comprehensive list of commands for specific, advanced, or rare scenarios.

---

## 📅 Section 1: Day-to-Day Developer Essentials
*These are the commands you will use 95% of the time in your daily workflow.*

### 1. Checking Status
Before doing anything, you usually want to know the current state of your files.
```bash
git status
```
*Explanation:* Shows which files are modified, which are staged for commit, and what branch you are currently on.

### 2. Getting Latest Changes
Before starting new work, always pull the latest code from the remote repository.
```bash
git pull origin <branch-name>
```
*Explanation:* Fetches the newest changes from the remote server (like GitHub) and automatically merges them into your current local branch.

### 3. Creating & Switching Branches
Never work directly on `master` or `main`. Always create a feature branch.
```bash
# Create a new branch and switch to it immediately
git checkout -b feature/login-page

# Switch to an existing branch
git checkout master
```
*Note:* In newer versions of Git, you can also use `git switch <branch-name>` and `git switch -c <new-branch>`.

### 4. Staging Changes
You've made changes and want to prepare them to be saved.
```bash
# Stage a specific file
git add index.html

# Stage ALL changed and new files in the current directory
git add .
```
*Explanation:* Moves your changes from the "working directory" to the "staging area", preparing them for the next commit.

### 5. Committing Changes
Saving your staged changes into the local history.
```bash
git commit -m "feat: added login form validation"
```
*Explanation:* Wraps all your staged changes into a single snapshot (a commit) with a descriptive message.

### 6. Pushing Changes
Sending your local commits to the remote server so others can see them (or to create a Pull Request).
```bash
git push origin <branch-name>

# If pushing a brand new branch for the first time:
git push -u origin <branch-name>
```

### 7. Merging
Bringing changes from one branch into another.
```bash
# First, ensure you are on the branch you want to pull INTO (e.g., master)
git checkout master

# Then, merge the feature branch into master
git merge feature/login-page
```

---

## 🛠️ Section 2: The Comprehensive Command Reference
*For advanced use cases, fixing mistakes, and deep configuration.*

### ⚙️ Setup & Configuration
```bash
# Set your name and email globally
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Turn a normal folder into a Git repository
git init

# Clone an existing repository
git clone https://github.com/user/repo.git
```

### 🔍 Viewing History & Diffs
```bash
# View the commit history
git log

# View a clean, one-line summary of commits
git log --oneline --graph

# See exactly what lines of code changed in the working directory
git diff

# See what lines of code changed in the STAGING area
git diff --staged
```

### 📦 Stashing (Saving work temporarily)
When you are in the middle of a task but need to switch branches quickly to fix a bug, you can "stash" your half-done work.
```bash
# Save modified and staged changes temporarily
git stash

# View your stashes
git stash list

# Bring the stashed changes back and delete the stash
git stash pop

# Delete the most recent stash without applying it
git stash drop
```

### ⏪ Undoing Changes (Local/Uncommitted)
```bash
# Discard all local, unstaged changes in a file (revert to last commit)
git restore <file-name>
# OR the older command:
git checkout -- <file-name>

# Unstage a file (move it out of the staging area, but keep the modifications)
git restore --staged <file-name>
# OR the older command:
git reset HEAD <file-name>
```

### 🔙 Undoing Commits (Careful!)
```bash
# Undo the latest commit, but KEEP the files modified in your working directory
git reset --soft HEAD~1

# Undo the latest commit AND DESTROY the files/modifications entirely (Warning!)
git reset --hard HEAD~1

# Safely undo a pushed commit by creating a NEW commit that does the exact opposite
git revert <commit-hash>
```

### 🔀 Advanced Branching & Remote Management
```bash
# List all local branches
git branch

# List all local and remote branches
git branch -a

# Delete a local branch safely (Git prevents deletion if unmerged)
git branch -d <branch-name>

# Force delete a local branch
git branch -D <branch-name>

# View remote connections
git remote -v

# Add a new remote connection
git remote add origin <url>
```

### 🍒 Cherry-Picking & Rebasing
```bash
# Take a specific commit from another branch and apply it to your current branch
git cherry-pick <commit-hash>

# Rebase your current branch on top of master (Rewrite history for a linear graph)
# WARNING: Do not rebase branches that have already been pushed/shared with others!
git rebase master
```

### 🏷️ Tagging (Releases)
```bash
# List all tags
git tag

# Create a new tag at the current commit
git tag v1.0.0

# Push tags to the remote repository
git push origin --tags
```
