# GitHub Actions

##  GitHub Actions Overview

**GitHub Actions** là built-in CI/CD platform của GitHub.

**Key Features:**
- Integrated with GitHub (no setup needed)
- YAML-based workflows
- 20,000+ marketplace actions
- Matrix builds
- Self-hosted runners
- GitHub-hosted runners (free tier)

**Free Tier:**
- Public repos: Unlimited
- Private repos: 2,000 minutes/month

**Architecture:**
```
┌──────────────────────────────────────────┐
│         GitHub Repository                │
│  .github/workflows/*.yml                 │
└────────────┬─────────────────────────────┘
             ↓
┌──────────────────────────────────────────┐
│      GitHub-hosted Runners               │
│  - Ubuntu, Windows, macOS                │
│  Or                                      │
│      Self-hosted Runners                 │
└──────────────────────────────────────────┘
```

## Workflow Basics

### **Simple Workflow**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Test
        run: npm test
```

### **Workflow Structure**

```yaml
name: Workflow Name         # Display name

on:                         # Trigger events
  push:
  pull_request:
  schedule:
  workflow_dispatch:

env:                        # Global environment variables
  NODE_ENV: production

jobs:                       # Jobs to run
  job1:
    runs-on: ubuntu-latest
    steps:
      - uses: action@v1     # Use action
      - run: command        # Run command
  
  job2:
    needs: job1             # Wait for job1
    runs-on: ubuntu-latest
    steps:
      - run: command
```

---

**📘 Xem thêm nội dung chi tiết trong file gốc đã tạo!**

**Tiếp theo**: [CI/CD Best Practices →](../best-practices/cicd-best-practices.md)
