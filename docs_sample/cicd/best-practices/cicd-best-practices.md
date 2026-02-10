# CI/CD Best Practices

## 🎯 Pipeline Design Principles

### **1. Pipeline Design Best Practices**

**Nguyên tắc thiết kế:**
```
┌─────────────────────────────────────────┐
│    PIPELINE DESIGN PRINCIPLES           │
├─────────────────────────────────────────┤
│ 1. Fast Feedback (< 10 minutes)        │
│ 2. Fail Fast                            │
│ 3. Reproducible Builds                  │
│ 4. Idempotent Operations                │
│ 5. Security First                       │
│ 6. Observable & Traceable               │
└─────────────────────────────────────────┘
```

**Optimal Pipeline Structure:**
```yaml
# .gitlab-ci.yml - Example
stages:
  - validate      # < 2 minutes
  - build         # < 5 minutes
  - test          # < 10 minutes (parallel)
  - security      # < 5 minutes (parallel with tests)
  - deploy-dev    # < 3 minutes
  - deploy-prod   # Manual gate

# Fail fast: Put fastest checks first
validate:
  stage: validate
  script:
    - npm run lint          # Code style
    - npm run format:check  # Formatting
    - npm run type-check    # TypeScript

# Build once, deploy many
build:
  stage: build
  script:
    - docker build -t $IMAGE:$CI_COMMIT_SHA .
    - docker push $IMAGE:$CI_COMMIT_SHA
  artifacts:
    paths:
      - build-manifest.json
    expire_in: 7 days

# Parallel tests
test:unit:
  stage: test
  parallel: 4
  script:
    - npm run test:unit -- --shard=$CI_NODE_INDEX/$CI_NODE_TOTAL

test:integration:
  stage: test
  script:
    - docker-compose up -d
    - npm run test:integration

test:e2e:
  stage: test
  script:
    - npm run test:e2e

# Security scanning
security:sast:
  stage: security
  script:
    - npm audit
    - semgrep --config=auto

security:container:
  stage: security
  script:
    - trivy image $IMAGE:$CI_COMMIT_SHA
```

### **2. Version Control Best Practices**

**Branch Strategy (GitFlow):**
```
main (production)           ────●────────●──────●───
                                 │        │      │
release/v2.0               ──────┴●───────┤      │
                                  │       │      │
develop                  ────●────┴───●───┴──────┤
                             │        │          │
feature/login         ───●───┴───●────┘          │
                         │       │               │
hotfix/security                  └───────────────┴●──
```

**Commit Message Convention:**
```bash
# Semantic commits
<type>(<scope>): <subject>

[optional body]

[optional footer]

# Examples:
feat(auth): add OAuth2 login support
fix(api): handle null response from payment service
docs(readme): update installation instructions
perf(db): add index on user_email column
chore(deps): upgrade to Node.js 20

# Types:
# feat     - New feature
# fix      - Bug fix
# docs     - Documentation
# style    - Formatting, missing semi colons, etc
# refactor - Code restructure
# perf     - Performance improvement
# test     - Adding tests
# chore    - Build process or tooling changes
```

**Tagging Strategy:**
```bash
# Semantic Versioning: MAJOR.MINOR.PATCH
git tag -a v1.2.3 -m "Release v1.2.3"

# v1.0.0 - Initial release
# v1.1.0 - New features (backward compatible)
# v1.1.1 - Bug fixes
# v2.0.0 - Breaking changes
```

---

## 🔒 Security Best Practices

### **1. Secret Management**

**❌ NEVER Do This:**
```yaml
# DON'T hardcode secrets!
env:
  DATABASE_PASSWORD: "my_password_123"
  API_KEY: "abc123xyz"
```

**✅ Do This Instead:**
```yaml
# GitHub Actions
jobs:
  deploy:
    env:
      DATABASE_PASSWORD: ${{ secrets.DB_PASSWORD }}
      API_KEY: ${{ secrets.API_KEY }}

# GitLab CI
deploy:
  script:
    - echo $DB_PASSWORD  # Protected variable
    - echo $API_KEY      # Masked in logs
```

**External Secret Management:**
```bash
# 1. HashiCorp Vault
vault kv get -field=password secret/database/prod

# 2. AWS Secrets Manager
aws secretsmanager get-secret-value \
  --secret-id prod/database/password \
  --query SecretString --output text

# 3. Kubernetes Secrets (with External Secrets Operator)
kubectl create secret generic db-credentials \
  --from-literal=password=$(vault kv get -field=password secret/db)
```

