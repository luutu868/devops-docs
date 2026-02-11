# Container Management

## Tổng quan về Containers

### Container vs Virtual Machine

```
Virtual Machines                  Containers
┌─────────────────────┐          ┌─────────────────────┐
│   App A   │  App B  │          │  App A  │  App B    │
├───────────┼─────────┤          ├─────────┼───────────┤
│  Bins/Libs│Bins/Libs│          │Bins/Libs│Bins/Libs  │
├───────────┼─────────┤          └─────────┴───────────┘
│ Guest OS  │Guest OS │          ┌─────────────────────┐
├───────────┴─────────┤          │   Container Engine  │
│    Hypervisor       │          ├─────────────────────┤
├─────────────────────┤          │     Host OS         │
│    Host OS          │          ├─────────────────────┤
├─────────────────────┤          │    Infrastructure   │
│   Infrastructure    │          └─────────────────────┘
└─────────────────────┘
  Size: GBs, Boot: mins   Size: MBs, Boot: seconds
```

### Container Lifecycle

```
          docker create              docker start
[Image] ────────────────→ [Created] ────────────→ [Running]
                              │                        │
                              │                        │ docker pause
                              │                        ↓
                              │                   [Paused]
                              │                        │
                              │                        │ docker unpause
                              │                        ↓
                              │                   [Running]
                              │                        │
                              │                        │ docker stop
                              │                        ↓
                              └───────────────────→ [Stopped]
                                                       │
                                   docker rm           │
                              [Removed] ←──────────────┘
```

## Container Operations

### Run Containers

```bash
# Basic run
docker run hello-world

# Interactive mode (-it)
docker run -it ubuntu bash

# Detached mode (-d)
docker run -d nginx

# Name container (--name)
docker run --name my-nginx -d nginx

# Port mapping (-p)
docker run -p 8080:80 nginx
docker run -p 127.0.0.1:8080:80 nginx  # Bind to specific IP

# Environment variables (-e)
docker run -e NODE_ENV=production -e PORT=3000 node-app

# Volume mount (-v)
docker run -v /host/path:/container/path nginx
docker run -v my-volume:/data postgres

# Bind mount (--mount)
docker run --mount type=bind,source=/host/path,target=/app nginx

# Remove after exit (--rm)
docker run --rm -it alpine sh

# Resource limits
docker run --memory="512m" --cpus="1.5" nginx

# Restart policy
docker run --restart=unless-stopped nginx
```

**Run options chi tiết:**
```bash
docker run \
  --name myapp \                    # Container name
  -d \                               # Detached mode
  -p 8080:80 \                      # Port mapping
  -e DB_HOST=localhost \            # Environment
  -v /data:/app/data \              # Volume
  --restart=always \                # Auto-restart
  --memory="1g" \                   # Memory limit
  --cpus="2.0" \                    # CPU limit
  --network=my-network \            # Network
  --health-cmd="curl -f http://localhost/" \
  --health-interval=30s \
  nginx:latest
```

### List Containers

```bash
# Running containers
docker ps
docker container ls

# All containers (including stopped)
docker ps -a
docker ps --all

# Latest created container
docker ps -l

# Filter containers
docker ps --filter "status=running"
docker ps --filter "name=nginx"
docker ps --filter "ancestor=nginx"

# Custom format
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"
docker ps --format "{{.Names}}: {{.Status}}"

# Size information
docker ps -s
```

### Start/Stop/Restart Containers

```bash
# Start stopped container
docker start container_name
docker start container_id

# Start multiple containers
docker start container1 container2

# Stop running container (SIGTERM)
docker stop container_name
docker stop -t 30 container_name  # Wait 30s before SIGKILL

# Force stop (SIGKILL)
docker kill container_name

# Restart container
docker restart container_name

# Pause/Unpause container
docker pause container_name
docker unpause container_name
```

### Remove Containers

