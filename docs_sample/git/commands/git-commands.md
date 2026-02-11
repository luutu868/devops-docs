# Git Commands Reference

## Repository Setup

### Khởi tạo Repository

```bash
# Create new repository
git init
git init my-project

# Clone existing repository
git clone https://github.com/user/repo.git
git clone https://github.com/user/repo.git my-folder

# Clone specific branch
git clone -b develop https://github.com/user/repo.git

# Shallow clone (save bandwidth)
git clone --depth 1 https://github.com/user/repo.git

# Clone with submodules
git clone --recursive https://github.com/user/repo.git
```

### Remote Management

```bash
# View remotes
git remote
git remote -v

# Add remote
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git

# Change remote URL
git remote set-url origin https://github.com/user/new-repo.git

# Remove remote
git remote remove origin

# Rename remote
git remote rename origin old-origin

# Show remote info
git remote show origin
```

## Working with Changes

### Check Status

```bash
# View status
git status
git status -s  # Short format
git status -sb # Short with branch

# Example output:
 M file1.txt    # Modified, not staged
M  file2.txt    # Modified, staged
A  file3.txt    # Added
D  file4.txt    # Deleted
?? file5.txt    # Untracked
```

### Staging Files

```bash
# Stage specific file
git add file.txt

# Stage multiple files
git add file1.txt file2.txt file3.txt

# Stage all changes
git add .
git add -A
git add --all

# Stage by pattern
git add *.js
git add src/

# Stage interactively
git add -i

# Stage patch (choose hunks)
git add -p file.txt
git add --patch file.txt
```

### Unstaging Files

```bash
# Unstage file
git reset HEAD file.txt
git restore --staged file.txt

# Unstage all
git reset HEAD
```

### Committing Changes

```bash
# Commit with message
git commit -m "Add new feature"

# Commit with detailed message
git commit -m "Add user authentication" -m "- Implement JWT tokens" -m "- Add password hashing"

# Commit all tracked changes (skip staging)
git commit -a -m "Update all files"
git commit -am "Quick fix"

# Amend last commit
git commit --amend -m "New commit message"

# Amend without changing message
git commit --amend --no-edit

# Empty commit (for triggering CI)
git commit --allow-empty -m "Trigger CI"
```

### Viewing Differences

```bash
# View unstaged changes
git diff

# View staged changes
git diff --staged
git diff --cached

# Compare specific file
git diff file.txt
git diff HEAD file.txt

# Compare branches
git diff main develop
git diff main..develop

# Compare commits
git diff abc123 def456
git diff HEAD~2 HEAD

# Show diff stats
git diff --stat

# Word diff
git diff --word-diff

# Show diff with context
git diff -U10  # 10 lines of context
```

## Viewing History

### Git Log

```bash
# View commit history
git log

# One line per commit
git log --oneline

# Graph view
git log --graph
git log --oneline --graph --all

# Limit number of commits
git log -5
git log -n 5

# Show commits in date range
git log --since="2 weeks ago"
git log --since="2023-01-01" --until="2023-12-31"

# Show commits by author
git log --author="John"
git log --author="john@example.com"

# Search commit messages
git log --grep="fix"
git log --grep="bug" --grep="fix" --all-match

# Show commits that modified file
git log -- file.txt
git log --follow file.txt  # Follow renames

# Show commits in range
git log main..develop
git log origin/main..HEAD

# Pretty format
git log --pretty=format:"%h - %an, %ar : %s"
git log --pretty=format:"%h %ad | %s%d [%an]" --date=short

# Show file changes
git log --stat
git log --name-only
git log --name-status

# Show patches
git log -p
git log -p -2  # Last 2 commits with patches
```

### Git Show

```bash
# Show commit details
git show HEAD
git show abc123

# Show specific file in commit
git show abc123:file.txt

# Show file at branch
git show main:src/app.js

# Show changes in commit
git show abc123 --stat
```

### Git Blame

```bash
# Show who changed each line
git blame file.txt

# Show line range
git blame -L 10,20 file.txt

# Ignore whitespace
git blame -w file.txt

# Show email instead of name
git blame -e file.txt
```

