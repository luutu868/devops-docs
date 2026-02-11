# Git Workflow và Best Practices

## Git Workflows

### Gitflow Workflow

Workflow phổ biến nhất cho projects lớn với release schedule rõ ràng.

**Branch structure:**
```
main (production)
  └── develop (integration)
      ├── feature/user-auth
      ├── feature/payment
      └── release/v1.0
      └── hotfix/critical-bug
```

**Branches:**
- `main`: Production-ready code
- `develop`: Integration branch for features
- `feature/*`: Feature development
- `release/*`: Release preparation
- `hotfix/*`: Production fixes

**Workflow:**

```bash
# 1. Start feature
git checkout develop
git pull origin develop
git checkout -b feature/user-login

# 2. Work on feature
git add .
git commit -m "Add login form"
git push -u origin feature/user-login

# 3. Merge to develop
git checkout develop
git pull origin develop
git merge --no-ff feature/user-login
git push origin develop
git branch -d feature/user-login

# 4. Create release
git checkout develop
git checkout -b release/v1.0

# Fix bugs in release
git commit -m "Fix bug in release"

# 5. Merge to main
git checkout main
git merge --no-ff release/v1.0
git tag -a v1.0.0 -m "Version 1.0.0"
git push origin main --tags

# 6. Merge back to develop
git checkout develop
git merge --no-ff release/v1.0
git push origin develop
git branch -d release/v1.0

# 7. Hotfix if needed
git checkout main
git checkout -b hotfix/critical-fix
git commit -m "Fix critical bug"
git checkout main
git merge --no-ff hotfix/critical-fix
git tag -a v1.0.1 -m "Hotfix 1.0.1"
git checkout develop
git merge --no-ff hotfix/critical-fix
git push origin main develop --tags
git branch -d hotfix/critical-fix
```

**Pros:**
- Rõ ràng, có cấu trúc
- Phù hợp với scheduled releases
- Dễ quản lý nhiều versions

**Cons:**
- Phức tạp cho small teams
- Nhiều branches cần maintain
- Slow feedback loop

### GitHub Flow

Workflow đơn giản hơn, phù hợp với continuous deployment.

**Branch structure:**
```
main (production)
  ├── feature/add-dashboard
  ├── feature/fix-login
  └── feature/update-api
```

**Workflow:**

```bash
# 1. Create feature branch from main
git checkout main
git pull origin main
git checkout -b feature/add-dashboard

# 2. Make changes
git add .
git commit -m "Add dashboard component"
git push -u origin feature/add-dashboard

# 3. Create Pull Request on GitHub

# 4. Code review, discussion, tests

# 5. Deploy to staging/preview (optional)

# 6. Merge to main (after approval)
git checkout main
git pull origin main
git merge feature/add-dashboard
git push origin main

# 7. Deploy to production (automatic)

# 8. Delete feature branch
git branch -d feature/add-dashboard
git push origin --delete feature/add-dashboard
```

**Rules:**
- `main` is always deployable
- Create descriptive branch names
- Open PR early for feedback
- Deploy after merging to main
- Delete branches after merge

**Pros:**
- Simple và dễ hiểu
- Fast deployment cycle
- Phù hợp với CI/CD

**Cons:**
- Khó quản lý multiple versions
- Không phù hợp với scheduled releases

### GitLab Flow

Kết hợp ưu điểm của Gitflow và GitHub Flow.

**Branch structure:**
```
main (development)
  ├── feature/new-feature
  └── production (deployed)
      └── staging (pre-production)
```

**Workflow với environment branches:**

```bash
# 1. Feature development
git checkout main
git checkout -b feature/new-api

# 2. Merge to main after review
git checkout main
git merge feature/new-api
git push origin main

# 3. Deploy to staging
git checkout staging
git merge main
git push origin staging

# 4. Deploy to production
git checkout production
git merge staging
git push origin production
git tag v1.0.0
```

**Workflow với release branches:**

```bash
# 1. Create release branch
git checkout -b release/v1.0 main

# 2. Bugfixes in release
git commit -m "Fix bug for release"

# 3. Cherry-pick fixes to main
git checkout main
git cherry-pick <commit-sha>

# 4. Deploy release
git checkout release/v1.0
git tag v1.0.0
```

**Pros:**
- Linh hoạt
- Phù hợp với staged deployments
- Dễ quản lý versions