```bash
# Remove stopped container
docker rm container_name

# Force remove running container
docker rm -f container_name

# Remove multiple containers
docker rm container1 container2

# Remove all stopped containers
docker container prune
docker container prune -f  # No confirmation

# Remove all containers (dangerous!)
docker rm -f $(docker ps -aq)
```

### Execute Commands in Containers

```bash
# Run command in running container
docker exec container_name ls -la

# Interactive shell
docker exec -it container_name bash
docker exec -it container_name sh  # Alpine

# Run as specific user
docker exec -u root container_name apt-get update

# Run with environment variables
docker exec -e VAR=value container_name env

# Run in specific directory
docker exec -w /app container_name npm install
```

### Attach to Container

```bash
# Attach to running container
docker attach container_name

# Detach without stopping: Ctrl+P, Ctrl+Q

# Attach with no-stdin
docker attach --no-stdin container_name
```

### View Logs

```bash
# View logs
docker logs container_name

# Follow logs (-f)
docker logs -f container_name

# Last N lines
docker logs --tail 100 container_name

# Logs with timestamps
docker logs -t container_name

# Logs since specific time
docker logs --since 2024-01-01T10:00:00 container_name
docker logs --since 1h container_name

# Logs until specific time
docker logs --until 2024-01-01T12:00:00 container_name

# Combine options
docker logs -f --tail 50 --since 10m container_name
```

### Inspect Containers

```bash
# Full container details
docker inspect container_name

# Specific field (Go template)
docker inspect --format='{{.State.Status}}' container_name
docker inspect --format='{{.NetworkSettings.IPAddress}}' container_name
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Multiple containers
docker inspect container1 container2

# Size information
docker inspect --size container_name

# Pretty print
docker inspect container_name | jq '.'
```

### Container Statistics

```bash
# Real-time stats
docker stats

# Specific container
docker stats container_name

# All containers (including stopped)
docker stats --all

# No streaming (one-time)
docker stats --no-stream

# Custom format
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### Copy Files

```bash
# Copy from container to host
docker cp container_name:/path/in/container /host/path

# Copy from host to container
docker cp /host/path container_name:/path/in/container

# Copy entire directory
docker cp container_name:/app /backup

# Follow symlinks
docker cp -L container_name:/app/link /backup
```

## Container Networking

### Network Types

```
┌─────────────────────────────────────────────┐
│              Docker Host                    │
│                                             │
│  ┌──────────┐         ┌──────────┐         │
│  │Container1│         │Container2│         │
│  │  eth0    │         │  eth0    │         │
│  └────┬─────┘         └────┬─────┘         │
│       │                    │                │
│  ┌────┴────────────────────┴─────┐         │
│  │      Bridge Network           │         │
│  │       (docker0)                │         │
│  └───────────┬────────────────────┘         │
│              │                              │
│         ┌────┴────┐                         │
│         │  eth0   │                         │
└─────────┴─────────┴─────────────────────────┘
                  │
           ┌──────┴──────┐
           │  Internet   │
           └─────────────┘
```

### Network Commands

```bash
# List networks
docker network ls

# Create network
docker network create my-network
docker network create --driver bridge my-bridge
docker network create --driver overlay my-overlay  # Swarm

# Inspect network
docker network inspect my-network

# Connect container to network
docker network connect my-network container_name

# Disconnect container from network
docker network disconnect my-network container_name

# Remove network
docker network rm my-network

# Prune unused networks
docker network prune
```

### Network Drivers

**Bridge (Default):**
```bash
# Create bridge network with subnet
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  my-bridge

# Run container on custom bridge
docker run --network=my-bridge nginx
```

**Host:**
```bash
# Use host network (no isolation)
docker run --network=host nginx
# Container uses host's network directly
```

**None:**
```bash
# No networking
docker run --network=none alpine
```

**Container:**
```bash
# Share network with another container
docker run --name container1 -d nginx
docker run --network=container:container1 alpine
```

### Container Communication

```bash
# Create network
docker network create app-network

