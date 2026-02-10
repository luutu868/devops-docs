# CI/CD Fundamentals

## 🔄 CI/CD Là Gì?

### **Continuous Integration (CI)**

**CI** là practice tự động integrate code changes từ nhiều developers vào shared repository **thường xuyên** (multiple times per day).

**Vấn đề trước khi có CI:**
- ❌ Integration hell: Merge conflicts khi integrate cuối project
- ❌ Bug phát hiện muộn
- ❌ Manual testing tốn thời gian
- ❌ Deployment khó khăn

**CI giải quyết:**
- ✅ Automated builds on every commit
- ✅ Automated testing
- ✅ Early bug detection
- ✅ Faster feedback loop

```
Developer → Commit → CI Server → Build → Test → Report
                         ↓ (if pass)
                    Artifact Repository
```

### **Continuous Delivery (CD)**

**CD** là practice đảm bảo code **luôn sẵn sàng** để deploy to production.

**Process:**
```
CI → Artifact → Staging Deploy → Manual Approval → Production Deploy
```

### **Continuous Deployment**

Tự động deploy to production **without manual approval**.

**Process:**
```
CI → Artifact → Staging Deploy → Automated Tests → Production Deploy
```

### **CI vs CD vs CD**

| Aspect | Continuous Integration | Continuous Delivery | Continuous Deployment |
|--------|----------------------|-------------------|---------------------|
| **Automation** | Build + Test | Build + Test + Deploy to Staging | Build + Test + Deploy to Prod |
| **Manual Step** | None | Manual approval for production | None |
| **Frequency** | Every commit | Ready anytime | Every successful build |
| **Risk** | Low | Medium | Requires mature testing |

## 🏗️ CI/CD Pipeline Architecture

### **Basic Pipeline**

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Code    │────▶│  Build   │────▶│   Test   │────▶│  Deploy  │
│  Commit  │     │          │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

### **Complete Pipeline**

```
┌────────────────────────────────────────────────────────────────┐
│                       CI/CD Pipeline                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Source          2. Build        3. Test                   │
│  ┌──────────┐      ┌──────────┐    ┌──────────────┐          │
│  │ Git Push │─────▶│ Compile  │───▶│ Unit Tests   │          │
│  │          │      │ Docker   │    │ Integration  │          │
│  └──────────┘      └──────────┘    │ Security     │          │
│                                     └──────┬───────┘          │
│                                            ↓                   │
│  4. Package         5. Deploy       6. Monitor                │
│  ┌──────────┐      ┌──────────┐    ┌──────────────┐          │
│  │ Artifact │─────▶│ Dev      │───▶│ Logs         │          │
│  │ Registry │      │ Staging  │    │ Metrics      │          │
│  └──────────┘      │ Prod     │    │ Alerts       │          │
│                    └──────────┘    └──────────────┘          │
└────────────────────────────────────────────────────────────────┘
```

## 🔑 Core Concepts

### **1. Version Control**

Central hub for source code.

**Best Practices:**
- Use Git (GitHub, GitLab, Bitbucket)
- Branching strategy (Git Flow, Trunk-Based)
- Meaningful commit messages
- Code reviews via Pull Requests

### **2. Build Automation**

Compile code, run tests, create artifacts.

**Tools:**
- **Build Tools**: Maven, Gradle, npm, pip
- **Containerization**: Docker
- **Artifacts**: JAR, WAR, Docker images

**Example Build Script:**
```bash
# Node.js
npm install
npm run build
npm test

# Java
mvn clean package
mvn test

# Docker
docker build -t myapp:${VERSION} .
docker push myapp:${VERSION}
```

### **3. Automated Testing**

**Testing Pyramid:**
```
        ┌─────────────┐
        │     E2E     │  ← Few (slow, expensive)
        ├─────────────┤
        │ Integration │  ← Some (medium)
        ├─────────────┤
        │    Unit     │  ← Many (fast, cheap)
        └─────────────┘
```

**Test Types:**

| Type | Coverage | Speed | Cost |
|------|----------|-------|------|
| **Unit Tests** | Single functions | Fast (ms) | Low |
| **Integration Tests** | Component interactions | Medium (seconds) | Medium |
| **E2E Tests** | Full user flows | Slow (minutes) | High |
| **Security Tests** | Vulnerabilities | Medium | Medium |
| **Performance Tests** | Load, stress | Slow | High |