### Search in History

```bash
# Search for string in history
git log -S "function_name"
git log -S "text" --source --all

# Search for regex
git log -G "pattern.*"

# Find when line was added/removed
git log -S "def my_function" --source --all
```

## Branching

### Create and Switch Branches

```bash
# List branches
git branch
git branch -a  # All (including remote)
git branch -r  # Remote only

# Create branch
git branch feature-login

# Create and switch
git checkout -b feature-login
git switch -c feature-login  # Modern way

# Switch to existing branch
git checkout main
git switch main  # Modern way

# Create from specific commit
git branch bugfix abc123
git checkout -b bugfix abc123

# Create from remote branch
git checkout -b feature origin/feature
git checkout --track origin/feature
```

### Branch Management

```bash
# Rename branch
git branch -m old-name new-name
git branch -m new-name  # Rename current branch

# Delete branch
git branch -d feature-login  # Safe delete
git branch -D feature-login  # Force delete

# Delete remote branch
git push origin --delete feature-login
git push origin :feature-login

# Show merged branches
git branch --merged
git branch --no-merged

# Show branch with last commit
git branch -v
git branch -vv  # With upstream
```

### Tracking Branches

```bash
# Set upstream branch
git branch -u origin/main
git branch --set-upstream-to=origin/main

# Push and set upstream
git push -u origin feature-login

# View tracking branches
git branch -vv

# Unset upstream
git branch --unset-upstream
```

## Merging

### Basic Merge

```bash
# Merge branch into current
git merge feature-login

# Merge with no fast-forward
git merge --no-ff feature-login

# Merge with squash
git merge --squash feature-login
git commit -m "Merge feature-login"

# Abort merge
git merge --abort
```

### Merge Strategies

```bash
# Recursive (default)
git merge -s recursive feature

# Ours (keep our version)
git merge -s ours feature
git merge -X ours feature  # Recursive with ours option

# Theirs (take their version)
git merge -X theirs feature
```

### Resolving Conflicts

```bash
# View conflicts
git status
git diff

# Choose version
git checkout --ours file.txt
git checkout --theirs file.txt

# Use merge tool
git mergetool

# After resolving
git add file.txt
git commit

# Abort merge
git merge --abort
```

## Rebasing

### Basic Rebase

```bash
# Rebase current branch onto main
git rebase main

# Continue after resolving conflicts
git rebase --continue

# Skip commit
git rebase --skip

# Abort rebase
git rebase --abort
```

### Interactive Rebase

```bash
# Rebase last 3 commits
git rebase -i HEAD~3

# Rebase from specific commit
git rebase -i abc123

# In editor:
pick abc123 Commit message 1
squash def456 Commit message 2  # Squash into previous
reword ghi789 Commit message 3  # Edit message
edit jkl012 Commit message 4    # Stop for amend
drop mno345 Commit message 5    # Drop commit
```

**Commands in interactive rebase:**
- `pick`: Keep commit
- `reword`: Keep commit, edit message
- `edit`: Keep commit, stop for amending
- `squash`: Combine with previous commit
- `fixup`: Like squash, discard message
- `drop`: Remove commit

### Rebase vs Merge

**Merge:**
```bash
# Preserves history
git checkout main
git merge feature
# Creates merge commit
```

**Rebase:**
```bash
# Linear history
git checkout feature
git rebase main
git checkout main
git merge feature  # Fast-forward
```

## Remote Operations

### Fetching

```bash
# Fetch from origin
git fetch

# Fetch from specific remote
git fetch upstream

# Fetch all remotes
git fetch --all

# Fetch and prune deleted branches
git fetch -p
git fetch --prune

# Fetch tags
git fetch --tags
```

### Pulling

```bash
# Pull (fetch + merge)
git pull

# Pull with rebase
git pull --rebase

# Pull from specific remote/branch
git pull origin main

# Pull and prune
git pull -p

# Pull all branches
git pull --all
```

### Pushing

