# Docker Security

## Tổng quan Docker Security

### Security Layers

```
┌──────────────────────────────────────┐
│    Application Security              │
│  - Code vulnerabilities              │
│  - Dependency scanning               │
├──────────────────────────────────────┤
│    Container Security                │
│  - Non-root user                     │
│  - Read-only filesystem              │
│  - Resource limits                   │
├──────────────────────────────────────┤
│    Image Security                    │
│  - Base image selection              │
│  - Layer scanning                    │
│  - Secrets management                │
├──────────────────────────────────────┤
│    Host Security                     │
│  - Docker daemon config              │
│  - Kernel capabilities               │
│  - Namespaces & cgroups              │
├──────────────────────────────────────┤
│    Infrastructure Security           │
│  - Network segmentation              │
│  - Registry security                 │
│  - Orchestration (K8s)               │
└──────────────────────────────────────┘
```

### Common Security Risks

1. **Running as root**
2. **Vulnerable base images**
3. **Exposed secrets**
4. **Excessive privileges**
5. **Unpatched vulnerabilities**
6. **Network exposure**
7. **Unrestricted resource usage**

## Image Security

### 1. Base Image Selection

```dockerfile
# Bad: Latest tag, large image
FROM ubuntu:latest

# Good: Specific version, minimal
FROM ubuntu:22.04-slim

# Better: Alpine for smaller attack surface
FROM alpine:3.18

# Best: Distroless (no shell, no package manager)
FROM gcr.io/distroless/static-debian11
```

**Image size comparison:**
```
ubuntu:latest      ~77MB
ubuntu:22.04-slim  ~28MB
alpine:3.18        ~7MB
distroless/static  ~2MB
```

### 2. Vulnerability Scanning

**Trivy:**
```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Scan image
trivy image nginx:latest

# Specific severities
trivy image --severity HIGH,CRITICAL nginx:latest

# Output as JSON
trivy image -f json -o results.json nginx:latest

# Scan Dockerfile
trivy config Dockerfile

# CI/CD integration
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

**Docker Scout:**
```bash
# Enable Docker Scout
docker scout quickview

# Analyze image
docker scout cves myapp:latest

# Compare images
docker scout compare myapp:v1 myapp:v2

# Recommendations
docker scout recommendations myapp:latest
```

**Snyk:**
```bash
# Install Snyk
npm install -g snyk

# Authenticate
snyk auth

# Test image
snyk container test nginx:latest

# Monitor image
snyk container monitor nginx:latest

# Test Dockerfile
snyk iac test Dockerfile
```

### 3. Image Signing and Verification

**Docker Content Trust (DCT):**
```bash
# Enable DCT
export DOCKER_CONTENT_TRUST=1

# Pull image (must be signed)
docker pull nginx:latest

# Push signed image
docker push myregistry/myapp:v1

# View signatures
docker trust inspect myregistry/myapp:v1
```

**Cosign (Sigstore):**
```bash
# Install Cosign
brew install cosign  # macOS
# or download from GitHub releases

# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key myregistry/myapp:v1

# Verify signature
cosign verify --key cosign.pub myregistry/myapp:v1

# Sign with keyless (OIDC)
cosign sign myregistry/myapp:v1
```

### 4. Multi-Stage Builds for Security

```dockerfile
# Build stage (contains build tools)
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server

# Production stage (minimal)
FROM gcr.io/distroless/static-debian11
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

**Benefits:**
- No build tools in final image
- Smaller attack surface
- Reduced image size
- No unnecessary packages

## Container Runtime Security

### 1. Run as Non-Root User

```dockerfile
# Method 1: Create user in Dockerfile
FROM ubuntu:22.04
RUN useradd -m -u 1000 appuser
USER appuser
WORKDIR /home/appuser
COPY --chown=appuser:appuser . .
CMD ["./app"]

# Method 2: Use existing user
FROM node:18-alpine
USER node
WORKDIR /home/node/app
COPY --chown=node:node . .
CMD ["node", "server.js"]

# Method 3: Numeric UID/GID
FROM alpine:3.18
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser
USER 1000:1000
CMD ["./app"]
```