**Cons:**
- Phức tạp hơn GitHub Flow
- Cần CI/CD tốt

### Trunk-Based Development

Mọi người commit trực tiếp vào `main` hoặc short-lived branches.

**Branch structure:**
```
main
  ├── feature/small-fix (< 1 day)
  └── feature/tiny-change (< 1 day)
```

**Workflow:**

```bash
# 1. Small feature (< 1 day)
git checkout main
git pull origin main
git checkout -b feature/quick-fix

# 2. Work and commit frequently
git add .
git commit -m "Initial implementation"
git push -u origin feature/quick-fix

# 3. Merge quickly (same day)
git checkout main
git merge feature/quick-fix
git push origin main
git branch -d feature/quick-fix

# 4. Or commit directly to main (with feature flags)
git checkout main
git add .
git commit -m "Add feature behind flag"
git push origin main
```

**Key practices:**
- Feature flags/toggles
- Frequent integration
- Automated testing
- Quick reviews

**Pros:**
- Simplest workflow
- Fastest integration
- Least merge conflicts

**Cons:**
- Requires excellent testing
- Needs feature flags
- High discipline required

## Commit Message Conventions

### Conventional Commits

Format: `<type>(<scope>): <subject>`

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Formatting, missing semi colons, etc
- `refactor`: Code restructuring
- `perf`: Performance improvement
- `test`: Adding tests
- `chore`: Maintenance tasks
- `ci`: CI/CD changes
- `build`: Build system changes
- `revert`: Reverting changes

**Examples:**

```bash
# Feature
git commit -m "feat(auth): add JWT authentication"
git commit -m "feat(api): add user profile endpoint"

# Bug fix
git commit -m "fix(login): resolve password validation error"
git commit -m "fix(payment): handle null card number"

# Breaking change
git commit -m "feat(api): change user endpoint structure

BREAKING CHANGE: user endpoint now returns array instead of object"

# Scope examples
git commit -m "feat(frontend): add dashboard component"
git commit -m "fix(backend): resolve database connection issue"
git commit -m "docs(readme): update installation instructions"

# With body
git commit -m "feat(api): add user search endpoint

- Add search by name
- Add search by email
- Add pagination support"
```

### Good Commit Messages

**Do:**
```bash
# Clear, concise subject
git commit -m "Add user authentication with JWT"

# Imperative mood
git commit -m "Fix login validation bug"
git commit -m "Update dependencies to latest versions"

# Detailed body when needed
git commit -m "Refactor database connection pool

- Extract connection logic to separate module
- Add connection retry mechanism
- Improve error handling
- Add connection pool monitoring"

# Reference issues
git commit -m "Fix login timeout issue

Fixes #123
Related to #456"
```

**Don't:**
```bash
# Too vague
git commit -m "Fix bug"
git commit -m "Update code"
git commit -m "Changes"

# Past tense
git commit -m "Fixed the login bug"
git commit -m "Added new feature"

# Not descriptive
git commit -m "WIP"
git commit -m "asdf"
git commit -m "oops"
```

### Commit Message Template

Create template file `.gitmessage`:
```
<type>(<scope>): <subject>

<body>

<footer>

# Type: feat, fix, docs, style, refactor, test, chore
# Scope: Component or module affected
# Subject: Brief description (imperative mood)
# Body: Detailed explanation (optional)
# Footer: Issue references, breaking changes
```

Set template:
```bash
git config --global commit.template ~/.gitmessage
```

## Branch Naming Strategies

### Common Patterns

```bash
# Feature branches
feature/user-authentication
feature/add-payment-gateway
feature/JIRA-123-user-profile

# Bug fixes
fix/login-error
fix/null-pointer-exception
bugfix/JIRA-456-payment-error

# Hotfixes
hotfix/critical-security-patch
hotfix/production-crash

# Release branches
release/v1.0.0
release/2023-12-01

# Experimental
experimental/new-architecture
spike/performance-test

# Personal branches
username/feature-name
john/test-new-approach
```

### Best Practices

```bash
# Use separators
feature/user-login        # Good
feature-user-login        # OK
feature_user_login        # Avoid

# Be descriptive
git checkout -b feature/jwt-authentication  # Good
git checkout -b feature/auth                # OK
git checkout -b feature/f1                  # Bad

# Include ticket number
git checkout -b feature/JIRA-123-add-dashboard
git checkout -b fix/GH-456-login-bug

# Use lowercase with hyphens
feature/add-user-profile  # Good
feature/Add_User_Profile  # Avoid
```

