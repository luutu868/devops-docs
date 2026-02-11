# Docker Compose

## Tổng quan Docker Compose

### Docker Compose là gì?

Docker Compose là tool để define và run multi-container Docker applications. Sử dụng YAML file để configure services.

```
Without Compose               With Compose
┌──────────────────┐         ┌──────────────────┐
│ docker run ...   │         │ docker-compose.yml│
│ docker run ...   │         │  - web           │
│ docker run ...   │    →    │  - database      │
│ docker network...│         │  - redis         │
│ docker volume... │         │  - nginx         │
└──────────────────┘         └──────────────────┘
  Many commands                One file, one command
```

### Installation

```bash
# Linux (standalone)
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Or via pip
pip install docker-compose

# Verify installation
docker-compose --version
docker compose version  # V2 syntax
```

## Basic Compose File

### Simple Example

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    environment:
      - NGINX_PORT=80

  database:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

**Run:**
```bash
docker-compose up
docker-compose up -d  # Detached mode
```

### Compose File Structure

```yaml
version: '3.8'              # Compose file version

services:                   # Container definitions
  service_name:
    # Build or image config
    # Network config
    # Volume mounts
    # Environment variables
    # Dependencies
    # Resource limits

networks:                   # Custom networks
  network_name:
    driver: bridge

volumes:                    # Named volumes
  volume_name:
    driver: local

configs:                    # Configuration files (v3.3+)
  config_name:
    file: ./config.txt

secrets:                    # Sensitive data (v3.1+)
  secret_name:
    file: ./secret.txt
```

## Service Configuration

### Image vs Build

```yaml
services:
  # Using pre-built image
  nginx:
    image: nginx:1.24-alpine
    
  # Building from Dockerfile
  app:
    build: ./app
    
  # Build with context and Dockerfile
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.prod
      args:
        - NODE_ENV=production
        - VERSION=1.0.0
      target: production  # Multi-stage build target
```

### Ports and Expose

```yaml
services:
  web:
    # Publish ports (HOST:CONTAINER)
    ports:
      - "8080:80"          # Bind to all interfaces
      - "127.0.0.1:8080:80"  # Bind to localhost only
      - "3000-3005:3000"   # Range mapping
      
    # Expose ports (only to linked services)
    expose:
      - "3000"
      - "8000"
```

### Environment Variables

```yaml
services:
  app:
    # Inline environment variables
    environment:
      - NODE_ENV=production
      - DEBUG=false
      - PORT=3000
      
    # Or as object
    environment:
      NODE_ENV: production
      DEBUG: "false"
      
    # From .env file
    env_file:
      - .env
      - .env.production
```

**.env file:**
```bash
# .env
NODE_ENV=production
DB_HOST=postgres
DB_PORT=5432
DB_NAME=myapp
```

### Volumes

```yaml
services:
  db:
    # Named volume
    volumes:
      - db-data:/var/lib/postgresql/data
      
  web:
    # Bind mount (relative path)
    volumes:
      - ./html:/usr/share/nginx/html
      
  app:
    # Bind mount (absolute path)
    volumes:
      - /data:/app/data
      
    # Read-only volume
    volumes:
      - ./config:/app/config:ro
      
    # Volume with options
    volumes:
      - type: bind
        source: ./app
        target: /app
        read_only: false
        
volumes:
  db-data:  # Define named volume
```

### Networks

```yaml
services:
  web:
    # Default network
    networks:
      - frontend
      
  api:
    # Multiple networks
    networks:
      - frontend
      - backend
      
  db:
    # Network with alias
    networks:
      backend:
        aliases:
          - database
          - postgres

networks:
  frontend:
    driver: bridge
    
  backend:
    driver: bridge
    internal: true  # No external access
```

### Dependencies

```yaml
services:
  web:
    depends_on:
      - db
      - redis
      
  api:
    # Wait for healthy status
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
        
  db:
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Resource Limits

```yaml
services:
  app:
    # CPU and Memory limits
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G
        reservations:
          cpus: '1.0'
          memory: 512M
          
    # Replicas (Swarm mode)
    deploy:
      replicas: 3
      
    # Restart policy
    restart: unless-stopped  # no, always, on-failure, unless-stopped
```

### Commands and Entrypoints

```yaml
services:
  app:
    # Override CMD
    command: npm start
    
    # Array form (preferred)
    command: ["npm", "run", "dev"]
    
    # Override entrypoint
    entrypoint: /app/entrypoint.sh
    
    # Array form
    entrypoint: ["/bin/sh", "-c"]
    command: ["npm start"]
```

### Health Checks

```yaml
services:
  web:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
      
  api:
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
      interval: 30s
```

## Docker Compose Commands

### Basic Commands

```bash
# Start services
docker-compose up
docker-compose up -d           # Detached mode
docker-compose up --build      # Rebuild images
docker-compose up service1     # Start specific service

