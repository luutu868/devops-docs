# Dockerfile Best Practices

## Tổng quan về Dockerfile

### Dockerfile là gì?

Dockerfile là file text chứa các instructions để build Docker image. Mỗi instruction tạo một layer trong image.

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx
COPY index.html /var/www/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Image Layers

```
┌─────────────────────────────────┐
│  CMD ["nginx", "-g", ...]       │ ← Layer 4 (Metadata)
├─────────────────────────────────┤
│  COPY index.html /var/www/      │ ← Layer 3 (1KB)
├─────────────────────────────────┤
│  RUN apt-get install nginx      │ ← Layer 2 (50MB)
├─────────────────────────────────┤
│  FROM ubuntu:22.04              │ ← Layer 1 (Base 70MB)
└─────────────────────────────────┘
         Total: ~120MB
```

**Layer characteristics:**
- Read-only (immutable)
- Cached và reusable
- Stacked theo thứ tự
- Mỗi instruction = 1 layer

## Dockerfile Instructions

### FROM - Base Image

```dockerfile
# Official image
FROM node:18-alpine

# Specific version (recommended)
FROM python:3.11-slim

# Multi-stage build
FROM golang:1.21 AS builder
FROM alpine:3.18 AS runtime

# Scratch (empty base)
FROM scratch
```

**Best practices:**
- Use official images
- Specify exact version tags
- Prefer slim/alpine variants
- Avoid `latest` tag

### RUN - Execute Commands

```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim

# Good: Single layer
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Use backslash for readability
RUN apk add --no-cache \
    git \
    openssh \
    ca-certificates

# Multi-line với &&
RUN set -ex && \
    apt-get update && \
    apt-get install -y nginx && \
    apt-get clean
```

### COPY vs ADD

```dockerfile
# COPY - Preferred for simple file copy
COPY package.json /app/
COPY src/ /app/src/

# COPY with ownership
COPY --chown=node:node package.json /app/

# ADD - Auto-extraction và remote URLs
ADD archive.tar.gz /app/        # Auto-extracts
ADD https://example.com/file /  # Downloads

# Best practice: Use COPY unless you need ADD features
```

### WORKDIR - Set Working Directory

```dockerfile
# Bad
RUN cd /app && npm install

# Good
WORKDIR /app
RUN npm install

# Multiple WORKDIR
WORKDIR /app
WORKDIR /app/src   # Now in /app/src
```

### ENV - Environment Variables

```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info

# Use in commands
ENV APP_HOME=/app
WORKDIR $APP_HOME
COPY . $APP_HOME
```

### ARG - Build Arguments

```dockerfile
# Define build argument
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

# ARG with default value
ARG APP_ENV=production
ENV NODE_ENV=$APP_ENV

# Multiple stages
ARG BUILDPLATFORM
FROM --platform=$BUILDPLATFORM golang:1.21
```

**ARG vs ENV:**
- `ARG`: Available during build only
- `ENV`: Available during build + runtime

### EXPOSE - Document Ports

```dockerfile
# Single port
EXPOSE 80

# Multiple ports
EXPOSE 80 443

# UDP port
EXPOSE 53/udp

# Note: EXPOSE doesn't publish ports
# Use -p flag when running container
```

### USER - Set User

```dockerfile
# Create and switch to non-root user
RUN useradd -m -u 1000 appuser
USER appuser

# Good practice example
FROM node:18-alpine
RUN addgroup -g 1000 node && \
    adduser -D -u 1000 -G node node
WORKDIR /app
COPY --chown=node:node . .
USER node
CMD ["node", "server.js"]
```

### VOLUME - Mount Points

```dockerfile
# Define volume
VOLUME /data

# Multiple volumes
VOLUME ["/data", "/logs"]

# Best practice: Document where data persists
VOLUME /var/lib/postgresql/data
```

### CMD vs ENTRYPOINT

