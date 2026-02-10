# Git Basics

## 🎯 Version Control System (VCS)

Git là một **distributed version control system** được tạo bởi Linus Torvalds (người tạo Linux) năm 2005.

### **Tại Sao Cần Git?**

- 📜 Track history của code
- 🔄 Collaboration giữa nhiều developers  
- 🌿 Branching & merging dễ dàng
- 🔙 Rollback khi có lỗi
- 💾 Backup code
- 🚀 Industry standard (99% companies sử dụng)

## 🏗️ Git Architecture

```
┌─────────────────────────────────────────────┐
│         Working Directory                    │
│    (Files you're editing)                   │
└───────────────┬─────────────────────────────┘
                │ git add
                ↓
┌─────────────────────────────────────────────┐
│         Staging Area (Index)                │
│    (Files ready to commit)                  │
└───────────────┬─────────────────────────────┘
                │ git commit
                ↓
┌─────────────────────────────────────────────┐
│         Local Repository                     │
│    (Committed changes)                      │
└───────────────┬─────────────────────────────┘
                │ git push
                ↓
┌─────────────────────────────────────────────┐
│         Remote Repository                    │
│    (GitHub, GitLab, Bitbucket)              │
└─────────────────────────────────────────────┘
```

## 🚀 Getting Started

### **Installation**

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install git

# CentOS/RHEL
sudo yum install git

# MacOS
brew install git

# Verify installation
git --version
```

### **Initial Configuration**

```bash
# Set your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default editor
git config --global core.editor "vim"

# Set default branch name
git config --global init.defaultBranch main

# View config
git config --list
git config user.name

# Config file location
cat ~/.gitconfig
```

## 📦 Basic Workflow

### **1. Create Repository**

```bash
# Create new repo
mkdir myproject
cd myproject
git init

# Clone existing repo
git clone https://github.com/user/repo.git
git clone https://github.com/user/repo.git mydir  # Custom directory name
```

### **2. Check Status**

```bash
# View status
git status

# Short status
git status -s
# Output:
#  M modified-file.txt
# ?? untracked-file.txt
# A  new-file.txt
```

### **3. Add Files to Staging**

```bash
# Add specific file
git add file.txt

# Add multiple files
git add file1.txt file2.txt

# Add all files
git add .
git add -A

# Add by pattern
git add *.py
git add src/

# Interactive add
git add -i

# Patch mode (select chunks)
git add -p
```

### **4. Commit Changes**

```bash
# Commit with message
git commit -m "Add user authentication feature"

# Commit with detailed message
git commit -m "Add login functionality" -m "- Added login form
- Implemented JWT authentication
- Added password hashing"

# Add and commit in one step
git commit -am "Update README"

# Amend last commit
git commit --amend -m "New message"

# Empty commit (for triggering CI)
git commit --allow-empty -m "Trigger CI"
```

### **5. View History**

```bash
# View commit history
git log

# One line per commit
git log --oneline

# Graph view
git log --oneline --graph --all

# Last n commits
git log -n 5

# By author
git log --author="John"

# By date
git log --since="2026-01-01"
git log --since="2 weeks ago"

# Search commit message
git log --grep="bug fix"

# Show changes
git log -p

# Show stats
git log --stat

# Beautiful format
git log --pretty=format:"%h - %an, %ar : %s"
```

### **6. View Changes**

```bash
# Unstaged changes
git diff

# Staged changes
git diff --staged
git diff --cached

# Changes between branches
git diff main..feature

# Changes between commits
git diff HEAD~2 HEAD

# Specific file
git diff file.txt

# Word diff
git diff --word-diff
```

## 🌿 Branching

### **Why Branching?**

- Work on features independently
- Keep main branch stable
- Easy collaboration
- Safe experimentation

### **Branch Commands**

```bash
# List branches
git branch

# List all branches (including remote)
git branch -a

# Create branch
git branch feature-login

# Switch to branch
git checkout feature-login
git switch feature-login  # Modern way

# Create and switch
git checkout -b feature-login
git switch -c feature-login  # Modern way

# Rename branch
git branch -m old-name new-name

# Delete branch
git branch -d feature-login  # Safe delete
git branch -D feature-login  # Force delete

# Delete remote branch
git push origin --delete feature-login
```

## 🔀 Merging

### **Merge Strategies**

#### **1. Fast-Forward Merge**

```bash
# When no divergent changes
git checkout main
git merge feature

# Diagram:
main:    A---B---C
                  \
feature:           D---E
# After merge:
main:    A---B---C---D---E
```

#### **2. Three-Way Merge**

```bash
git checkout main
git merge feature