# Run database
docker run -d \
  --name postgres \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Run app (can connect to postgres by hostname)
docker run -d \
  --name webapp \
  --network app-network \
  -e DB_HOST=postgres \
  myapp

# Test connectivity
docker exec webapp ping postgres
docker exec webapp curl http://postgres:5432
```

### DNS and Service Discovery

```bash
# Containers can resolve each other by name
docker network create mynet
docker run -d --name web --network mynet nginx
docker run --rm --network mynet alpine ping web
# PING web (172.18.0.2): 56 data bytes

# Custom DNS
docker run --dns 8.8.8.8 --dns 8.8.4.4 alpine

# DNS search domain
docker run --dns-search=example.com alpine
```

## Storage and Volumes

### Volume Types

```
┌──────────────────────────────────────┐
│ 1. Named Volume (Docker Managed)    │
│    /var/lib/docker/volumes/          │
│    ├─ my-volume/                     │
│    └─ postgres-data/                 │
├──────────────────────────────────────┤
│ 2. Bind Mount (Host Directory)      │
│    /home/user/data → /app/data       │
├──────────────────────────────────────┤
│ 3. tmpfs Mount (Memory)              │
│    Temporary, not persisted          │
└──────────────────────────────────────┘
```

### Volume Commands

```bash
# Create volume
docker volume create my-volume

# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-volume

# Remove volume
docker volume rm my-volume

# Prune unused volumes
docker volume prune

# Create volume with driver options
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/dir \
  my-nfs-volume
```

### Using Volumes

**Named Volume:**
```bash
# Create and use volume
docker run -v my-volume:/data postgres

# Multiple containers can share volume
docker run -v my-volume:/app/data -d app1
docker run -v my-volume:/app/data -d app2
```

**Bind Mount:**
```bash
# Bind host directory to container
docker run -v /host/path:/container/path nginx

# Read-only bind mount
docker run -v /host/path:/container/path:ro nginx

# Mount current directory
docker run -v $(pwd):/app node:18 npm install
```

**Mount Syntax (Preferred):**
```bash
# Volume mount
docker run --mount source=my-volume,target=/data postgres

# Bind mount
docker run --mount type=bind,source=/host/path,target=/app nginx

# Read-only
docker run --mount type=bind,source=/host/path,target=/app,readonly nginx

# tmpfs mount
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m nginx
```

### Backup and Restore Volumes

**Backup:**
```bash
# Backup volume to tar file
docker run --rm \
  -v my-volume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz -C /data .

# Backup with timestamp
docker run --rm \
  -v postgres-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .
```

**Restore:**
```bash
# Restore from backup
docker run --rm \
  -v my-volume:/data \
  -v $(pwd):/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/backup.tar.gz -C /data"
```

### Volume Plugins

```bash
# Install volume plugin
docker plugin install vieux/sshfs

# Create volume with plugin
docker volume create \
  --driver vieux/sshfs \
  -o sshcmd=user@host:/path \
  -o password=secret \
  ssh-volume

# Use plugin volume
docker run -v ssh-volume:/data alpine
```

## Resource Management

### CPU Limits

```bash
# Limit CPUs (0.5 = 50% of one CPU)
docker run --cpus="1.5" nginx

# CPU shares (relative weight)
docker run --cpu-shares=512 nginx  # Default 1024

# CPU period and quota
docker run --cpu-period=100000 --cpu-quota=50000 nginx  # 50%

# Specific CPU cores
docker run --cpuset-cpus="0,1" nginx  # Use CPU 0 and 1
```

### Memory Limits

```bash
# Memory limit
docker run --memory="512m" nginx
docker run -m 1g nginx

# Memory + Swap limit
docker run --memory="512m" --memory-swap="1g" nginx

# Memory reservation (soft limit)
docker run --memory="1g" --memory-reservation="512m" nginx

