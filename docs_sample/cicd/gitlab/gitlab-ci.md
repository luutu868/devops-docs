# GitLab CI/CD

## 🦊 GitLab CI/CD Overview

**GitLab CI/CD** là built-in CI/CD solution của GitLab - không cần setup server riêng.

**Key Features:**
- ✅ Built-in to GitLab (no external tool)
- ✅ YAML-based configuration (.gitlab-ci.yml)
- ✅ Auto DevOps
- ✅ Container Registry
- ✅ Security scanning
- ✅ Environment management
- ✅ Review Apps

**Architecture:**
```
┌─────────────────────────────────────────┐
│         GitLab Server                   │
│  ┌───────────────────────────────────┐  │
│  │  .gitlab-ci.yml (in repository)   │  │
│  └────────────┬──────────────────────┘  │
└───────────────┼─────────────────────────┘
                ↓
┌───────────────────────────────────────┐
│        GitLab Runner                  │
│  ┌─────────────────────────────────┐  │
│  │  Execute Pipeline Jobs          │  │
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
```

## 🏃 GitLab Runner Setup

### **Install Runner (Docker)**

```bash
# Run GitLab Runner container
docker run -d \
  --name gitlab-runner \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v gitlab-runner-config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
```

### **Install Runner (Linux)**

```bash
# Ubuntu/Debian
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Start runner
sudo gitlab-runner start
```

### **Register Runner**

```bash
# Interactive registration
sudo gitlab-runner register

# Or non-interactive
sudo gitlab-runner register \
  --url "https://gitlab.com/" \
  --registration-token "YOUR_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "docker-runner" \
  --tag-list "docker,aws" \
  --run-untagged="true" \
  --locked="false"
```

**Get Registration Token:**
- Project: Settings → CI/CD → Runners
- Group: Settings → CI/CD → Runners  
- Instance: Admin Area → Overview → Runners

### **Runner Executors**

| Executor | Description | Use Case |
|----------|-------------|----------|
| **docker** | Run in Docker containers | Most common |
| **shell** | Run on host | Simple scripts |
| **kubernetes** | Run in K8s pods | K8s clusters |
| **docker+machine** | Auto-scale with Docker Machine | AWS/GCP auto-scaling |

## 📝 .gitlab-ci.yml Basics

### **Simple Pipeline**

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building application..."
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/

test-job:
  stage: test
  script:
    - echo "Running tests..."
    - npm test

deploy-job:
  stage: deploy
  script:
    - echo "Deploying application..."
    - ./deploy.sh
  environment:
    name: production
  only:
    - main
```

### **Pipeline Structure**

```yaml
# Global settings
image: node:18
variables:
  NODE_ENV: production

# Stages definition
stages:
  - build
  - test
  - deploy

# Before script (runs before every job)
before_script:
  - echo "Starting job..."

# Jobs
job-name:
  stage: build
  image: alpine:latest  # Override global image
  variables:
    CUSTOM_VAR: value
  before_script:
    - apk add curl
  script:
    - echo "Main commands"
  after_script:
    - echo "Cleanup"
  only:
    - main
  tags:
    - docker
```

## 🎯 Core Concepts

### **1. Stages**

```yaml
stages:
  - .pre          # Special: runs first
  - build
  - test
  - security
  - deploy
  - .post         # Special: runs last

# Jobs in same stage run in parallel
# Next stage waits for previous to complete
```

### **2. Jobs**

```yaml
job-name:
  stage: build
  image: node:18
  
  # Scripts
  before_script:
    - npm install
  script:
    - npm run build
  after_script:
    - echo "Job finished"
  
  # Artifacts
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  
  # Cache
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  
  # Rules
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  
  # Tags (select runner)
  tags:
    - docker
    
  # Retry
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
```

### **3. Variables**

```yaml
# Global variables
variables:
  GLOBAL_VAR: "value"
  DATABASE_URL: "postgres://db:5432"

# Job variables
job:
  variables:
    JOB_VAR: "value"
  script:
    - echo $GLOBAL_VAR
    - echo $JOB_VAR
    - echo $CI_COMMIT_SHA  # Built-in GitLab variable