```bash
# Push to origin
git push

# Push to specific remote/branch
git push origin main

# Push and set upstream
git push -u origin feature

# Push all branches
git push --all

# Push tags
git push --tags

# Push specific tag
git push origin v1.0.0

# Force push (dangerous!)
git push --force
git push -f

# Force push with lease (safer)
git push --force-with-lease

# Delete remote branch
git push origin --delete feature
git push origin :feature

# Push to all remotes
git remote | xargs -L1 git push --all
```

## Stashing

### Save and Apply Stash

```bash
# Stash changes
git stash
git stash save "Work in progress"

# Stash including untracked files
git stash -u
git stash --include-untracked

# Stash including ignored files
git stash -a
git stash --all

# List stashes
git stash list

# Show stash contents
git stash show
git stash show -p  # With patch
git stash show stash@{1}

# Apply stash
git stash apply
git stash apply stash@{1}

# Apply and remove
git stash pop
git stash pop stash@{1}

# Create branch from stash
git stash branch feature-name stash@{0}
```

### Managing Stashes

```bash
# Clear single stash
git stash drop
git stash drop stash@{1}

# Clear all stashes
git stash clear

# Create stash with message
git stash push -m "Work on feature X"

# Stash specific files
git stash push -m "Stash only these files" file1.txt file2.txt

# Apply stash to different branch
git checkout other-branch
git stash apply stash@{0}
```

## Tagging

### Create Tags

```bash
# Lightweight tag
git tag v1.0.0

# Annotated tag (recommended)
git tag -a v1.0.0 -m "Version 1.0.0"

# Tag specific commit
git tag v0.9.0 abc123

# Tag with GPG signature
git tag -s v1.0.0 -m "Signed version"
```

### View Tags

```bash
# List tags
git tag
git tag -l
git tag -l "v1.*"

# Show tag info
git show v1.0.0

# List with messages
git tag -n
git tag -n5  # 5 lines of message
```

### Manage Tags

```bash
# Delete local tag
git tag -d v1.0.0

# Delete remote tag
git push origin --delete v1.0.0
git push origin :refs/tags/v1.0.0

# Push tag
git push origin v1.0.0

# Push all tags
git push --tags
git push origin --tags

# Fetch tags
git fetch --tags

# Checkout tag
git checkout v1.0.0
git checkout tags/v1.0.0 -b branch-name
```

## Undoing Changes

### Discard Changes

```bash
# Discard unstaged changes
git checkout -- file.txt
git restore file.txt

# Discard all unstaged changes
git checkout -- .
git restore .

# Discard staged changes
git reset HEAD file.txt
git restore --staged file.txt
```

### Reset

```bash
# Soft reset (keep changes staged)
git reset --soft HEAD~1

# Mixed reset (keep changes unstaged) - default
git reset HEAD~1
git reset --mixed HEAD~1

# Hard reset (discard all changes)
git reset --hard HEAD~1
git reset --hard origin/main

# Reset to specific commit
git reset --hard abc123

# Reset file to specific commit
git checkout abc123 -- file.txt
```

### Revert

```bash
# Revert commit (create new commit)
git revert HEAD
git revert abc123

# Revert without committing
git revert -n abc123
git revert --no-commit abc123

# Revert merge commit
git revert -m 1 abc123

# Revert range
git revert HEAD~3..HEAD
```

### Clean

```bash
# Remove untracked files (dry run)
git clean -n
git clean --dry-run

# Remove untracked files
git clean -f
git clean --force

# Remove untracked files and directories
git clean -fd

# Remove ignored files too
git clean -fdx

# Interactive clean
git clean -i
```

## Advanced Commands

### Cherry-pick

```bash
# Apply specific commit
git cherry-pick abc123

# Cherry-pick multiple commits
git cherry-pick abc123 def456

# Cherry-pick range
git cherry-pick abc123..def456

# Cherry-pick without committing
git cherry-pick -n abc123
```

### Reflog

```bash
# View reflog
git reflog
git reflog show HEAD

# View reflog for branch
git reflog show main

# Recover lost commit
git reflog
git checkout abc123
git branch recovered-branch abc123
```

### Bisect