# Disable OOM killer
docker run --memory="512m" --oom-kill-disable nginx

# Memory swappiness (0-100)
docker run --memory="1g" --memory-swappiness=0 nginx
```

### I/O Limits

```bash
# Block I/O weight (10-1000)
docker run --blkio-weight=500 nginx

# Read/Write rate limit
docker run \
  --device-read-bps /dev/sda:1mb \
  --device-write-bps /dev/sda:1mb \
  nginx

# Read/Write IOPS limit
docker run \
  --device-read-iops /dev/sda:1000 \
  --device-write-iops /dev/sda:1000 \
  nginx
```

### PID Limits

```bash
# Limit number of processes
docker run --pids-limit=100 nginx

# Update running container
docker update --pids-limit=200 container_name
```

### Update Resource Limits

```bash
# Update container resources
docker update --cpus="2" --memory="1g" container_name

# Update multiple containers
docker update --restart=always container1 container2

# Verify changes
docker inspect container_name | grep -A 10 "HostConfig"
```

## Container Health Checks

### Built-in Health Checks

```bash
# Run with health check
docker run -d \
  --name web \
  --health-cmd="curl -f http://localhost/ || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  nginx

# Check health status
docker ps
# STATUS shows "healthy" or "unhealthy"

# Inspect health
docker inspect --format='{{.State.Health.Status}}' web
docker inspect web | jq '.[0].State.Health'
```

### Health Check in Dockerfile

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

### Custom Health Check Scripts

```bash
#!/bin/bash
# health-check.sh

# Check if app responds
if curl -f http://localhost:8080/health; then
    exit 0  # Healthy
else
    exit 1  # Unhealthy
fi
```

## Restart Policies

```bash
# No restart (default)
docker run --restart=no nginx

# Always restart
docker run --restart=always nginx

# Restart on failure
docker run --restart=on-failure nginx
docker run --restart=on-failure:5 nginx  # Max 5 retries

# Restart unless stopped manually
docker run --restart=unless-stopped nginx

# Update restart policy
docker update --restart=always container_name
```

**Policy comparison:**

| Policy | Behavior |
|--------|----------|
| `no` | Never restart |
| `always` | Always restart, even after host reboot |
| `on-failure` | Restart only if exit code != 0 |
| `unless-stopped` | Like always, but not after manual stop |

## Container Monitoring

### View Processes

```bash
# List processes in container
docker top container_name

# With custom format
docker top container_name aux

# Find specific process
docker top container_name | grep nginx
```

### Events

```bash
# Stream events
docker events

# Events from specific container
docker events --filter container=my-container

# Events with type filter
docker events --filter event=start
docker events --filter event=stop

# Events since time
docker events --since '2024-01-01T00:00:00'
docker events --since 1h

# Format events
docker events --format '{{.Time}}: {{.Action}} - {{.Actor.Attributes.name}}'
```

### Port Mapping

```bash
# List port mappings
docker port container_name

# Specific port
docker port container_name 80

# List all with ps
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

## Advanced Operations

### Commit Container to Image

```bash
# Create image from container
docker commit container_name my-image:v1

# With message and author
docker commit -m "Added config" -a "DevOps Team" container_name my-image:v1

# Pause container during commit
docker commit --pause=true container_name my-image:v1
```

### Export/Import Containers

```bash
# Export container filesystem
docker export container_name > container.tar
docker export container_name -o container.tar

# Import as image
cat container.tar | docker import - my-image:latest
docker import container.tar my-image:latest
```

### Save/Load Images

```bash
# Save image to tar
docker save my-image:latest > image.tar
docker save -o image.tar my-image:latest

# Save multiple images
docker save -o images.tar image1 image2

# Load image from tar
docker load < image.tar
docker load -i image.tar
```

### Container Diff

```bash
# Show filesystem changes
docker diff container_name

# Output:
# A /file.txt     (Added)
# C /etc/config   (Changed)
# D /tmp/old      (Deleted)
```