```

**Built-in Variables:**

| Variable | Description |
|----------|-------------|
| `CI_COMMIT_SHA` | Git commit SHA |
| `CI_COMMIT_BRANCH` | Branch name |
| `CI_COMMIT_TAG` | Tag name |
| `CI_PROJECT_NAME` | Project name |
| `CI_PIPELINE_ID` | Pipeline ID |
| `CI_JOB_ID` | Job ID |
| `CI_REGISTRY` | GitLab Container Registry |
| `CI_REGISTRY_IMAGE` | Full image path |

### **4. Artifacts**

```yaml
build:
  script:
    - npm run build
  artifacts:
    name: "build-${CI_COMMIT_REF_NAME}"
    paths:
      - dist/
      - public/
    exclude:
      - dist/*.map
    expire_in: 30 days
    when: on_success  # on_success, on_failure, always

test:
  script:
    - npm test
  dependencies:
    - build  # Download artifacts from build job
```

### **5. Cache**

```yaml
# Cache dependencies between pipeline runs
cache:
  key: ${CI_COMMIT_REF_SLUG}  # Per branch
  paths:
    - node_modules/
    - .npm/
  policy: pull-push  # pull, push, pull-push

# Job-specific cache
job:
  cache:
    key: ${CI_JOB_NAME}
    paths:
      - vendor/
```

**Cache vs Artifacts:**

| Feature | Cache | Artifacts |
|---------|-------|-----------|
| **Purpose** | Speed up builds | Pass data between stages |
| **Duration** | Long-term | Short-term |
| **Between** | Pipelines | Jobs in same pipeline |

### **6. Rules & Only/Except**

```yaml
# Using rules (recommended)
deploy:
  script: ./deploy.sh
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    - if: '$CI_COMMIT_BRANCH == "develop"'
      when: manual
    - when: never

# Using only/except (legacy)
deploy:
  script: ./deploy.sh
  only:
    - main
    - tags
  except:
    - schedules

# Complex rules
job:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS'
      when: never
    - if: '$CI_COMMIT_BRANCH'
```

## 🚀 Real-World Pipelines

### **Node.js Application**

```yaml
image: node:18

stages:
  - build
  - test
  - security
  - deploy

variables:
  NPM_CONFIG_CACHE: "$CI_PROJECT_DIR/.npm"

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

before_script:
  - npm ci --cache .npm --prefer-offline

build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

lint:
  stage: test
  script:
    - npm run lint

test:unit:
  stage: test
  script:
    - npm run test:unit
  coverage: '/Statements\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:integration:
  stage: test
  services:
    - postgres:14
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: user
    POSTGRES_PASSWORD: password
  script:
    - npm run test:integration

security:dependency-scan:
  stage: security
  script:
    - npm audit --production --audit-level=high

security:sast:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto .

deploy:staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
  script:
    - echo "Deploying to staging..."
    - curl -X POST $STAGING_WEBHOOK
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy:production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
  script:
    - echo "Deploying to production..."
    - curl -X POST $PRODUCTION_WEBHOOK
  environment:
    name: production
    url: https://example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

### **Docker Build & Push**

```yaml
image: docker:latest

services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2376
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

stages:
  - build
  - test
  - push
  - deploy

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker images

test:
  stage: test
  script:
    - docker run --rm $IMAGE_TAG npm test

security:scan:
  stage: test
  script:
    - apk add --no-cache curl
    - |
      docker run --rm \
        -v /var/run/docker.sock:/var/run/docker.sock \
        aquasec/trivy image \
        --severity HIGH,CRITICAL \
        $IMAGE_TAG

push:
  stage: push
  script:
    - docker push $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
    - develop

deploy:k8s:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context $KUBE_CONTEXT
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n production
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    kubernetes:
      namespace: production
  only:
    - main
  when: manual
```

### **Multi-Stage Build (Monorepo)**

```yaml
stages:
  - build
  - test
  - deploy

variables:
  FRONTEND_DIR: "frontend"
  BACKEND_DIR: "backend"

# Frontend jobs
build:frontend:
  stage: build
  image: node:18
  script:
    - cd $FRONTEND_DIR
    - npm ci
    - npm run build
  artifacts:
    paths:
      - $FRONTEND_DIR/dist/
  only:
    changes:
      - frontend/**/*

test:frontend:
  stage: test
  image: node:18
  script:
    - cd $FRONTEND_DIR
    - npm ci
    - npm test
  only:
    changes:
      - frontend/**/*

deploy:frontend:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache aws-cli
  script:
    - cd $FRONTEND_DIR
    - aws s3 sync dist/ s3://my-bucket/ --delete
    - aws cloudfront create-invalidation --distribution-id $CDN_ID --paths "/*"
  only:
    refs:
      - main
    changes:
      - frontend/**/*

# Backend jobs
build:backend:
  stage: build
  image: maven:3.8-openjdk-11
  script:
    - cd $BACKEND_DIR
    - mvn clean package -DskipTests
  artifacts:
    paths:
      - $BACKEND_DIR/target/*.jar
  only:
    changes:
      - backend/**/*

test:backend:
  stage: test
  image: maven:3.8-openjdk-11
  services:
    - postgres:14
  variables:
    POSTGRES_DB: testdb
  script:
    - cd $BACKEND_DIR
    - mvn test
  only:
    changes:
      - backend/**/*