```dockerfile
# CMD - Default command (can be overridden)
CMD ["nginx", "-g", "daemon off;"]
# Run: docker run myimage        → nginx
# Run: docker run myimage bash   → bash

# ENTRYPOINT - Always runs
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
# Run: docker run myimage              → nginx -g daemon off;
# Run: docker run myimage -v           → nginx -v

# Exec form (preferred)
CMD ["python", "app.py"]

# Shell form (avoid)
CMD python app.py
```

**Best practices:**
```dockerfile
# Flexible binary
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]

# Static command
CMD ["python", "app.py"]

# Script + args
ENTRYPOINT ["/entrypoint.sh"]
CMD ["start"]
```

### LABEL - Metadata

```dockerfile
LABEL maintainer="devops@example.com"
LABEL version="1.0"
LABEL description="My awesome application"

# OCI annotations
LABEL org.opencontainers.image.authors="DevOps Team"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/org/repo"
```

### HEALTHCHECK - Health Monitoring

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# TCP check
HEALTHCHECK CMD nc -z localhost 3000 || exit 1

# Script-based
HEALTHCHECK CMD /health-check.sh

# Disable inherited healthcheck
HEALTHCHECK NONE
```

## Multi-Stage Builds

### Basic Multi-Stage

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**Size comparison:**
- Single-stage: 900MB (includes build tools)
- Multi-stage: 150MB (only runtime)

### Golang Multi-Stage

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server

# Production stage (minimal)
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/server ./
EXPOSE 8080
CMD ["./server"]
```

**Result:**
- Builder: 800MB
- Final: 15MB

### Python Multi-Stage

```dockerfile
# Build stage with dependencies
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

### Advanced Multi-Stage with Dependencies

```dockerfile
# Base stage
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

# Dependencies stage
FROM base AS dependencies
RUN npm ci --only=production
RUN cp -R node_modules prod_node_modules
RUN npm ci  # Include dev dependencies

# Build stage
FROM base AS builder
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Test stage
FROM base AS test
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm test

# Production stage
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=dependencies /app/prod_node_modules ./node_modules
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

## Best Practices

### 1. Minimize Layer Size

```dockerfile
# Bad: Large layer
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl
# ... use tools ...
# Build tools remain in image (500MB)

# Good: Clean up in same layer
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    && npm install \
    && apt-get purge -y build-essential \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*
```

### 2. Leverage Build Cache

```dockerfile
# Bad: Cache busted frequently
COPY . /app/
RUN npm install

# Good: Install dependencies first
COPY package*.json /app/
RUN npm ci
COPY . /app/

# Cache hits when only source code changes
```

**Layer ordering strategy:**
```dockerfile
FROM node:18-alpine
WORKDIR /app

# 1. System dependencies (rarely change)
RUN apk add --no-cache git

# 2. Application dependencies (change occasionally)
COPY package*.json ./
RUN npm ci

# 3. Source code (change frequently)
COPY . .

# 4. Build
RUN npm run build
```

### 3. Use .dockerignore

```bash
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
*.log
dist/
coverage/

# Documentation
docs/
*.md
```

### 4. Run as Non-Root User

```dockerfile
# Method 1: Create user
FROM ubuntu:22.04
RUN useradd -m -u 1000 -s /bin/bash appuser
USER appuser

# Method 2: Use existing user (alpine)
FROM node:18-alpine
USER node

# Method 3: Explicit UID/GID
FROM python:3.11-slim
RUN groupadd -r -g 1000 appgroup && \
    useradd -r -u 1000 -g appgroup appuser
USER 1000:1000
```

### 5. Use Specific Tags

```dockerfile
# Bad
FROM node:latest          # Unpredictable
FROM ubuntu               # Unpredictable

# Good
FROM node:18.17.1-alpine3.18
FROM ubuntu:22.04

# Even better: Use digest
FROM node:18-alpine@sha256:abc123...
```