```bash
# Start bisect
git bisect start

# Mark bad commit
git bisect bad

# Mark good commit
git bisect good abc123

# Git will checkout middle commit
# Test and mark:
git bisect good  # or bad

# Reset bisect
git bisect reset

# Automated bisect
git bisect start HEAD abc123
git bisect run ./test.sh
```

### Submodules

```bash
# Add submodule
git submodule add https://github.com/user/repo.git libs/repo

# Initialize submodules
git submodule init

# Update submodules
git submodule update

# Clone with submodules
git clone --recursive https://github.com/user/repo.git

# Update all submodules
git submodule update --remote

# Update specific submodule
git submodule update --remote libs/repo

# Remove submodule
git submodule deinit libs/repo
git rm libs/repo
rm -rf .git/modules/libs/repo
```

### Worktree

```bash
# Add worktree
git worktree add ../feature-branch feature

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../feature-branch

# Prune worktrees
git worktree prune
```

## Configuration

### User Configuration

```bash
# Set username and email
git config --global user.name "John Doe"
git config --global user.email "john@example.com"

# Set editor
git config --global core.editor "vim"
git config --global core.editor "code --wait"

# View configuration
git config --list
git config --global --list
git config user.name
```

### Aliases

```bash
# Create aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.lg 'log --oneline --graph --all'
git config --global alias.undo 'reset --soft HEAD~1'

# Use alias
git st
git lg
```

### Other Settings

```bash
# Default branch name
git config --global init.defaultBranch main

# Line endings
git config --global core.autocrlf true   # Windows
git config --global core.autocrlf input  # Mac/Linux

# Colors
git config --global color.ui auto

# Push default
git config --global push.default simple

# Pull rebase
git config --global pull.rebase true

# Credential helper
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'

# Diff tool
git config --global diff.tool vimdiff
git config --global merge.tool vimdiff
```

## Useful Combinations

```bash
# View changes in last commit
git show HEAD
git log -1 -p

# Undo last commit but keep changes
git reset --soft HEAD~1

# Undo last commit and discard changes
git reset --hard HEAD~1

# Amend last commit with new changes
git add file.txt
git commit --amend --no-edit

# Create branch and push
git checkout -b feature
git push -u origin feature

# Update feature branch with main
git checkout feature
git rebase main (or git merge main)

# Squash last 3 commits
git rebase -i HEAD~3

# View file from another branch
git show main:file.txt

# Compare file between branches
git diff main..feature -- file.txt

# List files in commit
git diff-tree --no-commit-id --name-only -r abc123

# Count commits
git rev-list --count HEAD
git rev-list --count main..feature

# Show branches containing commit
git branch --contains abc123

# Show most recent branch
git for-each-ref --sort=-committerdate refs/heads/

# Find large files
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  awk '/^blob/ {print substr($0,6)}' | sort -k2nr | head
```

## .gitignore

```bash
# .gitignore syntax

# Ignore specific file
file.txt

# Ignore directory
logs/
node_modules/

# Ignore pattern
*.log
*.tmp
*.swp

# Ignore in subdirectories
**/node_modules/

# Negation (don't ignore)
!important.log

# Ignore files in root only
/config.local

# Comments
# This is a comment
```

**Common patterns:**
```gitignore
# Dependencies
node_modules/
vendor/
bower_components/

# Build output
dist/
build/
*.min.js
*.min.css

# Logs
*.log
logs/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local

# Temporary
tmp/
temp/
*.tmp
```

## Git Cheat Sheet

**Setup:**
```bash
git init
git clone <url>
git config --global user.name "name"
git config --global user.email "email"
```

**Changes:**
```bash
git status
git diff
git add <file>
git commit -m "message"
git commit -am "message"
```

**Branching:**
```bash
git branch
git branch <name>
git checkout <branch>
git checkout -b <branch>
git merge <branch>
git branch -d <branch>
```

**Remote:**
```bash
git remote -v
git fetch
git pull
git push
git push -u origin <branch>
```

**History:**
```bash
git log
git log --oneline --graph
git show <commit>
git blame <file>
```

**Undo:**
```bash
git reset HEAD <file>
git checkout -- <file>
git revert <commit>
git reset --hard HEAD~1
```

Master these Git commands for effective version control!