### **2. Dependency Scanning**

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  schedule:
    - cron: '0 8 * * 1'  # Weekly Monday 8AM

jobs:
  scan-dependencies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # npm audit
      - name: npm audit
        run: |
          npm audit --audit-level=moderate
          npm audit fix --dry-run
      
      # Snyk
      - name: Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      # OWASP Dependency Check
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'my-app'
          path: '.'
          format: 'HTML'
  
  scan-container:
    runs-on: ubuntu-latest
    steps:
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      
      # Trivy scan
      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      # Upload to GitHub Security tab
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### **3. Least Privilege Access**

```yaml
# GitHub Actions - Explicit permissions
permissions:
  contents: read        # Read repo
  packages: write       # Push to registry
  id-token: write       # OIDC token
  pull-requests: write  # Comment on PRs
  security-events: write # Upload SARIF

jobs:
  build:
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v3
      - name: Build and push
        run: docker build -t ghcr.io/myorg/app .
```

**GitLab CI - Protected Branches:**
```bash
# Settings > Repository > Protected Branches
# main branch:
#   - Allowed to merge: Maintainers
#   - Allowed to push: No one
#   - Deploy keys: Read only

# Settings > CI/CD > Variables
# PROD_DEPLOY_KEY:
#   - Protected: Yes (only for protected branches)
#   - Masked: Yes (hide in logs)
#   - Environment scope: production
```

---

## 🧪 Testing Strategies

### **Testing Pyramid**

```
                  △
                 / \
                /   \
               /  E2E \         10% - UI/Integration tests
              /  Tests \
             ───────────
            /           \
           / Integration \     20% - API/Service tests
          /    Tests      \
         ─────────────────
        /                 \
       /   Unit Tests      \   70% - Function/Component tests
      /_____________________\
```

**Test Configuration:**
```javascript
// jest.config.js
module.exports = {
  projects: [
    {
      displayName: 'unit',
      testMatch: ['**/__tests__/unit/**/*.test.ts'],
      collectCoverageFrom: ['src/**/*.ts'],
      coverageThresholds: {
        global: {
          branches: 80,
          functions: 80,
          lines: 80,
          statements: 80
        }
      }
    },
    {
      displayName: 'integration',
      testMatch: ['**/__tests__/integration/**/*.test.ts'],
      setupFilesAfterEnv: ['<rootDir>/tests/setup-integration.ts']
    }
  ]
};
```

**Pipeline with Testing:**
```yaml
# .github/workflows/test.yml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20]
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
  
  integration-tests:
    needs: unit-tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v3
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
  
  e2e-tests:
    needs: integration-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

---

## 📦 Artifact Management

### **Build Artifacts Best Practices**

**Versioning Strategy:**
```bash
# Semantic versioning for releases
v1.2.3

# Additional tags for Docker images:
# 1. Git SHA (unique, immutable)
myapp:abc123def456789

# 2. Branch name (latest for branch)
myapp:main
myapp:develop

# 3. Semantic version (release)
myapp:1.2.3
myapp:1.2
myapp:1
myapp:latest
```

**Docker Image Tagging:**
```yaml
# .github/workflows/build.yml
jobs:
  build:
    steps:
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=sha,prefix=,format=long
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Artifact Retention:**
```yaml
# GitLab CI
build:
  artifacts:
    paths:
      - dist/
    expire_in: 30 days  # Auto cleanup
    when: on_success    # Only if job succeeds

# GitHub Actions
- uses: actions/upload-artifact@v3
  with:
    name: build-artifacts
    path: dist/
    retention-days: 30
```

---

## 🚀 Deployment Strategies

### **Strategy Comparison**

| Strategy | Downtime | Rollback Speed | Cost | Complexity |
|----------|----------|----------------|------|------------|
| **Recreate** | ❌ Yes | Fast | Low | Low |
| **Rolling** | ✅ No | Medium | Low | Medium |
| **Blue-Green** | ✅ No | Instant | High (2x) | Medium |
| **Canary** | ✅ No | Fast | Medium | High |

### **1. Blue-Green Deployment**

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: main
    path: k8s/overlays/prod
  
  syncPolicy:
    automated:
      prune: false  # Don't auto-delete old version
  
  # Rollout strategy
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false  # Manual promotion
```

**Kubernetes Service:**
```yaml
# myapp-active-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-active
spec:
  selector:
    app: myapp
    version: blue  # Switch between blue/green
  ports:
    - port: 80
      targetPort: 8080

