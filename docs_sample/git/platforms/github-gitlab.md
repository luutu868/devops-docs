# GitHub và GitLab Platforms

## GitHub

### Repository Management

**Tạo Repository:**
```bash
# Via web interface
# 1. Click "New repository"
# 2. Enter name, description
# 3. Choose public/private
# 4. Add README, .gitignore, license

# Via CLI (GitHub CLI)
gh repo create my-project --public
gh repo create my-project --private
gh repo create my-project --public --clone
```

**Clone Repository:**
```bash
# HTTPS
git clone https://github.com/username/repo.git

# SSH (recommended)
git clone git@github.com:username/repo.git

# GitHub CLI
gh repo clone username/repo
```

**Repository Settings:**
- **General**: Description, features, danger zone
- **Collaborators**: Add team members
- **Branches**: Branch protection rules
- **Webhooks**: Integration events
- **Secrets**: CI/CD secrets
- **Actions**: Workflow permissions

### Issues

**Create Issue:**
```bash
# Via web interface
# Click "New issue"

# Via GitHub CLI
gh issue create --title "Bug in login" --body "Description"
gh issue create --title "Feature request" --label "enhancement"
```

**Issue Templates:**
Create `.github/ISSUE_TEMPLATE/bug_report.md`:
```markdown
---
name: Bug Report
about: Report a bug
title: '[BUG] '
labels: bug
assignees: ''
---

**Describe the bug**
A clear description of what the bug is.

**To Reproduce**
Steps to reproduce:
1. Go to '...'
2. Click on '...'
3. See error

**Expected behavior**
What you expected to happen.

**Screenshots**
If applicable, add screenshots.

**Environment:**
- OS: [e.g. Ubuntu 20.04]
- Browser: [e.g. Chrome 90]
- Version: [e.g. 1.0.0]
```

**Issue Commands:**
```bash
# List issues
gh issue list
gh issue list --state open
gh issue list --label bug
gh issue list --assignee @me

# View issue
gh issue view 123

# Close issue
gh issue close 123

# Reopen issue
gh issue reopen 123

# Comment on issue
gh issue comment 123 --body "This is fixed"
```

**Keywords in Commits:**
```bash
# Close issue
git commit -m "Fix login bug

Fixes #123
Closes #456"

# Reference issue
git commit -m "Update authentication

Related to #789"
```

### Pull Requests

**Create PR:**
```bash
# Push branch
git push origin feature/my-feature

# Create PR via web
# Click "Compare & pull request"

# Via CLI
gh pr create --title "Add new feature" --body "Description"
gh pr create --title "Fix bug" --base main --head feature/bug-fix

# Create draft PR
gh pr create --draft
```

**PR Commands:**
```bash
# List PRs
gh pr list
gh pr list --state open
gh pr list --author username

# View PR
gh pr view 456
gh pr view 456 --web

# Checkout PR locally
gh pr checkout 456
git fetch origin pull/456/head:pr-456
git checkout pr-456

# Review PR
gh pr review 456 --approve
gh pr review 456 --request-changes
gh pr review 456 --comment --body "Looks good"

# Merge PR
gh pr merge 456
gh pr merge 456 --squash
gh pr merge 456 --rebase
gh pr merge 456 --merge

# Close PR
gh pr close 456

# Reopen PR
gh pr reopen 456
```

**PR Template:**
Create `.github/PULL_REQUEST_TEMPLATE.md`:
```markdown
## Description
Brief summary of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Changes Made
- Item 1
- Item 2
- Item 3

## How to Test
1. Step 1
2. Step 2
3. Expected result

## Checklist
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] Code follows style guidelines
- [ ] All tests pass
- [ ] No new warnings

## Related Issues
Fixes #123
Related to #456

## Screenshots
[If applicable]
```

### Branch Protection

**Settings → Branches → Add rule:**

```yaml
Branch name pattern: main

Protection rules:
☑ Require pull request before merging
  ☑ Require approvals: 2
  ☑ Dismiss stale reviews
  ☑ Require review from Code Owners

☑ Require status checks before merging
  ☑ Require branches to be up to date
  Status checks:
    - build
    - test
    - lint

☑ Require conversation resolution

☑ Require signed commits

☑ Include administrators

☑ Restrict who can push
  Teams/Users: [Select teams]

☑ Allow force pushes: Nobody

☑ Allow deletions: No
```

### GitHub Actions

**Basic Workflow** - `.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm test
    
    - name: Build
      run: npm run build
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: dist
        path: dist/
```

**Matrix Build:**
```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    
    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js ${{ matrix.node }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node }}
    - run: npm ci
    - run: npm test
```

**Deploy Workflow:**
```yaml
name: Deploy

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install and build
      run: |
        npm ci
        npm run build
    
    - name: Deploy to server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /var/www/app
          git pull origin main
          npm ci --production
          pm2 restart app
```

### GitHub Packages