# Stop services
docker-compose stop
docker-compose stop service1   # Stop specific service

# Stop and remove containers
docker-compose down
docker-compose down -v         # Remove volumes too
docker-compose down --rmi all  # Remove images too

# Restart services
docker-compose restart
docker-compose restart service1
```

### Service Management

```bash
# List containers
docker-compose ps
docker-compose ps -a           # All (including stopped)

# View logs
docker-compose logs
docker-compose logs -f         # Follow logs
docker-compose logs service1   # Specific service
docker-compose logs --tail=100 service1

# Execute command in service
docker-compose exec service1 bash
docker-compose exec service1 sh
docker-compose exec -u root service1 bash

# Run one-off command
docker-compose run service1 npm test
docker-compose run --rm service1 python manage.py migrate
```

### Build and Images

```bash
# Build images
docker-compose build
docker-compose build service1
docker-compose build --no-cache

# Pull images
docker-compose pull

# Push images (to registry)
docker-compose push
```

### Scaling

```bash
# Scale specific service
docker-compose up -d --scale web=3 --scale worker=2

# Scale in compose file (V3+)
# Use deploy: replicas: 3
```

### Validation

```bash
# Validate compose file
docker-compose config

# Validate and view resolved config
docker-compose config --resolve-image-digests

# Show volumes
docker-compose config --volumes

# Show services
docker-compose config --services
```

## Advanced Compose Examples

### Full-Stack Application

```yaml
version: '3.8'

services:
  # Frontend
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8000
    depends_on:
      - api
    networks:
      - frontend
      
  # API Backend
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@postgres:5432/db
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - frontend
      - backend
    restart: unless-stopped
    
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=db
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
      
  # Redis Cache
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend
      
  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - frontend
      - api
    networks:
      - frontend

networks:
  frontend:
  backend:
    internal: true

volumes:
  postgres-data:
  redis-data:
```

### Development Environment

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      # Mount source code
      - ./src:/app/src
      - ./public:/app/public
      # Anonymous volume for node_modules
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - CHOKIDAR_USEPOLLING=true  # Hot reload
    command: npm run dev
    depends_on:
      - db
      - redis
      
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=devpass
    volumes:
      - dev-db:/var/lib/postgresql/data
    ports:
      - "5432:5432"  # Expose for local access
      
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
      
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db

volumes:
  dev-db:
```

### Microservices Architecture

```yaml
version: '3.8'

services:
  # API Gateway
  gateway:
    build: ./gateway
    ports:
      - "80:80"
    depends_on:
      - user-service
      - product-service
      - order-service
    networks:
      - public
      
  # User Service
  user-service:
    build: ./services/user
    environment:
      - DB_HOST=user-db
    depends_on:
      - user-db
    networks:
      - public
      - user-network
    deploy:
      replicas: 2
      
  user-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=users
    volumes:
      - user-db-data:/var/lib/postgresql/data
    networks:
      - user-network
      
  # Product Service
  product-service:
    build: ./services/product
    environment:
      - DB_HOST=product-db
    depends_on:
      - product-db
    networks:
      - public
      - product-network
      
  product-db:
    image: mongo:6
    volumes:
      - product-db-data:/data/db
    networks:
      - product-network
      
  # Order Service
  order-service:
    build: ./services/order
    environment:
      - DB_HOST=order-db
      - RABBITMQ_HOST=rabbitmq
    depends_on:
      - order-db
      - rabbitmq
    networks:
      - public
      - order-network
      - messaging
      
  order-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=orders
    volumes:
      - order-db-data:/var/lib/postgresql/data
    networks:
      - order-network
      
  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"  # Management UI
    networks:
      - messaging
      
  # Monitoring
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
      
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus

networks:
  public:
  user-network:
    internal: true
  product-network:
    internal: true
  order-network:
    internal: true
  messaging:

volumes:
  user-db-data:
  product-db-data:
  order-db-data:
  prometheus-data:
  grafana-data:
```

### CI/CD Testing Environment

```yaml
version: '3.8'

services:
  # Unit Tests
  test-unit:
    build:
      context: .
      target: test
    command: npm run test:unit
    volumes:
      - ./coverage:/app/coverage
      
  # Integration Tests
  test-integration:
    build:
      context: .
      target: test
    command: npm run test:integration
    environment:
      - DB_HOST=test-db
      - REDIS_HOST=test-redis
    depends_on:
      - test-db
      - test-redis
      
  test-db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=testpass
    tmpfs:
      - /var/lib/postgresql/data  # In-memory for speed
      
  test-redis:
    image: redis:7-alpine
    
  # E2E Tests
  test-e2e:
    build:
      context: .
      target: test
    command: npm run test:e2e
    depends_on:
      - app
      - selenium
      
  app:
    build: .
    environment:
      - NODE_ENV=test
    depends_on:
      - test-db
      
  selenium:
    image: selenium/standalone-chrome
    shm_size: 2gb
```