### Wait for Container

```bash
# Wait for container to stop
docker wait container_name

# Returns exit code
EXIT_CODE=$(docker wait container_name)
echo "Container exited with code: $EXIT_CODE"
```

## Practical Examples

### Run PostgreSQL

```bash
docker run -d \
  --name postgres \
  --restart=always \
  -p 5432:5432 \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# Connect to database
docker exec -it postgres psql -U admin -d mydb
```

### Run Redis

```bash
docker run -d \
  --name redis \
  --restart=always \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7-alpine redis-server --appendonly yes

# Redis CLI
docker exec -it redis redis-cli
```

### Run Node.js Application

```bash
docker run -d \
  --name myapp \
  --restart=unless-stopped \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e DB_HOST=postgres \
  -e REDIS_HOST=redis \
  -v $(pwd)/logs:/app/logs \
  --network app-network \
  myapp:latest

# View logs
docker logs -f myapp

# Check health
docker exec myapp wget -O- http://localhost:3000/health
```

### Development Container

```bash
docker run -it --rm \
  --name dev \
  -v $(pwd):/workspace \
  -w /workspace \
  -p 8080:8080 \
  node:18-alpine \
  sh

# Inside container
npm install
npm run dev
```

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker logs container_name
docker logs --tail 100 container_name

# Inspect exit code
docker inspect --format='{{.State.ExitCode}}' container_name

# Check events
docker events --filter container=container_name
```

### Container Keeps Restarting

```bash
# Check restart count
docker inspect --format='{{.RestartCount}}' container_name

# View logs around restarts
docker logs --since 1h container_name

# Disable auto-restart temporarily
docker update --restart=no container_name
```

### Can't Connect to Container

```bash
# Check if container is running
docker ps | grep container_name

# Check port mappings
docker port container_name

# Check network
docker network inspect bridge

# Test from host
curl localhost:8080

# Test from another container
docker run --rm --network container:container_name alpine wget -O- localhost
```

### High Resource Usage

```bash
# Check stats
docker stats container_name

# Check processes
docker top container_name

# Exec into container
docker exec -it container_name sh
top  # or htop if available
```

### Clean Up

```bash
# Remove all stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# Remove everything unused
docker system prune

# Remove including volumes
docker system prune --volumes

# Force (no confirmation)
docker system prune -f --all --volumes
```

## Best Practices

### 1. Container Naming
```bash
# Use descriptive names
docker run --name web-frontend nginx
docker run --name api-backend node-app
docker run --name db-postgres postgres
```

### 2. Resource Limits
```bash
# Always set limits in production
docker run \
  --memory="1g" \
  --memory-reservation="512m" \
  --cpus="2" \
  --pids-limit=100 \
  myapp
```

### 3. Health Checks
```bash
# Include health checks
docker run \
  --health-cmd="curl -f http://localhost/health || exit 1" \
  --health-interval=30s \
  myapp
```

### 4. Restart Policies
```bash
# Use unless-stopped for most services
docker run --restart=unless-stopped myapp
```

### 5. Logging
```bash
# Configure log driver
docker run \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp
```

### 6. Security
```bash
# Run as non-root
docker run --user 1000:1000 myapp

# Read-only filesystem
docker run --read-only myapp

# Drop capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp
```

## Tổng kết

**Container Management essentials:**
-  **Lifecycle**: create → start → run → stop → remove
- **Networking**: Bridge, host, overlay, custom networks
- **Storage**: Volumes, bind mounts, tmpfs
- **Monitoring**: logs, stats, events, inspect
- **Resources**: CPU, memory, I/O limits
- **Security**: Non-root, read-only, capabilities

**Production checklist:**
- Set resource limits
- Configure restart policies
- Implement health checks
- Use named volumes
- Set up monitoring
- Configure logging
- Network isolation
- Regular cleanup

Container management là core skill cho DevOps!
