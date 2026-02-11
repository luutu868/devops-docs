# Docker Fundamentals

## Docker Là Gì?

**Docker** là một platform để develop, ship và run applications trong containers. Container cho phép pack ứng dụng với tất cả dependencies vào một unit chuẩn hóa.

### **Docker vs Virtual Machines**

```
┌──────────────────────┐    ┌──────────────────────┐
│   Virtual Machine    │    │      Container       │
├──────────────────────┤    ├──────────────────────┤
│  App A  │  App B     │    │  App A  │  App B     │
│  Bins   │  Bins      │    │  Bins   │  Bins      │
│  Libs   │  Libs      │    │  Libs   │  Libs      │
├─────────┴────────────┤    ├──────────────────────┤
│ Guest OS │ Guest OS  │    │   Docker Engine      │
├──────────────────────┤    ├──────────────────────┤
│    Hypervisor        │    │     Host OS          │
├──────────────────────┤    ├──────────────────────┤
│     Host OS          │    │     Hardware         │
├──────────────────────┤    └──────────────────────┘
│     Hardware         │    
└──────────────────────┘    

VM: GBs, Minutes to start    Container: MBs, Seconds to start
```

### **Lợi Ích của Docker**

**Lightweight**: Containers share OS kernel
**Fast**: Start trong giây
**Portable**: "Works on my machine" → Works everywhere
**Consistent**: Same environment Dev/Staging/Prod
**Efficient**: Chạy nhiều containers trên 1 host
**Scalable**: Easy to scale up/down
**Version Control**: Image versioning

## Docker Architecture

```
┌─────────────────────────────────────────────┐
│           Docker Client                     │
│  (docker commands: run, build, push)        │
└──────────────────┬──────────────────────────┘
                   │ Docker API (REST)
┌──────────────────▼──────────────────────────┐
│          Docker Daemon (dockerd)            │
│  - Manages containers, images, networks     │
│  - Builds images from Dockerfile            │
└─────────────────┬───────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
┌───▼────┐  ┌────▼────┐  ┌────▼────┐
│Container│  │Container│  │Container│
│   A     │  │   B     │  │   C     │
└─────────┘  └─────────┘  └─────────┘

┌─────────────────────────────────────────────┐
│          Docker Registry                    │
│  (Docker Hub, private registry)             │
│  - Stores Docker images                     │
└─────────────────────────────────────────────┘
```

## Core Concepts

### **1. Images**

- Blueprint/template for containers
- Read-only
- Layered file system
- Can be versioned/tagged

### **2. Containers**

- Running instance of an image
- Isolated process
- Has own filesystem, network, process tree
- Ephemeral by default

### **3. Dockerfile**

- Text file with instructions to build image
- Automated image creation

### **4. Registry**

- Storage for Docker images
- Docker Hub, AWS ECR, Google GCR, etc.

##  Installation

### **Ubuntu/Debian**

```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Update and install dependencies
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release

# Add Docker GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verify installation
docker --version
sudo docker run hello-world

# Add user to docker group (no need sudo)
sudo usermod -aG docker $USER
newgrp docker  # Or logout/login

# Test
docker run hello-world
```

## Basic Commands

### **Container Operations**

```bash
# Run container
docker run nginx

# Run in detached mode
docker run -d nginx

# Run with name
docker run -d --name mynginx nginx

# Run with port mapping
docker run -d -p 8080:80 nginx

# Run with environment variable
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql

# Run with volume
docker run -d -v /host/path:/container/path nginx

# Run interactive shell
docker run -it ubuntu bash

# List running containers
docker ps

# List all containers
docker ps -a

# Stop container
docker stop mynginx
docker stop <container-id>

# Start stopped container
docker start mynginx

# Restart container
docker restart mynginx

# Remove container
docker rm mynginx

# Remove running container
docker rm -f mynginx

# View container logs
docker logs mynginx
docker logs -f mynginx  # Follow logs

# Execute command in running container
docker exec mynginx ls /usr/share/nginx/html
docker exec -it mynginx bash  # Interactive shell

# View container details
docker inspect mynginx

# View container stats
docker stats
docker stats mynginx

# View container processes
docker top mynginx

# Copy files to/from container
docker cp file.txt mynginx:/path/
docker cp mynginx:/path/file.txt ./
```

### **Image Operations**