---
# Deployment Blue
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: app
          image: myapp:v1.2.3

---
# Deployment Green
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: app
          image: myapp:v1.2.4  # New version
```

**Deployment Script:**
```bash
#!/bin/bash
# deploy-blue-green.sh

CURRENT=$(kubectl get svc myapp-active -o jsonpath='{.spec.selector.version}')
NEW=$([ "$CURRENT" == "blue" ] && echo "green" || echo "blue")

echo "Current: $CURRENT, Deploying to: $NEW"

# 1. Deploy new version
kubectl apply -f myapp-${NEW}-deployment.yaml
kubectl rollout status deployment/myapp-${NEW}

# 2. Run smoke tests
./smoke-tests.sh myapp-preview

# 3. Switch traffic (manual confirmation)
read -p "Switch traffic to $NEW? (y/n) " -n 1 -r
if [[ $REPLY =~ ^[Yy]$ ]]; then
  kubectl patch svc myapp-active -p "{\"spec\":{\"selector\":{\"version\":\"$NEW\"}}}"
  echo "Traffic switched to $NEW"
else
  echo "Deployment cancelled"
fi
```

### **2. Canary Deployment**

```yaml
# Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10   # 10% traffic
        - pause:
            duration: 5m  # Monitor for 5 minutes
        - setWeight: 25
        - pause:
            duration: 5m
        - setWeight: 50
        - pause:
            duration: 5m
        - setWeight: 75
        - pause:
            duration: 5m
      
      analysis:
        templates:
          - templateName: error-rate
        startingStep: 1
        args:
          - name: service-name
            value: myapp
  
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:v1.2.4
          ports:
            - containerPort: 8080

---
# Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result < 0.05  # < 5% error rate
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              status=~"5.."
            }[1m])) /
            sum(rate(http_requests_total{
              service="{{args.service-name}}"
            }[1m]))
```

---

## 📊 Monitoring & Observability

### **Key Metrics to Track**

**DORA Metrics:**
```yaml
# 1. Deployment Frequency
deployment_frequency:
  target: "Multiple times per day"
  current: "Calculate from CI/CD logs"

# 2. Lead Time for Changes
lead_time:
  target: "< 1 hour"
  measurement: "Commit to production"

# 3. Mean Time to Recovery (MTTR)
mttr:
  target: "< 1 hour"
  measurement: "Incident detection to resolution"

# 4. Change Failure Rate
change_failure_rate:
  target: "< 15%"
  measurement: "Failed deployments / Total deployments"
```

**Pipeline Metrics:**
```python
# Example: Track metrics with Prometheus
from prometheus_client import Counter, Histogram, Gauge
import time

# Counters
pipeline_runs_total = Counter(
    'pipeline_runs_total',
    'Total pipeline runs',
    ['pipeline', 'status']  # Labels
)

pipeline_failures_total = Counter(
    'pipeline_failures_total',
    'Total pipeline failures',
    ['pipeline', 'stage']
)

# Histogram for duration
pipeline_duration_seconds = Histogram(
    'pipeline_duration_seconds',
    'Pipeline execution duration',
    ['pipeline', 'stage']
)

# Gauge for current state
pipelines_running = Gauge(
    'pipelines_running',
    'Currently running pipelines'
)

# Usage
def run_pipeline(name):
    pipelines_running.inc()
    start_time = time.time()
    
    try:
        # Run pipeline
        result = execute_pipeline(name)
        
        duration = time.time() - start_time
        pipeline_duration_seconds.labels(pipeline=name, stage='total').observe(duration)
        pipeline_runs_total.labels(pipeline=name, status='success').inc()
        
    except Exception as e:
        pipeline_runs_total.labels(pipeline=name, status='failure').inc()
        pipeline_failures_total.labels(pipeline=name, stage='unknown').inc()
        raise
    finally:
        pipelines_running.dec()