**Verify:**
```bash
# Check user in running container
docker exec container_name whoami
docker exec container_name id

# Inspect image
docker inspect --format='{{.Config.User}}' image_name
```

### 2. Read-Only Filesystem

```bash
# Run with read-only root filesystem
docker run --read-only nginx

# With writable tmpfs for required directories
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  --tmpfs /var/cache/nginx \
  nginx
```

**Dockerfile:**
```dockerfile
FROM nginx:alpine
# Make required directories writable
RUN mkdir -p /var/cache/nginx && \
    chown -R nginx:nginx /var/cache/nginx
USER nginx
```

### 3. Drop Capabilities

```bash
# Drop all capabilities, add only needed ones
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx

# View container capabilities
docker exec container_name capsh --print
```

**Common capabilities:**
- `NET_BIND_SERVICE`: Bind ports < 1024
- `CHOWN`: Change file ownership
- `SETUID/SETGID`: Change UID/GID
- `DAC_OVERRIDE`: Bypass file permissions

**Dockerfile:**
```dockerfile
FROM nginx:alpine
USER nginx
# Nginx will need NET_BIND_SERVICE if using port 80
# Add via docker run or docker-compose
```

### 4. Security Options

```bash
# AppArmor profile
docker run --security-opt apparmor=docker-default nginx

# SELinux labels
docker run --security-opt label=level:s0:c100,c200 nginx

# No new privileges
docker run --security-opt no-new-privileges:true nginx

# Seccomp profile
docker run --security-opt seccomp=default.json nginx
```

**Custom Seccomp Profile:**
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### 5. Resource Limits

```bash
# Memory limits
docker run --memory="512m" --memory-swap="1g" nginx

# CPU limits
docker run --cpus="1.5" nginx

# PID limits (prevent fork bombs)
docker run --pids-limit=100 nginx

# Ulimits
docker run --ulimit nofile=1024:1024 nginx

# Combine all
docker run \
  --memory="1g" \
  --cpus="2" \
  --pids-limit=200 \
  --ulimit nofile=2048:4096 \
  nginx
```

## Secrets Management

### 1. Never Store Secrets in Images

```dockerfile
# Bad: Hardcoded secrets
ENV API_KEY=secret123
ENV DB_PASSWORD=password

# Bad: Build-time secrets in final image
ARG GITHUB_TOKEN
RUN git clone https://${GITHUB_TOKEN}@github.com/repo
# GITHUB_TOKEN visible in image history!

# Good: Use multi-stage to avoid secrets in final image
FROM golang:1.21 AS builder
ARG GITHUB_TOKEN
RUN git config --global url."https://${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"
RUN go build

FROM alpine:3.18
COPY --from=builder /app/binary /binary
# GITHUB_TOKEN not in final image
```

### 2. Docker Secrets (Swarm)

```bash
# Create secret from file
echo "my_secret_password" | docker secret create db_password -

# Create from file
docker secret create db_password ./password.txt

# List secrets
docker secret ls

# Inspect secret
docker secret inspect db_password

# Use in service
docker service create \
  --name myapp \
  --secret db_password \
  myapp:latest

# Access in container: /run/secrets/db_password
```

**Docker Compose with Secrets:**
```yaml
version: '3.8'

services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true  # Created separately
```

### 3. Environment Variables (Better Practices)

```bash
# Pass at runtime (not in image)
docker run -e DB_PASSWORD=secret myapp

# From .env file (keep out of version control)
docker run --env-file .env myapp

# From host environment
docker run -e DB_PASSWORD=$DB_PASSWORD myapp
```

### 4. Secrets with BuildKit

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:3.18