## Pull Request / Merge Request Best Practices

### Creating Good PRs

**PR Title:**
```
[Type] Brief description

Examples:
[Feature] Add user authentication
[Fix] Resolve login validation error
[Refactor] Improve database query performance
```

**PR Description Template:**
```markdown
## Description
Brief summary of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Changes Made
- Added JWT authentication
- Implemented login endpoint
- Added password hashing

## Testing
- [ ] Unit tests added
- [ ] Integration tests added
- [ ] Manual testing completed

## Screenshots (if applicable)
[Add screenshots here]

## Related Issues
Fixes #123
Related to #456

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex code
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] Tests pass locally
```

### PR Size

```bash
# Small PR (Recommended)
# 1-200 lines changed
# Easy to review
# Fast feedback

# Medium PR
# 200-500 lines
# Reasonable review time

# Large PR (Avoid)
# 500+ lines
# Hard to review
# Slow feedback
# Consider splitting
```

**Split large PRs:**
```bash
# Instead of one large PR
git checkout -b feature/complete-user-system

# Split into smaller PRs
git checkout -b feature/user-model
git checkout -b feature/user-api
git checkout -b feature/user-frontend
git checkout -b feature/user-tests
```

### Code Review Process

**As Author:**
```bash
# 1. Self-review first
git diff main...feature/my-feature

# 2. Test thoroughly
npm test
npm run lint

# 3. Update documentation
# Update README, add comments

# 4. Create PR with good description

# 5. Respond to feedback promptly

# 6. Make requested changes
git add .
git commit -m "Address review comments"
git push

# 7. Request re-review
```

**As Reviewer:**
```markdown
# Good feedback:
"Consider using Array.map() here for better readability"
"This function could be simplified by extracting the validation logic"
"Great implementation! Just a minor suggestion: add error handling"

# Avoid:
"This is wrong"
"Bad code"
"Why did you do this?"
```

## Resolving Merge Conflicts

### Merge Conflicts Workflow

```bash
# 1. Update your branch
git checkout feature/my-feature
git fetch origin
git merge origin/main

# Or rebase
git rebase origin/main

# 2. Conflict appears
Auto-merging file.txt
CONFLICT (content): Merge conflict in file.txt

# 3. Check conflicts
git status

# 4. Open conflicted file
# <<<<<<< HEAD (your changes)
# Your code
# =======
# Their code
# >>>>>>> main (incoming changes)

# 5. Resolve conflict
# Edit file, remove markers, keep desired code

# 6. Mark as resolved
git add file.txt

# 7. Continue merge/rebase
git merge --continue
# or
git rebase --continue

# 8. Push changes
git push origin feature/my-feature
```

### Conflict Resolution Tools

```bash
# Use merge tool
git mergetool

# Configure merge tool
git config --global merge.tool vimdiff
git config --global merge.tool vscode

# Accept ours or theirs
git checkout --ours file.txt
git checkout --theirs file.txt
git add file.txt
```

### Avoiding Conflicts

```bash
# 1. Pull frequently
git pull origin main

# 2. Keep branches short-lived
# Merge within days, not weeks

# 3. Rebase before creating PR
git fetch origin
git rebase origin/main

# 4. Communicate with team
# Coordinate on overlapping files

# 5. Small, focused commits
# Easier to identify conflict source
```

## Git Hooks

### Client-Side Hooks

**pre-commit** - Run before commit:
```bash
#!/bin/bash
# .git/hooks/pre-commit

# Run linter
npm run lint
if [ $? -ne 0 ]; then
    echo "Linting failed. Commit aborted."
    exit 1
fi

# Run tests
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi
```

**commit-msg** - Validate commit message:
```bash
#!/bin/bash
# .git/hooks/commit-msg

commit_msg=$(cat "$1")

# Check conventional commit format
if ! echo "$commit_msg" | grep -qE "^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .+"; then
    echo "Invalid commit message format."
    echo "Use: <type>(<scope>): <subject>"
    exit 1
fi
```

**pre-push** - Run before push:
```bash
#!/bin/bash
# .git/hooks/pre-push

# Run full test suite
npm run test:all
if [ $? -ne 0 ]; then
    echo "Tests failed. Push aborted."
    exit 1
fi
```