deploy:backend:
  stage: deploy
  image: alpine:latest
  script:
    - cd $BACKEND_DIR
    - scp target/*.jar user@server:/opt/app/
    - ssh user@server "sudo systemctl restart app"
  only:
    refs:
      - main
    changes:
      - backend/**/*
  when: manual
```

### **Matrix Builds**

```yaml
test:
  image: node:${NODE_VERSION}
  parallel:
    matrix:
      - NODE_VERSION: ["16", "18", "20"]
  script:
    - npm ci
    - npm test

# Results in 3 jobs:
# - test: [16]
# - test: [18]
# - test: [20]
```

### **Parent-Child Pipelines**

```yaml
# .gitlab-ci.yml (parent)
trigger:child:
  trigger:
    include: ci/child-pipeline.yml
    strategy: depend

# ci/child-pipeline.yml (child)
child-job:
  script:
    - echo "Running child pipeline"
```

## 🐳 Docker Integration

### **Docker-in-Docker (DinD)**

```yaml
build:docker:
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker info
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA
```

### **Using GitLab Container Registry**

```yaml
build:
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
```

## 🔐 Security & Secrets

### **Protected Variables**

Settings → CI/CD → Variables

```yaml
# Use in pipeline
deploy:
  script:
    - echo "API Key: $API_KEY"
    - curl -H "Authorization: Bearer $API_TOKEN" ...
```

**Variable Types:**
- **Protected**: Only available on protected branches
- **Masked**: Hidden in job logs
- **File**: Create temp file with content

### **Security Scanning**

GitLab provides built-in security scanners:

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/License-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
```

## 🌍 Environments

```yaml
deploy:staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop:staging

deploy:production:
  stage: deploy
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://example.com
    on_stop: stop:production
  when: manual

stop:staging:
  stage: deploy
  script:
    - ./stop.sh staging
  environment:
    name: staging
    action: stop
  when: manual

# View environments: Deployments → Environments
```

## 🎭 Review Apps

```yaml
review:
  stage: deploy
  script:
    - ./deploy-review.sh
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: https://$CI_COMMIT_REF_SLUG.example.com
    on_stop: stop:review
    auto_stop_in: 1 week
  only:
    - merge_requests

stop:review:
  stage: deploy
  script:
    - ./stop-review.sh
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  when: manual
  only:
    - merge_requests
```

## 🎓 Best Practices

### **1. Use Templates**

```yaml
# templates/build.yml
.build_template:
  image: node:18
  before_script:
    - npm ci
  script:
    - npm run build
  artifacts:
    paths:
      - dist/

# .gitlab-ci.yml
include:
  - local: 'templates/build.yml'

build:frontend:
  extends: .build_template
  variables:
    APP_DIR: frontend

build:backend:
  extends: .build_template
  variables:
    APP_DIR: backend
```

### **2. Use Anchors (YAML)**

```yaml
.base_job:
  image: node:18
  before_script: &base_before_script
    - npm ci
    - npm run lint

job1:
  before_script: *base_before_script
  script:
    - npm test

job2:
  before_script:
    - *base_before_script
    - npm audit
  script:
    - npm run build
```

### **3. Keep Pipelines Fast**

```yaml
# Parallel execution
test:
  parallel: 5
  script:
    - npm test -- --shard=$CI_NODE_INDEX/$CI_NODE_TOTAL

# Fail fast
test:
  script:
    - npm test
  allow_failure: false
```

### **4. Use Proper Cache**

```yaml
cache:
  key:
    files:
      - package-lock.json  # Cache key based on file
  paths:
    - node_modules/
  policy: pull-push

# Or per-branch
cache:
  key: ${CI_COMMIT_REF_SLUG}
```

## 🚨 Troubleshooting

### **Common Issues**

```yaml
# Job stuck on "pending"
# - Check runner availability
# - Check tags match

# Cache not working
# - Verify cache key
# - Check runner cache location
# - Use cache:policy correctly

# Artifacts not available
# - Check artifacts:paths
# - Verify expire_in
# - Use dependencies correctly

# Docker permission denied
# - Use docker:dind service
# - Set DOCKER_HOST correctly
```

### **Debug Mode**

```yaml
job:
  variables:
    CI_DEBUG_TRACE: "true"  # Enable debug output
  script:
    - set -x  # Enable command echoing
    - your commands
```

## ✅ Quick Reference

```yaml
# Minimal .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - echo "build commands"

test:
  stage: test
  script:
    - echo "test commands"

deploy:
  stage: deploy
  script:
    - echo "deploy commands"
  only:
    - main
```

---

**Tiếp theo**: [GitHub Actions →](../github-actions/github-actions.md)