# Mount secret during build (not in final image)
RUN --mount=type=secret,id=github_token \
    TOKEN=$(cat /run/secrets/github_token) && \
    git clone https://$TOKEN@github.com/private/repo

# Secret not in image layers
```

**Build:**
```bash
export DOCKER_BUILDKIT=1
docker build --secret id=github_token,src=token.txt .
```

### 5. External Secret Management

**HashiCorp Vault:**
```bash
# Get secret from Vault
docker run \
  -e VAULT_ADDR=https://vault.example.com \
  -e VAULT_TOKEN=$VAULT_TOKEN \
  myapp
```

```python
# Application code
import hvac
client = hvac.Client(url=os.getenv('VAULT_ADDR'))
client.token = os.getenv('VAULT_TOKEN')
secret = client.secrets.kv.v2.read_secret_version(path='database/config')
db_password = secret['data']['data']['password']
```

**AWS Secrets Manager:**
```python
import boto3

client = boto3.client('secretsmanager')
response = client.get_secret_value(SecretId='prod/db/password')
db_password = response['SecretString']
```

## Network Security

### 1. Network Segmentation

```bash
# Create networks
docker network create frontend
docker network create backend  # Internal only
docker network create backend --internal

# Run services on specific networks
docker run --network=frontend web-app
docker run --network=backend database
docker run --network=frontend --network=backend api
```

**Docker Compose:**
```yaml
version: '3.8'

services:
  web:
    networks:
      - frontend
      
  api:
    networks:
      - frontend
      - backend
      
  db:
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true  # No external access
```

### 2. Limit Port Exposure

```bash
# Bad: Expose to all interfaces
docker run -p 5432:5432 postgres

# Good: Bind to localhost only
docker run -p 127.0.0.1:5432:5432 postgres

# Best: No port binding, use Docker networks
docker run --network=backend postgres
```

### 3. Enable User Namespace Remapping

```bash
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}

# Restart Docker
sudo systemctl restart docker

# Container root maps to non-root on host
docker run -d nginx
ps aux | grep nginx
# Shows as dockremap user on host
```

### 4. Use Network Policies

**Docker Network Driver Options:**
```bash
docker network create \
  --driver bridge \
  --opt com.docker.network.bridge.enable_ip_masquerade=false \
  --opt com.docker.network.bridge.enable_icc=false \
  isolated-network
```

## Registry Security

### 1. Private Registry Setup

```bash
# Run private registry
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v registry-data:/var/lib/registry \
  registry:2

# Push to private registry
docker tag myapp:latest localhost:5000/myapp:latest
docker push localhost:5000/myapp:latest
```

### 2. Registry with TLS

```bash
# Generate certificates
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout domain.key -x509 -days 365 -out domain.crt

# Run registry with TLS
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

### 3. Registry Authentication

```bash
# Create htpasswd file
docker run --rm --entrypoint htpasswd \
  httpd:2 -Bbn admin password > auth/htpasswd

# Run with authentication
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v $(pwd)/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2

# Login
docker login localhost:5000
```

### 4. Image Scanning in CI/CD

**GitLab CI:**
```yaml
# .gitlab-ci.yml
image_scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
```

**GitHub Actions:**
```yaml
# .github/workflows/scan.yml
name: Scan Image
on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
        
      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: 1
```

## Docker Daemon Security

### 1. Docker Daemon Configuration

```json
// /etc/docker/daemon.json
{
  "icc": false,                    // Disable inter-container communication
  "userns-remap": "default",       // User namespace remapping
  "no-new-privileges": true,       // Prevent privilege escalation
  "live-restore": true,            // Keep containers running on daemon restart
  "userland-proxy": false,         // Use iptables instead
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
```

### 2. Restrict Docker Socket Access