```

**Grafana Dashboard:**
```json
{
  "dashboard": {
    "title": "CI/CD Metrics",
    "panels": [
      {
        "title": "Deployment Frequency",
        "targets": [
          {
            "expr": "sum(rate(pipeline_runs_total{status='success'}[24h]))"
          }
        ]
      },
      {
        "title": "Pipeline Success Rate",
        "targets": [
          {
            "expr": "sum(rate(pipeline_runs_total{status='success'}[1h])) / sum(rate(pipeline_runs_total[1h])) * 100"
          }
        ]
      },
      {
        "title": "Average Pipeline Duration",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, pipeline_duration_seconds_bucket)"
          }
        ]
      }
    ]
  }
}
```

---

## 💰 Cost Optimization

### **1. Build Cache Optimization**

**Docker Multi-stage Build:**
```dockerfile
# ❌ Bad: Rebuilds everything on code change
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

# ✅ Good: Cached layers
FROM node:18 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS runner
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**GitHub Actions Cache:**
```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v3
      
      # Cache node_modules
      - uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      # Cache Docker layers
      - uses: docker/setup-buildx-action@v2
      
      - uses: docker/build-push-action@v4
        with:
          context: .
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### **2. Runner Optimization**

**Self-hosted Runners:**
```yaml
# Cost comparison
github_hosted:
  cost: "$0.008/minute for Linux"
  wait_time: "Variable (queue)"
  
self_hosted:
  cost: "EC2 t3.medium = $0.0416/hour"
  wait_time: "Instant (dedicated)"
  
# Break-even: ~300 minutes/month
# Recommendation: Use self-hosted for high-volume projects
```

**Spot Instances for CI:**
```bash
# AWS EC2 Spot (70% cost savings)
aws ec2 run-instances \
  --instance-type t3.medium \
  --image-id ami-0c55b159cbfafe1f0 \
  --spot-instance-requests \
  --user-data file://runner-setup.sh

# Kubernetes with Karpenter (auto-scaling spot nodes)
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: ci-runners
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["t3.medium", "t3a.medium"]
  limits:
    resources:
      cpu: 100
  ttlSecondsAfterEmpty: 30
```

### **3. Parallel Execution**

```yaml
# Serial execution (slow)
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint  # 2 min
  
  test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - run: npm test      # 10 min
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: npm run build  # 5 min
  
# Total: 17 minutes

# Parallel execution (fast)
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint  # 2 min
  
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npm test -- --shard=${{ matrix.shard }}/4  # 2.5 min
  
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build  # 5 min

# Total: 5 minutes (max of parallel jobs)
```

---

## 👥 Team Workflows

### **Code Review Best Practices**

**Pre-commit Hooks:**
```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npm run lint
npm run format
npm run type-check
npm run test:affected
```

**PR Template:**
```markdown
<!-- .github/pull_request_template.md -->
## Description
<!-- What changes does this PR introduce? -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Checklist
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No new warnings
- [ ] Passes all CI checks
- [ ] Reviewed by at least one team member

## Screenshots (if applicable)

## Related Issues
Closes #123
```

**CODEOWNERS:**
```bash
# .github/CODEOWNERS
# Auto-assign reviewers

# Global owners
* @team-leads

# Backend
/backend/ @backend-team
/backend/auth/ @security-team

# Frontend
/frontend/ @frontend-team

# Infrastructure
/k8s/ @devops-team
/.github/workflows/ @devops-team
/terraform/ @devops-team

# Docs
/docs/ @tech-writers
README.md @tech-writers
```

---

## 🎓 Summary

**✅ Key Takeaways:**

1. **Fast Feedback**: Keep pipelines < 10 minutes
2. **Security First**: Scan dependencies, secrets, containers
3. **Test Pyramid**: 70% unit, 20% integration, 10% E2E
4. **Deployment Strategies**: Choose based on requirements
5. **Monitor Everything**: DORA metrics, pipeline metrics
6. **Cost Optimization**: Cache, parallel execution, spot instances
7. **Team Practices**: PR templates, code review, CODEOWNERS

**📚 Recommended Reading:**
- [12-Factor App](https://12factor.net/)
- [State of DevOps Report](https://dora.dev/)
- [GitLab CI/CD Best Practices](https://docs.gitlab.com/ee/ci/best_practices.html)
- [GitHub Actions Best Practices](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

**🔗 Related Sections:**
- [CI/CD Fundamentals ←](../fundamentals/cicd-basics.md)
- [Jenkins Pipeline](../jenkins/jenkins-pipeline.md)
- [GitLab CI/CD](../gitlab/gitlab-ci.md)
- [GitHub Actions](../github-actions/github-actions.md)
- [ArgoCD & GitOps](../argocd/argocd-gitops.md)