# Diagram:
main:    A---B---C---F
              \
feature:       D---E
# After merge:
main:    A---B---C---F---M (merge commit)
              \         /
feature:       D---E---
```

### **Handling Merge Conflicts**

```bash
# Start merge
git merge feature-branch

# Conflicts occur
Auto-merging file.txt
CONFLICT (content): Merge conflict in file.txt
Automatic merge failed; fix conflicts and then commit the result.

# Check conflicted files
git status

# File will look like:
<<<<<<< HEAD
Current branch content
=======
Feature branch content
>>>>>>> feature-branch

# Resolve manually, then:
git add file.txt
git commit -m "Merge feature-branch"

# Or abort merge
git merge --abort
```

## 🏷️ Tagging

```bash
# Lightweight tag
git tag v1.0.0

# Annotated tag (recommended)
git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag specific commit
git tag -a v1.0.0 9fceb02 -m "Version 1.0.0"

# List tags
git tag

# Show tag info
git show v1.0.0

# Push tags
git push origin v1.0.0
git push origin --tags  # All tags

# Delete tag
git tag -d v1.0.0
git push origin --delete v1.0.0
```

## 🌐 Remote Operations

### **Remote Management**

```bash
# View remotes
git remote
git remote -v

# Add remote
git remote add origin https://github.com/user/repo.git

# Remove remote
git remote remove origin

# Rename remote
git remote rename origin upstream

# Change remote URL
git remote set-url origin https://github.com/user/new-repo.git
```

### **Push & Pull**

```bash
# Push to remote
git push origin main

# Push all branches
git push --all origin

# Push tags
git push --tags

# Force push (dangerous!)
git push -f origin main

# Pull from remote
git pull origin main

# Pull with rebase
git pull --rebase origin main

# Fetch (download without merge)
git fetch origin

# Fetch all remotes
git fetch --all
```

## 🔄 Undoing Changes

### **Undo Working Directory Changes**

```bash
# Discard changes in file
git checkout -- file.txt
git restore file.txt  # Modern way

# Discard all changes
git checkout -- .
git restore .
```

### **Undo Staged Changes**

```bash
# Unstage file
git reset HEAD file.txt
git restore --staged file.txt  # Modern way

# Unstage all
git reset HEAD
```

### **Undo Commits**

```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (unstage changes)
git reset --mixed HEAD~1
git reset HEAD~1  # Default

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Undo to specific commit
git reset --hard abc123

# Create new commit that undoes changes (safe)
git revert HEAD
git revert abc123
```

## 📋 .gitignore

```bash
# Create .gitignore file
cat > .gitignore << 'EOF'
# OS files
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp

# Dependencies
node_modules/
venv/
__pycache__/
*.pyc

# Build outputs
dist/
build/
*.exe
*.dll

# Logs
*.log
logs/

# Environment
.env
.env.local
secrets.yml

# Temp files
*.tmp
*.bak
*~
EOF

# Global gitignore
git config --global core.excludesfile ~/.gitignore_global
```

## 🎓 Best Practices

### **Commit Messages**

```bash
# Good commit messages
git commit -m "feat: add user authentication"
git commit -m "fix: resolve login bug for IE11"
git commit -m "docs: update API documentation"
git commit -m "refactor: simplify database queries"

# Conventional Commits format:
# <type>(<scope>): <subject>
#
# Types:
# feat: New feature
# fix: Bug fix
# docs: Documentation
# style: Formatting
# refactor: Code restructuring
# test: Adding tests
# chore: Maintenance
```

### **Branching Strategy**

```bash
# GitFlow naming convention
main/master          # Production
develop              # Development
feature/login        # New features
hotfix/security-fix  # Urgent fixes
release/v1.2.0       # Release preparation
```

### **Regular Operations**

```bash
# Daily workflow
git pull origin main
git checkout -b feature/new-feature
# ... make changes ...
git add .
git commit -m "feat: implement new feature"
git push origin feature/new-feature
# Create Pull Request on GitHub/GitLab
```

## ✅ Quick Reference

```bash
# Initialize & Clone
git init
git clone <url>

# Status & Diff
git status
git diff

# Add & Commit
git add .
git commit -m "message"

# Branch
git branch <name>
git checkout <name>
git merge <branch>

# Remote
git push origin <branch>
git pull origin <branch>

# Undo
git restore <file>
git reset --hard HEAD~1

# History
git log --oneline
```

---

**Tiếp theo**: [Git Commands Advanced →](../commands/git-commands.md)