### 6. Combine RUN Commands

```dockerfile
# Bad: 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good: 1 layer
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# Best: Readable and optimized
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        netcat \
    ; \
    rm -rf /var/lib/apt/lists/*
```

### 7. Pin Package Versions

```dockerfile
# Bad
RUN apt-get install -y nginx

# Good
RUN apt-get install -y nginx=1.18.0-0ubuntu1

# Python
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# requirements.txt
flask==2.3.2
gunicorn==20.1.0
redis==4.5.5
```

### 8. Use COPY Instead of ADD

```dockerfile
# Prefer COPY
COPY package.json /app/
COPY src/ /app/src/

# Use ADD only for tar extraction
ADD app.tar.gz /app/

# Not for URLs (use curl/wget in RUN)
RUN curl -L https://example.com/file.tar.gz | tar -xz
```

### 9. Minimize Installed Packages

```dockerfile
# Install only what's needed
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    package1 \
    package2 \
    && rm -rf /var/lib/apt/lists/*

# Alpine: use --no-cache
RUN apk add --no-cache \
    nodejs \
    npm
```

### 10. Use Multi-Stage for Secrets

```dockerfile
# Build stage (secrets here)
FROM golang:1.21 AS builder
ARG GITHUB_TOKEN
RUN git config --global url."https://${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"
RUN go build -o app

# Production (no secrets)
FROM alpine:3.18
COPY --from=builder /app/app ./
CMD ["./app"]

# GITHUB_TOKEN not in final image
```

## Real-World Examples

### Node.js Production Dockerfile

```dockerfile
FROM node:18-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Dependencies
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci

# Builder
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production
FROM base AS runner
ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### Python FastAPI Application

```dockerfile
FROM python:3.11-slim AS builder

# System dependencies
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Python dependencies
WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# Production stage
FROM python:3.11-slim

# Create user
RUN useradd -m -u 1000 appuser