```bash
# Check Docker socket permissions
ls -l /var/run/docker.sock
# srw-rw---- 1 root docker 0 Jan 1 00:00 /var/run/docker.sock

# Never mount Docker socket in untrusted containers
# Bad:
docker run -v /var/run/docker.sock:/var/run/docker.sock alpine

# This gives container full control over host!
```

### 3. Enable Docker Rootless Mode

```bash
# Install rootless Docker
curl -fsSL https://get.docker.com/rootless | sh

# Set environment
export PATH=/home/$USER/bin:$PATH
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock

# Start rootless daemon
systemctl --user start docker

# Enable on boot
systemctl --user enable docker
```

### 4. Use Docker Bench Security

```bash
# Run Docker Bench Security audit
docker run -it --net host --pid host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /var/lib:/var/lib \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/lib/systemd:/usr/lib/systemd \
  -v /etc:/etc --label docker_bench_security \
  docker/docker-bench-security

# Review recommendations
```

## Security Scanning Tools

### 1. Trivy (Comprehensive)

```bash
# Image scan
trivy image nginx:latest

# Filesystem scan
trivy fs /path/to/project

# Kubernetes manifest scan
trivy k8s deployment.yaml

# SBOM generation
trivy image --format cyclonedx nginx:latest
```

### 2. Clair (CoreOS)

```bash
# Run Clair
docker run -d --name clair \
  -p 6060:6060 \
  quay.io/coreos/clair:latest

# Scan with clairctl
clairctl analyze myapp:latest
```

### 3. Anchore

```bash
# Run Anchore Engine
docker-compose -f docker-compose.yaml up -d

# Add image
anchore-cli image add myapp:latest

# Wait for analysis
anchore-cli image wait myapp:latest

# View vulnerabilities
anchore-cli image vuln myapp:latest all
```

### 4. Grype (Anchore's standalone scanner)

```bash
# Install Grype
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh

# Scan image
grype nginx:latest

# Scan directory
grype dir:/path/to/project

# Output as JSON
grype -o json nginx:latest
```

## Compliance and Hardening

### 1. CIS Docker Benchmark

**Key recommendations:**
```bash
# 1. Run Docker in rootless mode
# 2. Enable Content Trust
export DOCKER_CONTENT_TRUST=1

# 3. Restrict network traffic between containers
docker network create --internal backend

# 4. Set default ulimit
# /etc/docker/daemon.json
{
  "default-ulimits": {
    "nofile": {"Hard": 1024, "Soft": 1024}
  }
}

# 5. Enable user namespace support
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}

# 6. Configure TLS on Docker daemon
dockerd --tlsverify \
  --tlscacert=/path/to/ca.pem \
  --tlscert=/path/to/cert.pem \
  --tlskey=/path/to/key.pem
```

### 2. Hardened Dockerfile

```dockerfile
# Use minimal base
FROM gcr.io/distroless/static-debian11

# Or Alpine with no shell
FROM alpine:3.18
RUN apk add --no-cache ca-certificates && \
    rm -rf /bin/sh /bin/ash /bin/bash

# Copy only needed files
COPY --from=builder /app/binary /app/binary

# Create non-root user
FROM alpine:3.18
RUN addgroup -g 1000 app && \
    adduser -D -u 1000 -G app app && \
    rm -rf /bin/sh /bin/ash

# Run as non-root
USER 1000:1000

# Read-only filesystem
# Use docker run --read-only

# Drop all capabilities
# Use docker run --cap-drop=ALL

# No new privileges
# Use docker run --security-opt=no-new-privileges:true

# Resource limits in Kubernetes
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
  requests:
    memory: "256Mi"
    cpu: "250m"
```

### 3. Security Policies with OPA

**Open Policy Agent:**
```rego
# policy.rego
package docker.authz

default allow = false

# Allow specific images
allow {
  input.Image = "myregistry.com/approved/*"
}

# Deny running as root
deny["Container runs as root"] {
  input.User == "root"
}

# Require resource limits
deny["No memory limit set"] {
  not input.HostConfig.Memory
}
```