```bash
# List images
docker images
docker image ls

# Pull image from registry
docker pull nginx
docker pull nginx:1.21-alpine
docker pull ubuntu:22.04

# Search images on Docker Hub
docker search nginx

# Build image from Dockerfile
docker build -t myapp:1.0 .
docker build -t myapp:latest --no-cache .

# Tag image
docker tag myapp:1.0 myrepo/myapp:1.0

# Push image to registry
docker push myrepo/myapp:1.0

# Remove image
docker rmi nginx
docker rmi -f nginx  # Force remove

# Remove unused images
docker image prune

# Remove all unused images
docker image prune -a

# View image history/layers
docker history nginx

# Save image to tar file
docker save nginx > nginx.tar
docker save -o nginx.tar nginx

# Load image from tar file
docker load < nginx.tar
docker load -i nginx.tar

# Export container to tar
docker export mynginx > mynginx.tar

# Import from tar
docker import mynginx.tar myimage:latest
```

### **System Commands**

```bash
# View Docker info
docker info

# View Docker version
docker version

# Disk usage
docker system df

# Clean up
docker system prune  # Remove stopped containers, unused networks, dangling images
docker system prune -a  # Also remove unused images
docker system prune -a --volumes  # Also remove unused volumes

# Real-time events
docker events
```

## Dockerfile Basics

### **Simple Example**

```dockerfile
# Use base image
FROM ubuntu:22.04

# Set working directory
WORKDIR /app

# Copy files
COPY . /app

# Run commands
RUN apt-get update && apt-get install -y python3

# Expose port
EXPOSE 8000

# Set default command
CMD ["python3", "app.py"]
```

### **Dockerfile Instructions**

| Instruction | Purpose | Example |
|-------------|---------|---------|
| `FROM` | Base image | `FROM python:3.9` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files | `COPY . /app` |
| `ADD` | Copy + extract archives | `ADD file.tar.gz /app` |
| `RUN` | Execute commands | `RUN npm install` |
| `CMD` | Default command | `CMD ["python", "app.py"]` |
| `ENTRYPOINT` | Main command | `ENTRYPOINT ["nginx"]` |
| `ENV` | Environment variables | `ENV PORT=8080` |
| `EXPOSE` | Document port | `EXPOSE 80` |
| `VOLUME` | Mount point | `VOLUME /data` |
| `USER` | Switch user | `USER appuser` |
| `ARG` | Build argument | `ARG VERSION=1.0` |

### **Real-World Examples**

#### **Node.js Application**

```dockerfile
FROM node:16-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build app (if needed)
RUN npm run build

EXPOSE 3000

# Run as non-root user
USER node

CMD ["node", "server.js"]
```

#### **Python Application**

```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

EXPOSE 8000

# Run as non-root
RUN useradd -m appuser
USER appuser

CMD ["python", "app.py"]
```

#### **Multi-stage Build (Go)**

```dockerfile
# Build stage
FROM golang:1.19 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Production stage
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
EXPOSE 8080
CMD ["./main"]
```

## 🎓 Common Use Cases

### **1. Run Database**

```bash
# MySQL
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=mydb \
  -p 3306:3306 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

# PostgreSQL
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:14

# Redis
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:alpine
```

### **2. Run Web Server**

```bash
# Nginx
docker run -d \
  --name nginx \
  -p 80:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:alpine

# Apache
docker run -d \
  --name apache \
  -p 8080:80 \
  -v $(pwd)/html:/usr/local/apache2/htdocs \
  httpd:alpine
```

### **3. Development Environment**

```bash
# Run and get shell
docker run -it --rm \
  -v $(pwd):/workspace \
  -w /workspace \
  python:3.9 \
  bash

# Development with hot reload
docker run -d \
  --name dev-server \
  -p 3000:3000 \
  -v $(pwd):/app \
  -w /app \
  node:16-alpine \
  npm run dev
```

## Best Practices

### **Security**

Use official images
Scan for vulnerabilities
Run as non-root user
Don't store secrets in images
Use specific image tags (not :latest)
Multi-stage builds
Minimal base images (alpine)

### **Performance**

Minimize layers
Order Dockerfile instructions properly
Use .dockerignore
Leverage build cache
Multi-stage builds for smaller images

### **.dockerignore Example**

```
node_modules/
npm-debug.log
.git/
.gitignore
README.md
.env
.env.local
dist/
coverage/
*.log
```

## 🚨 Troubleshooting

```bash
# Container won't start
docker logs container_name

# Check container processes
docker top container_name

# Inspect container
docker inspect container_name

# Get container IP
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Debug running container
docker exec -it container_name bash

# View resource usage
docker stats container_name

# Clean up disk space
docker system prune -a --volumes
```

## Quick Reference

```bash
# Common commands
docker run -d -p 8080:80 --name web nginx
docker ps
docker logs web
docker exec -it web bash
docker stop web
docker rm web
docker images
docker rmi nginx
docker build -t myapp .
docker push myuser/myapp
```

---

**Tiếp theo**: [Docker Images →](../images/dockerfile-best-practices.md)