## Environment Variables

### Multiple Environments

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  app:
    build: .
    env_file:
      - .env
      - .env.${ENV:-dev}  # .env.dev or .env.prod
```

**.env.dev:**
```bash
NODE_ENV=development
DEBUG=true
LOG_LEVEL=debug
DATABASE_URL=postgresql://localhost:5432/dev
```

**.env.prod:**
```bash
NODE_ENV=production
DEBUG=false
LOG_LEVEL=info
DATABASE_URL=postgresql://prod-db:5432/production
```

**Usage:**
```bash
# Development
docker-compose up

# Production
ENV=prod docker-compose up
```

### Override Files

**docker-compose.yml** (base):
```yaml
version: '3.8'

services:
  app:
    image: myapp:latest
    environment:
      - NODE_ENV=production
```

**docker-compose.override.yml** (auto-loaded):
```yaml
version: '3.8'

services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - NODE_ENV=development
    command: npm run dev
```

**docker-compose.prod.yml**:
```yaml
version: '3.8'

services:
  app:
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
    restart: always
```

**Usage:**
```bash
# Development (uses override automatically)
docker-compose up

# Production (explicit prod file)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

## Best Practices

### 1. Use Version Control

```bash
# .gitignore
.env
.env.local
*.log
docker-compose.override.yml  # Personal dev overrides
```

### 2. Health Checks

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### 3. Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '1'
          memory: 512M
```

### 4. Logging Configuration

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 5. Security

```yaml
services:
  app:
    # Run as non-root
    user: "1000:1000"
    
    # Read-only root filesystem
    read_only: true
    
    # Drop capabilities
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      
    # Security options
    security_opt:
      - no-new-privileges:true
```

### 6. Use Named Volumes

```yaml
# Bad: Anonymous volumes
services:
  db:
    volumes:
      - /var/lib/postgresql/data

# Good: Named volumes
services:
  db:
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

### 7. Network Segmentation

```yaml
networks:
  frontend:
    # Web-facing services
  backend:
    # Internal services only
    internal: true
```

## Troubleshooting

### Check Logs

```bash
# All services
docker-compose logs

# Follow logs
docker-compose logs -f

# Specific service
docker-compose logs api

# Last N lines
docker-compose logs --tail=100 api

# Timestamps
docker-compose logs -t
```

### Debug Services

```bash
# Check service status
docker-compose ps

# Inspect specific service
docker-compose exec service_name env
docker-compose exec service_name cat /etc/hosts

# Check networks
docker network ls
docker network inspect project_network

# Check volumes
docker volume ls
docker volume inspect project_volume
```

### Rebuild Everything

```bash
# Stop and remove all
docker-compose down -v --rmi all

# Rebuild from scratch
docker-compose build --no-cache
docker-compose up -d
```

### Port Conflicts

```bash
# Check what's using port
sudo lsof -i :8080
sudo netstat -tulpn | grep 8080

# Use different port
docker-compose up -d --scale web=1 -p 8081:80
```

## Makefile Integration

```makefile
# Makefile
.PHONY: help build up down restart logs shell test clean

help:
@echo "Available commands:"
@echo "  make build   - Build images"
@echo "  make up      - Start services"
@echo "  make down    - Stop services"
@echo "  make restart - Restart services"
@echo "  make logs    - Show logs"
@echo "  make shell   - Open shell in app"
@echo "  make test    - Run tests"
@echo "  make clean   - Clean everything"

build:
docker-compose build

up:
docker-compose up -d

down:
docker-compose down

restart:
docker-compose restart

logs:
docker-compose logs -f

shell:
docker-compose exec app bash

test:
docker-compose run --rm app npm test

clean:
docker-compose down -v --rmi all
docker system prune -f
```

**Usage:**
```bash
make build
make up
make logs
make test
make clean
```

## Tổng kết

**Docker Compose advantages:**
- **Infrastructure as Code**: Define entire stack in YAML
-  **Simplicity**: One command để start/stop all services
- **Reproducibility**: Same environment everywhere
- **Networking**: Automatic service discovery
- **Volume Management**: Persistent data handling
- **Development**: Perfect for local dev environments

**Key concepts:**
- Services: Containers definitions
- Networks: Service communication
- Volumes: Data persistence
- Environment: Configuration management
- Dependencies: Service orchestration

**Common use cases:**
- Local development environments
-Full-stack applications
- Microservices architecture
- Testing environments
- CI/CD pipelines
- Multi-container demos

Docker Compose là essential tool cho modern DevOps workflow!