### **4. Artifact Management**

Store build outputs.

**Artifact Repositories:**
- **Docker Images**: Docker Hub, ECR, GCR, ACR
- **Packages**: npm registry, PyPI, Maven Central
- **Generic**: Artifactory, Nexus
- **Cloud**: S3, GCS, Azure Blob

### **5. Deployment Automation**

**Deployment Strategies:**

#### **Blue-Green Deployment**
```
┌─────────────────────────────────┐
│  Load Balancer                  │
└───────────┬─────────────────────┘
            │
    ┌───────┴────────┐
    ↓                ↓
┌────────┐      ┌────────┐
│ Blue   │      │ Green  │
│ v1.0   │      │ v2.0   │
│(active)│      │(idle)  │
└────────┘      └────────┘

Switch traffic: Blue ↔ Green
Rollback: Switch back
```

#### **Canary Deployment**
```
┌─────────────────────────────────┐
│  Load Balancer                  │
└───────────┬─────────────────────┘
            │
    ┌───────┴────────────┐
    │                    │
  95% v1.0            5% v2.0
    │                    │
Gradually increase v2.0 traffic
```

#### **Rolling Update**
```
Initial:    [v1] [v1] [v1] [v1]
Step 1:     [v2] [v1] [v1] [v1]
Step 2:     [v2] [v2] [v1] [v1]
Step 3:     [v2] [v2] [v2] [v1]
Final:      [v2] [v2] [v2] [v2]
```

### **6. Monitoring & Feedback**

**Metrics to Track:**
- Build success rate
- Test coverage
- Deployment frequency
- Mean Time to Recovery (MTTR)
- Change failure rate

## 🛠️ CI/CD Tools Ecosystem

### **CI/CD Platforms**

| Tool | Type | Best For |
|------|------|----------|
| **Jenkins** | Self-hosted | Flexibility, plugins |
| **GitLab CI/CD** | Built-in Git | All-in-one solution |
| **GitHub Actions** | Cloud | GitHub integration |
| **CircleCI** | Cloud | Fast builds |
| **Travis CI** | Cloud | Open source |
| **TeamCity** | Self-hosted | JetBrains ecosystem |
| **Bamboo** | Self-hosted | Atlassian suite |
| **Azure DevOps** | Cloud | Microsoft stack |

### **GitOps Tools**

| Tool | Description |
|------|-------------|
| **ArgoCD** | Declarative GitOps for Kubernetes |
| **Flux** | GitOps operator for Kubernetes |
| **Spinnaker** | Multi-cloud CD platform |

### **Container Orchestration**

| Tool | Use Case |
|------|----------|
| **Kubernetes** | Container orchestration |
| **Docker Swarm** | Simple Docker clustering |
| **Nomad** | Multi-workload orchestration |

## 📊 DORA Metrics

**DevOps Research and Assessment (DORA)** key metrics:

### **1. Deployment Frequency**

How often you deploy to production.

**Elite**: Multiple per day
**High**: Once per day to once per week
**Medium**: Once per week to once per month
**Low**: Less than once per month

### **2. Lead Time for Changes**

Time from commit to production.

**Elite**: Less than 1 hour
**High**: 1 day to 1 week
**Medium**: 1 week to 1 month
**Low**: More than 1 month

### **3. Mean Time to Recovery (MTTR)**

Time to restore service after incident.

**Elite**: Less than 1 hour
**High**: Less than 1 day
**Medium**: 1 day to 1 week
**Low**: More than 1 week

### **4. Change Failure Rate**

Percentage of deployments causing failures.

**Elite**: 0-15%
**High**: 16-30%
**Medium**: 31-45%
**Low**: 46-60%

## 🎯 CI/CD Best Practices

### **1. Commit Often**

```bash
# Bad: Large commits after days of work
git commit -m "Fixed everything"

# Good: Small, frequent commits
git commit -m "Add user validation"
git commit -m "Fix login bug"
git commit -m "Update error messages"
```

### **2. Keep Builds Fast**

**Target:** Under 10 minutes