**Publish npm package:**
```yaml
name: Publish Package

on:
  release:
    types: [created]

jobs:
  publish:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - uses: actions/setup-node@v3
      with:
        node-version: '18'
        registry-url: 'https://npm.pkg.github.com'
    
    - run: npm ci
    - run: npm publish
      env:
        NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**package.json:**
```json
{
  "name": "@username/package-name",
  "version": "1.0.0",
  "repository": {
    "type": "git",
    "url": "git://github.com/username/repo.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

### GitHub Pages

**Deploy static site:**
```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Build
      run: |
        npm ci
        npm run build
    
    - name: Deploy
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./dist
```

Access at: `https://username.github.io/repo/`

### Secrets Management

**Add secrets:**
Settings → Secrets and variables → Actions → New repository secret

**Use in workflows:**
```yaml
steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.API_KEY }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    run: ./deploy.sh
```

## GitLab

### Repository Management

**Create Project:**
```bash
# Via web interface
# Click "New project"

# Via CLI
gitlab project create --name my-project
```

**Clone Repository:**
```bash
# HTTPS
git clone https://gitlab.com/username/project.git

# SSH
git clone git@gitlab.com:username/project.git
```

### Issues

**Issue Boards:**
- Kanban-style boards
- Multiple boards per project
- Customizable lists
- Drag and drop

**Issue Weights:**
```markdown
/weight 3
```

**Time Tracking:**
```markdown
/estimate 2h
/spend 1h
```

**Quick Actions:**
```markdown
/assign @username
/label ~bug ~priority::high
/milestone %v1.0
/due 2024-01-15
/close
```

### Merge Requests

**Create MR:**
```bash
# Push and create MR
git push -o merge_request.create origin feature/my-feature

# Push with options
git push -o merge_request.create \
        -o merge_request.title="Add feature" \
        -o merge_request.target=main \
        -o merge_request.label="feature" \
        origin feature/my-feature
```

**MR Approvals:**
Settings → Merge requests → Approval rules
```yaml
Approvals required: 2
Approvers: @team-lead, @senior-dev
Prevent approval by author: Yes
Require approval from code owners: Yes
```

**MR Pipelines:**
- Automatic pipeline trigger
- Pipeline status in MR
- Deploy to review apps
- Merge when pipeline succeeds

### GitLab CI/CD

**Basic Pipeline** - `.gitlab-ci.yml`:
```yaml
stages:
  - build
  - test
  - deploy

variables:
  NODE_VERSION: "18"

before_script:
  - node --version
  - npm --version

# Build job
build:
  stage: build
  image: node:18
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
  only:
    - main
    - develop

# Test job
test:
  stage: test
  image: node:18
  script:
    - npm ci
    - npm run lint
    - npm test
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# Deploy to staging
deploy_staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
  script:
    - ssh user@staging.example.com "
        cd /var/www/app &&
        git pull origin develop &&
        npm ci --production &&
        pm2 restart app
      "
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

# Deploy to production
deploy_production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
  script:
    - ssh user@production.example.com "
        cd /var/www/app &&
        git pull origin main &&
        npm ci --production &&
        pm2 restart app
      "
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

**Docker Build:**
```yaml
docker_build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

**Include templates:**
```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Code-Quality.gitlab-ci.yml

stages:
  - test
  - build
  - deploy
```

### Container Registry

**Build and push:**
```yaml
build_image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
  only:
    - tags
```

**Pull image:**
```bash
docker login registry.gitlab.com
docker pull registry.gitlab.com/username/project:latest
```

### Environments

**Configure environment:**
```yaml
deploy_staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop_staging

stop_staging:
  stage: deploy
  script:
    - ./stop.sh staging
  environment:
    name: staging
    action: stop
  when: manual
```

**View environments:**
Deployments → Environments

### GitLab Runner

**Install Runner:**
```bash
# Linux
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner

# Register runner
sudo gitlab-runner register
# Enter GitLab URL
# Enter registration token (from Settings → CI/CD → Runners)
# Enter description
# Enter tags
# Enter executor (shell, docker, kubernetes)
```

**Runner config** - `/etc/gitlab-runner/config.toml`:
```toml
concurrent = 4

[[runners]]
  name = "my-runner"
  url = "https://gitlab.com"
  token = "xxx"
  executor = "docker"
  [runners.docker]
    image = "node:18"
    privileged = false
    volumes = ["/cache"]
```

**Runner commands:**
```bash
# Start runner
sudo gitlab-runner start

# Stop runner
sudo gitlab-runner stop

# Status
sudo gitlab-runner status

# Run specific job
sudo gitlab-runner exec docker test
```

### Auto DevOps

Enable auto DevOps:
Settings → CI/CD → Auto DevOps

**Features:**
- Auto Build (Docker image)
- Auto Test (unit, integration)
- Auto Code Quality
- Auto SAST (Security testing)
- Auto Dependency Scanning
- Auto Container Scanning
- Auto Review Apps
- Auto DAST (Dynamic security)
- Auto Deploy (Kubernetes)
- Auto Monitoring

### GitLab Pages

**.gitlab-ci.yml:**
```yaml
pages:
  stage: deploy
  image: node:18
  script:
    - npm ci
    - npm run build
    - mv dist public
  artifacts:
    paths:
      - public
  only:
    - main
```

Access at: `https://username.gitlab.io/project/`

## SSH Keys

### Generate SSH Key

```bash
# Generate key
ssh-keygen -t ed25519 -C "your_email@example.com"

# For older systems
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Save to default location
# Enter passphrase (recommended)

# Copy public key
cat ~/.ssh/id_ed25519.pub
```

### Add to GitHub

1. Settings → SSH and GPG keys
2. New SSH key
3. Paste public key
4. Save

**Test connection:**
```bash
ssh -T git@github.com
```

### Add to GitLab

1. Preferences → SSH Keys
2. Paste public key
3. Set expiration date
4. Save

**Test connection:**
```bash
ssh -T git@gitlab.com
```

## Personal Access Tokens

### GitHub Token

1. Settings → Developer settings → Personal access tokens
2. Generate new token
3. Select scopes:
   - `repo`: Full repository access
   - `workflow`: GitHub Actions
   - `admin:org`: Organization access
4. Generate token
5. Copy token (shown once!)

**Use token:**
```bash
# HTTPS clone with token
git clone https://TOKEN@github.com/username/repo.git

# Or use credential helper
git config --global credential.helper store
git clone https://github.com/username/repo.git
# Enter username and token as password
```

### GitLab Token

1. Preferences → Access Tokens
2. Add new token
3. Select scopes:
   - `api`: API access
   - `read_repository`: Read repo
   - `write_repository`: Write repo
   - `read_registry`: Read container registry
4. Create token

**Use token:**
```bash
# Clone with token
git clone https://oauth2:TOKEN@gitlab.com/username/project.git

# Or
git clone https://gitlab-ci-token:TOKEN@gitlab.com/username/project.git
```

## Webhooks

### GitHub Webhook

Settings → Webhooks → Add webhook

**Payload URL:** `https://your-server.com/webhook`
**Content type:** `application/json`
**Events:**
- Push
- Pull request
- Issues
- Release

**Payload example:**
```json
{
  "ref": "refs/heads/main",
  "before": "abc123...",
  "after": "def456...",
  "repository": {
    "name": "repo",
    "full_name": "username/repo"
  },
  "pusher": {
    "name": "username",
    "email": "user@example.com"
  },
  "commits": [...]
}
```

### GitLab Webhook

Settings → Webhooks → Add webhook

**URL:** `https://your-server.com/webhook`
**Trigger:**
- Push events
- Tag events
- Merge request events
- Pipeline events

## Comparison: GitHub vs GitLab

| Feature | GitHub | GitLab |
|---------|--------|--------|
| **Hosting** | Cloud only | Cloud + Self-hosted |
| **CI/CD** | GitHub Actions | GitLab CI/CD (built-in) |
| **Container Registry** | GitHub Packages | Built-in |
| **Issue Boards** | Projects (beta) | Built-in Kanban |
| **Code Review** | Pull Requests | Merge Requests |
| **Wiki** | Basic | Advanced |
| **Security Scanning** | Limited (paid) | Comprehensive |
| **Free Private Repos** | Unlimited | Unlimited |
| **Free CI/CD Minutes** | 2000/month | 400/month |
| **Self-hosted** | Enterprise only | Free (CE) |
| **Auto DevOps** | No | Yes |
| **Environments** | Basic | Advanced |
| **Monitoring** | Limited | Built-in |

**Choose GitHub if:**
- Need large community
- Want simplicity
- Use GitHub Actions
- Prefer cloud-only

**Choose GitLab if:**
- Need complete DevOps platform
- Want self-hosting option
- Require advanced CI/CD
- Need built-in security scanning

## Migration

### GitHub to GitLab

```bash
# 1. Create new GitLab project

# 2. Add GitLab remote
git remote add gitlab git@gitlab.com:username/project.git

# 3. Push all branches
git push gitlab --all

# 4. Push all tags
git push gitlab --tags

# 5. Import issues (use GitLab import tool)
# Project → Settings → General → Import project
# Select GitHub

# 6. Update CI/CD
# Convert .github/workflows to .gitlab-ci.yml
```

### GitLab to GitHub

```bash
# 1. Create new GitHub repository

# 2. Add GitHub remote
git remote add github git@github.com:username/repo.git

# 3. Push all branches
git push github --all

# 4. Push all tags
git push github --tags

# 5. Import issues manually or use tools
# GitHub doesn't provide direct import from GitLab
```

## Best Practices

1. **Security:**
   - Enable 2FA
   - Use SSH keys
   - Rotate access tokens
   - Review webhooks regularly

2. **Branch Protection:**
   - Require reviews
   - Require status checks
   - Prevent force push
   - Protect from deletion

3. **CI/CD:**
   - Use caching
   - Parallelize jobs
   - Store secrets securely
   - Monitor pipeline performance

4. **Collaboration:**
   - Use templates (PR, Issue)
   - Clear contribution guidelines
   - Code of conduct
   - License file

5. **Organization:**
   - Use organizations/groups
   - Team-based permissions
   - Consistent naming
   - Regular cleanup

Master GitHub and GitLab for effective collaboration!