# Copy wheels and install
COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache /wheels/*

# Copy application
WORKDIR /app
COPY --chown=appuser:appuser . .

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Go Binary (Scratch)

```dockerfile
FROM golang:1.21-alpine AS builder

# Install SSL certificates
RUN apk add --no-cache ca-certificates

WORKDIR /app

# Dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -o /app/server

# Minimal production image
FROM scratch

# Copy SSL certs
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /app/server /server

# Copy static files if needed
COPY --from=builder /app/static /static

EXPOSE 8080
USER 1000:1000

ENTRYPOINT ["/server"]
```

### Nginx with Custom Config

```dockerfile
FROM nginx:1.24-alpine

# Remove default config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom config
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/

# Copy static files
COPY dist/ /usr/share/nginx/html/

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

# Create cache directories
RUN mkdir -p /var/cache/nginx/client_temp && \
    chown -R nginx:nginx /var/cache/nginx

EXPOSE 80 443

USER nginx

CMD ["nginx", "-g", "daemon off;"]
```

## Advanced Techniques

### Build Arguments for Flexibility

```dockerfile
ARG NODE_VERSION=18
ARG ALPINE_VERSION=3.18

FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION}

ARG BUILD_DATE
ARG VCS_REF
ARG VERSION

LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.version=$VERSION

# Build command:
# docker build \
#   --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
#   --build-arg VCS_REF=$(git rev-parse --short HEAD) \
#   --build-arg VERSION=1.0.0 \
#   -t myapp:1.0.0 .
```

### Dynamic Entrypoint Script

```dockerfile
FROM python:3.11-slim

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

WORKDIR /app
COPY . .

ENTRYPOINT ["/entrypoint.sh"]
CMD ["python", "app.py"]
```

```bash
#!/bin/sh
# entrypoint.sh

set -e

# Wait for database
echo "Waiting for database..."
while ! nc -z db 5432; do
  sleep 0.1
done
echo "Database ready!"

# Run migrations
python manage.py migrate

# Execute command
exec "$@"
```

### BuildKit Features

**Enable BuildKit:**
```bash
export DOCKER_BUILDKIT=1
docker build -t myapp .
```

**Dockerfile with BuildKit features:**
```dockerfile
# syntax=docker/dockerfile:1

FROM ubuntu:22.04

# Mount secrets (not in image)
RUN --mount=type=secret,id=github_token \
    TOKEN=$(cat /run/secrets/github_token) && \
    git clone https://$TOKEN@github.com/private/repo

# Cache mount (speeds up builds)
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y nginx

# Bind mount (access files without COPY)
RUN --mount=type=bind,source=.,target=/src \
    cd /src && make build
```

### Testing in Multi-Stage Build

```dockerfile
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS test
COPY . .
RUN npm run lint
RUN npm run test
RUN npm run test:e2e

FROM base AS production
COPY . .
RUN npm run build
CMD ["node", "dist/main.js"]

# Build with testing:
# docker build --target test -t myapp:test .
# docker build --target production -t myapp:latest .
```

## Security Best Practices

### 1. Scan Images for Vulnerabilities

```bash
# Trivy scanner
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image myapp:latest

# Snyk
snyk container test myapp:latest

# Docker Scout
docker scout cves myapp:latest
```

### 2. Use Distroless Images

```dockerfile
# Google distroless (no shell, no package manager)
FROM gcr.io/distroless/python3-debian11
COPY --from=builder /app /app
WORKDIR /app
CMD ["app.py"]
```

### 3. Don't Store Secrets in Images

```dockerfile
# Bad
ENV API_KEY=secret123

# Good: Use runtime secrets
# docker run -e API_KEY=$API_KEY myapp

# Or Docker secrets (Swarm/Compose)
# docker secret create api_key api_key.txt
```

## Optimization Checklist

### Build Process
- [ ] Use `.dockerignore`
- [ ] Layer caching strategy
- [ ] Multi-stage builds
- [ ] Combine RUN commands
- [ ] Remove build dependencies

### Image Size
- [ ] Use alpine/slim variants
- [ ] Clean package cache
- [ ] Remove unnecessary files
- [ ] Minimize layers
- [ ] Use distroless for production

### Security
- [ ] Run as non-root user
- [ ] Scan for vulnerabilities
- [ ] Pin base image versions
- [ ] Don't include secrets
- [ ] Use READ-ONLY filesystem where possible

### Runtime
- [ ] Add HEALTHCHECK
- [ ] Document EXPOSE ports
- [ ] Use ENTRYPOINT + CMD
- [ ] Set proper signals handling
- [ ] Add metadata LABELs

## Troubleshooting

### Debug Build Issues

```bash
# Build with verbose output
docker build --progress=plain -t myapp .

# Build without cache
docker build --no-cache -t myapp .

# Build specific stage
docker build --target builder -t myapp:builder .

# Inspect layers
docker history myapp:latest
docker history --no-trunc myapp:latest
```

### Inspect Image

```bash
# Image details
docker image inspect myapp:latest

# Check size
docker images myapp

# Analyze layers
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    wagoodman/dive myapp:latest
```

## Tổng kết

**Key takeaways:**
- Multi-stage builds giảm image size 80-90%
- Layer ordering quan trọng cho caching
- Security: non-root user, scan vulnerabilities
- Optimization: minimize layers, clean caches
- Reproducibility: pin versions, use specific tags

**Dockerfile checklist:**
1. FROM: Official image với specific tag
2. RUN: Combined commands, clean cache
3. COPY: Use `.dockerignore`, copy dependencies first
4. USER: Run as non-root
5. HEALTHCHECK: Monitor container health
6. Multi-stage: Separate build từ runtime

**Next steps:**
- Practice với real applications
- Measure image sizes before/after optimization
- Setup CI/CD để scan images
- Document Docker build process
- Monitor production images regularly