```yaml
# Optimize with:
- Parallel execution
- Caching dependencies
- Incremental builds
- Split long tests
```

### **3. Test Automation**

```yaml
# Minimum automated tests
- Unit tests (100% coverage goal)
- Integration tests (critical paths)
- Security scanning
- Code quality checks (linting)
```

### **4. Fail Fast**

```yaml
# Pipeline order
stages:
  - lint        # Fast (seconds)
  - unit-test   # Fast (seconds)
  - build       # Medium (minutes)
  - integration # Slow (minutes)
  - deploy      # Slowest
```

### **5. Single Source of Truth**

```yaml
# Everything in Git
- Application code
- Infrastructure code (Terraform)
- Configuration (Kubernetes YAML)
- Pipeline definitions (.gitlab-ci.yml)
- Documentation
```

### **6. Immutable Artifacts**

```bash
# Build once, deploy everywhere
Build → Docker Image (tag: v1.2.3)
Deploy: Dev → Staging → Production
(Same image, different config)
```

### **7. Environment Parity**

```yaml
# Keep environments similar
Dev ≈ Staging ≈ Production

# Use same:
- Container images
- Infrastructure setup
- Configuration structure
```

### **8. Security Integration**

```yaml
# Security in pipeline
stages:
  - dependency-scan    # OWASP Dependency Check
  - secret-scan        # GitLeaks, TruffleHog
  - sast               # Static analysis (SonarQube)
  - container-scan     # Trivy, Clair
  - dast               # Dynamic testing (OWASP ZAP)
```

## 🚀 Example Pipelines

### **Simple Node.js Pipeline**

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  NODE_VERSION: "18"

build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

test:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm install
    - npm run test
    - npm run lint
  coverage: '/Statements\s*:\s*(\d+\.?\d*)%/'

deploy:staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
    - scp -r dist/ user@staging-server:/var/www/
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy:production:
  stage: deploy
  script:
    - echo "Deploying to production..."
    - scp -r dist/ user@prod-server:/var/www/
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual
```

### **Docker Build Pipeline**

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

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

### **Kubernetes Deploy Pipeline**

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy:k8s:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials admin --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=admin
    - kubectl config use-context default
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    kubernetes:
      namespace: production
  only:
    - main
```

## 🔍 Debugging Pipelines

### **Common Issues**

**1. Build Failures**
```bash
# Check logs
- Review build output
- Verify dependencies installed
- Check environment variables

# Solutions
- Run build locally
- Update dependencies
- Fix broken tests
```

**2. Test Failures**
```bash
# Debug
- Run tests locally
- Check test reports
- Review coverage

# Solutions
- Fix failing tests
- Update test data
- Mock external services
```

**3. Deployment Failures**
```bash
# Check
- Credentials correct?
- Network access?
- Resource quotas?

# Solutions
- Verify secrets
- Check firewall rules
- Review deployment logs
```

## 📈 Maturity Model

### **Level 1: Manual**
- Manual builds
- Manual testing
- Manual deployments
- No automation

### **Level 2: Basic CI**
- Automated builds
- Some automated tests
- Manual deployments
- Build on commit

### **Level 3: Advanced CI**
- Comprehensive testing
- Code quality checks
- Security scanning
- Artifact management

### **Level 4: Continuous Delivery**
- Automated deployments to staging
- Manual approval for production
- Environment parity
- Rollback capability

### **Level 5: Continuous Deployment**
- Fully automated production deploys
- Feature flags
- Canary/Blue-green deployments
- Advanced monitoring

## ✅ Quick Reference

### **Pipeline Stages**
```yaml
1. Source     → Git checkout
2. Build      → Compile, package
3. Test       → Unit, integration
4. Scan       → Security, quality
5. Package    → Create artifact
6. Deploy     → Dev → Staging → Prod
7. Monitor    → Logs, metrics, alerts
```

### **Essential Tools**
```bash
# Version Control
Git + GitHub/GitLab

# CI/CD
Jenkins / GitLab CI / GitHub Actions

# Container
Docker + Kubernetes

# Monitoring
Prometheus + Grafana

# Security
OWASP, Trivy, SonarQube
```

---

**Tiếp theo**: [Jenkins Pipeline →](../jenkins/jenkins-pipeline.md)