## Monitoring and Auditing

### 1. Docker Events Monitoring

```bash
# Stream all events
docker events

# Filter specific events
docker events --filter event=start --filter event=stop

# Security-relevant events
docker events --filter event=pull --filter event=create
```

**Send to Syslog:**
```json
// /etc/docker/daemon.json
{
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "udp://syslog-server:514"
  }
}
```

### 2. Falco for Runtime Security

```bash
# Install Falco
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | tee /etc/apt/sources.list.d/falcosecurity.list
apt-get update
apt-get install falco

# Run Falco
sudo falco

# Custom rules
# /etc/falco/falco_rules.local.yaml
- rule: Shell in Container
  desc: Detect shell spawned in container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh)
  output: Shell spawned in container (user=%user.name container=%container.id)
  priority: WARNING
```

### 3. Log Analysis

```bash
# Container logs with metadata
docker logs --details container_name

# Export logs
docker logs container_name > /var/log/containers/app.log

# Integrate with ELK/Loki
# docker-compose.yml
services:
  app:
    logging:
      driver: "loki"
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
```

## Security Checklist

### Image Security
- [ ] Use minimal base images (Alpine/Distroless)
- [ ] Pin specific image versions
- [ ] Scan images for vulnerabilities
- [ ] Sign and verify images
- [ ] Use multi-stage builds
- [ ] Remove unnecessary tools (shell, package managers)
- [ ] Keep images updated

### Container Security
- [ ] Run as non-root user
- [ ] Use read-only filesystem where possible
- [ ] Drop all capabilities, add only necessary ones
- [ ] Enable `no-new-privileges`
- [ ] Set resource limits (CPU, memory, PIDs)
- [ ] Use security profiles (AppArmor, SELinux, Seccomp)
- [ ] Implement health checks

### Secrets Management
- [ ] Never hardcode secrets in images
- [ ] Use Docker secrets or external secret managers
- [ ] Pass secrets at runtime via environment
- [ ] Use BuildKit secret mounts for build-time secrets
- [ ] Rotate secrets regularly
- [ ] Encrypt secrets at rest

### Network Security
- [ ] Segment networks (frontend/backend/internal)
- [ ] Bind ports to localhost only when possible
- [ ] Use custom bridge networks
- [ ] Enable user namespace remapping
- [ ] Disable inter-container communication if not needed

### Registry Security
- [ ] Use private registry
- [ ] Enable TLS/SSL
- [ ] Implement authentication
- [ ] Scan images before push
- [ ] Sign images with Content Trust

### Runtime Security
- [ ] Monitor container events
- [ ] Implement runtime security (Falco)
- [ ] Log all security events
- [ ] Regular security audits
- [ ] Incident response plan

### Compliance
- [ ] Follow CIS Docker Benchmark
- [ ] Regular vulnerability scans
- [ ] Automated security testing in CI/CD
- [ ] Document security policies
- [ ] Security training for team

## Tổng kết

**Docker Security = Defense in Depth**

```
┌─────────────────────────────────┐
│  Every layer adds security      │
├─────────────────────────────────┤
│  1. Secure base images          │
│  2. Non-root users              │
│  3. Minimal privileges          │
│  4. Network isolation           │
│  5. Secrets management          │
│  6. Runtime monitoring          │
│  7. Regular updates & scans     │
└─────────────────────────────────┘
```

**Key principles:**
- **Least Privilege**: Minimal permissions necessary
- **Defense in Depth**: Multiple security layers
- **Continuous Monitoring**: Always watch for threats
- **Regular Audits**: Scan and update frequently
- **Documentation**: Security policies and procedures

**Remember:**
- Security is not one-time, it's continuous
- No silver bullet, use multiple techniques
- Balance security with usability
- Automate security checks in CI/CD
- Stay updated with latest vulnerabilities

Docker security là critical aspect của production deployments!