### Using Husky

```bash
# Install husky
npm install --save-dev husky

# Initialize husky
npx husky install

# Add pre-commit hook
npx husky add .husky/pre-commit "npm run lint"
npx husky add .husky/pre-commit "npm test"

# Add commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

**package.json:**
```json
{
  "scripts": {
    "prepare": "husky install"
  },
  "devDependencies": {
    "husky": "^8.0.0",
    "@commitlint/cli": "^17.0.0",
    "@commitlint/config-conventional": "^17.0.0"
  }
}
```

## Rebase vs Merge Strategy

### When to Merge

```bash
# Public branches
git checkout main
git merge feature/login

# Preserve complete history
git merge --no-ff feature/login

# Multiple collaborators on branch
git merge origin/feature/shared
```

**Pros of Merge:**
- Complete history preservation
- Shows when features were integrated
- Safe for public branches

**Cons of Merge:**
- Cluttered history with merge commits
- Non-linear history

### When to Rebase

```bash
# Private/local branches
git checkout feature/my-work
git rebase main

# Clean up local commits
git rebase -i HEAD~5

# Update feature branch
git fetch origin
git rebase origin/main
```

**Pros of Rebase:**
- Clean, linear history
- Easier to follow
- No merge commits

**Cons of Rebase:**
- Rewrites history (dangerous for shared branches)
- Harder to recover from mistakes

### Golden Rule

**Never rebase public/shared branches!**

```bash
# DON'T
git checkout main
git rebase feature  # BAD!

# DO
git checkout feature
git rebase main  # GOOD

# Then merge with fast-forward
git checkout main
git merge feature  # Fast-forward merge
```

## Monorepo vs Multi-Repo

### Monorepo

Single repository containing multiple projects.

**Structure:**
```
monorepo/
├── packages/
│   ├── frontend/
│   ├── backend/
│   └── shared/
├── package.json
└── nx.json
```

**Pros:**
- Easy code sharing
- Atomic commits across projects
- Simplified dependency management
- Single CI/CD pipeline

**Cons:**
- Large repository size
- Slower operations
- Requires specialized tools (Nx, Lerna)

**Tools:**
- Nx
- Lerna
- Turborepo
- Rush

### Multi-Repo

Separate repository for each project.

**Structure:**
```
frontend-repo/
backend-repo/
shared-lib-repo/
```

**Pros:**
- Clear boundaries
- Smaller, faster repositories
- Independent versioning
- Team autonomy

**Cons:**
- Harder code sharing
- Complex cross-repo changes
- Multiple CI/CD pipelines

## Git Best Practices

### Commit Practices

1. **Commit often:** Small, logical commits
2. **Atomic commits:** One logical change per commit
3. **Test before commit:** Ensure code works
4. **Good messages:** Clear, descriptive
5. **Don't commit:** Generated files, secrets, large binaries

### Branch Practices

1. **Short-lived branches:** Days, not weeks
2. **Descriptive names:** feature/add-user-auth
3. **Delete merged branches:** Keep repository clean
4. **Protect main:** Require reviews, tests
5. **Update regularly:** Sync with main/develop

### Collaboration Practices

1. **Pull before push:** Avoid conflicts
2. **Review code thoroughly:** Quality over speed
3. **Communicate:** Discuss complex changes
4. **Document:** Update README, add comments
5. **Be respectful:** Constructive feedback

### Security Practices

```bash
# Never commit secrets
git rm --cached .env
echo ".env" >> .gitignore

# Use git-secrets
git secrets --scan
git secrets --install

# Review before push
git diff origin/main..HEAD

# Sign commits (GPG)
git config --global user.signingkey <key-id>
git config --global commit.gpgsign true
git commit -S -m "Signed commit"
```

### Performance Practices

```bash
# Shallow clone for CI
git clone --depth 1 <repo>

# Sparse checkout
git sparse-checkout init
git sparse-checkout set <dir1> <dir2>

# Git LFS for large files
git lfs install
git lfs track "*.psd"

# Periodic maintenance
git gc
git prune
```

## Tổng kết

**Choose workflow based on:**
- Team size
- Release frequency
- Project complexity
- Deployment process

**Key principles:**
- Clear commit messages
- Small, focused changes
- Frequent integration
- Good code reviews
- Protect production branches

Master Git workflows for efficient collaboration!
