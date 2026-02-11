# Linux Performance Tuning

## Tổng quan Performance Tuning

### Performance Metrics Quan Trọng

```
┌─────────────────────────────────────────┐
│         Performance Layers              │
├─────────────────────────────────────────┤
│  Application     │ Code Optimization    │
│  Runtime         │ JVM, Python, Node.js │
│  OS Kernel       │ Scheduler, Memory    │
│  Hardware        │ CPU, RAM, Disk, NIC  │
└─────────────────────────────────────────┘
```

**Golden Signals:**
1. **Latency** - Thời gian response
2. **Throughput** - Requests/transactions per second
3. **Errors** - Error rate
4. **Saturation** - Resource utilization

### USE Method (Utilization, Saturation, Errors)

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| CPU | %CPU usage | Run queue length | CPU errors |
| Memory | Used/Total | Swap usage | OOM kills |
| Disk I/O | %Busy | Queue depth | I/O errors |
| Network | Bandwidth % | Socket buffer full | Packet drops |

## CPU Performance

### Check CPU Usage

```bash
# Overall CPU usage
top
htop
mpstat 1

# Per-CPU usage
mpstat -P ALL 1

# CPU details
lscpu
cat /proc/cpuinfo

# Load average
uptime
cat /proc/loadavg

# Context switches
vmstat 1
pidstat -w 1
```

**Hiểu Load Average:**
```bash
$ uptime
15:30:00 up 10 days, 3:15, 2 users, load average: 2.5, 1.8, 1.2
                                                   │    │    │
                                                   │    │    └─ 15 min
                                                   │    └────── 5 min
                                                   └─────────── 1 min
```

**Rule of thumb:**
- Load < số CPU cores: OK
- Load = số CPU cores: Saturated
- Load > số CPU cores: Overloaded

### CPU Frequency Scaling

```bash
# Check current CPU governor
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Available governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

# Set performance mode (tất cả cores)
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Set powersave mode
echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Install cpufrequtils
sudo apt install cpufrequtils

# Set governor
sudo cpufreq-set -g performance
```

**Governors:**
- **performance**: Max frequency luôn
- **powersave**: Min frequency
- **ondemand**: Dynamic scaling (default)
- **conservative**: Gradual frequency changes
- **schedutil**: Scheduler-based

### CPU Affinity - Pin Process to Cores

```bash
# Check process CPU affinity
taskset -p 1234

# Set affinity to CPU 0-3
taskset -cp 0-3 1234

# Run command on specific CPUs
taskset -c 0,1 ./myapp

# Sử dụng numactl
numactl --physcpubind=0-3 --membind=0 ./myapp
```

### Disable CPU Features (Security vs Performance)

```bash
# Check vulnerabilities
cat /sys/devices/system/cpu/vulnerabilities/*

# Disable mitigations (BOOT PARAMETER)
# /etc/default/grub
GRUB_CMDLINE_LINUX="mitigations=off"

sudo update-grub
sudo reboot

# Or specific mitigations
GRUB_CMDLINE_LINUX="nospectre_v1 nospectre_v2 nopti"
```

⚠️ **Warning:** Only disable on trusted systems!

## Memory Performance

### Check Memory Usage

```bash
# Memory overview
free -h
free -m -s 1       # Update every second

# Detailed memory
cat /proc/meminfo

# Per-process memory
ps aux --sort=-%mem | head
pmap -x 1234
smem -s uss        # USS = Unique Set Size

# Memory pressure
vmstat 1
vmstat -s

# OOM killer logs
dmesg | grep -i "out of memory"
journalctl -k | grep -i "killed process"
```

### Memory Management

**Hiểu Memory Types:**
```
┌──────────────────────────────────┐
│  Total RAM (16 GB)               │
├──────────────────────────────────┤
│  Used (6 GB)                     │
│  ├─ Apps (4 GB)                  │
│  └─ Buffers/Cache (2 GB)         │ <─ Reclaimable
├──────────────────────────────────┤
│  Free (10 GB)                    │
│  ├─ Actually Free (8 GB)         │
│  └─ Available (10 GB)            │ <─ Free + Reclaimable
└──────────────────────────────────┘
```

### Swap Configuration

```bash
# Check swap
swapon --show
free -h

# Tạo swap file
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Persistent swap
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Swappiness (0-100)
cat /proc/sys/vm/swappiness

# Set swappiness
sudo sysctl vm.swappiness=10
# Persistent
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
```

**Swappiness values:**
- `0`: Avoid swap, only use khi critical
- `10`: Recommended cho servers
- `60`: Default
- `100`: Swap aggressively

### Transparent Huge Pages (THP)

```bash
# Check THP status
cat /sys/kernel/mm/transparent_hugepage/enabled

# Disable THP (recommended cho databases)
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Persistent disable
# /etc/rc.local hoặc systemd service
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### Page Cache Management

```bash
# Clear page cache
sync
echo 1 | sudo tee /proc/sys/vm/drop_caches

# Clear dentries and inodes
echo 2 | sudo tee /proc/sys/vm/drop_caches

# Clear all
echo 3 | sudo tee /proc/sys/vm/drop_caches

# Script để free memory an toàn
#!/bin/bash
used=$(free | awk '/Mem:/ {print $3/$2 * 100}')
if (( $(echo "$used > 90" | bc -l) )); then
    sync
    echo 3 > /proc/sys/vm/drop_caches
fi
```

### Memory Overcommit

```bash
# Check overcommit mode
cat /proc/sys/vm/overcommit_memory

# Set overcommit mode
sudo sysctl vm.overcommit_memory=1
```

**Modes:**
- `0`: Heuristic (default)
- `1`: Always overcommit (no checks)
- `2`: Strict (no overcommit)

## Disk I/O Performance

### Check Disk I/O

```bash
# I/O statistics
iostat -x 1
iostat -xz 1       # Skip zero-activity devices

# Per-process I/O
iotop
iotop -o           # Chỉ hiện active processes
pidstat -d 1

# Disk usage
df -h
df -i              # Inodes

# Directory size
du -sh /var/log/*
du -h --max-depth=1 /var

# Find large files
find / -type f -size +1G -exec ls -lh {} \;
```

### I/O Scheduler

```bash
# Check scheduler
cat /sys/block/sda/queue/scheduler

# Available schedulers
cat /sys/block/sda/queue/scheduler
# Output: noop deadline [cfq] mq-deadline kyber bfq

# Set scheduler
echo deadline | sudo tee /sys/block/sda/queue/scheduler

# Persistent configuration
# /etc/udev/rules.d/60-ioschedulers.rules
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/scheduler}="deadline"
```

**Schedulers:**
- **noop**: First-in-first-out (SSDs)
- **deadline**: Latency-focused (default)
- **cfq**: Fairness-focused
- **mq-deadline**: Multi-queue deadline
- **kyber**: Token-based (modern)
- **bfq**: Budget fair queueing

**Recommendations:**
- SSDs: `noop` or `mq-deadline`
- HDDs: `deadline` or `cfq`
- NVMe: `none` (no scheduler)

### Filesystem Tuning

**ext4 Mount Options:**
```bash
# /etc/fstab
/dev/sda1 / ext4 defaults,noatime,nodiratime,errors=remount-ro 0 1
```

**Options:**
- `noatime`: Không update access time (faster)
- `nodiratime`: Không update directory access time
- `data=writeback`: Performance > durability
- `barrier=0`: Disable write barriers (only with UPS)

**XFS Mount Options:**
```bash
/dev/sdb1 /data xfs defaults,noatime,nodiratime,logbufs=8,logbsize=256k 0 0
```

### Read-ahead Tuning

```bash
# Check read-ahead
blockdev --getra /dev/sda

# Set read-ahead (KB)
blockdev --setra 8192 /dev/sda

# Persistent
# /etc/udev/rules.d/99-readahead.rules
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{bdi/read_ahead_kb}="8192"
```

### Disable Disk Barriers (with UPS)

```bash
# For ext4
mount -o barrier=0 /dev/sda1 /mnt

# For XFS
mount -o nobarrier /dev/sdb1 /data
```

### RAID Performance

```bash
# Check RAID speed
hdparm -Tt /dev/md0

# Set stripe cache size
echo 8192 > /sys/block/md0/md/stripe_cache_size

# Read-ahead for RAID
blockdev --setra 16384 /dev/md0
```

## Network Performance

### Check Network Performance

```bash
# Network statistics
ifconfig
ip -s link

# Real-time monitoring
iftop
iftop -i eth0
nethogs          # Per-process bandwidth

# Socket statistics
ss -s
ss -tulpn

# netstat (legacy)
netstat -s
netstat -i

# Packet drops
ethtool -S eth0 | grep -i drop
```

### Network Settings

```bash
# Check ring buffer size
ethtool -g eth0

# Set ring buffer
ethtool -G eth0 rx 4096 tx 4096

# Network offloading
ethtool -k eth0

# Enable TCP offloading
ethtool -K eth0 tso on gso on gro on
```

### TCP Tuning

**Sysctl TCP Parameters:**
```bash
# /etc/sysctl.conf

# Increase TCP buffer sizes
net.core.rmem_max = 134217728           # 128 MB
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864  # min default max
net.ipv4.tcp_wmem = 4096 65536 67108864

# Increase backlog
net.core.netdev_max_backlog = 5000
net.core.somaxconn = 1024

# TCP tuning
net.ipv4.tcp_congestion_control = bbr    # Better algorithm
net.ipv4.tcp_fastopen = 3                # Fast open
net.ipv4.tcp_slow_start_after_idle = 0   # Disable slow start
net.ipv4.tcp_tw_reuse = 1                # Reuse TIME_WAIT sockets
net.ipv4.tcp_fin_timeout = 30            # Reduce FIN timeout

# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 8192

# Connection tracking
net.netfilter.nf_conntrack_max = 262144
```

**Apply changes:**
```bash
sudo sysctl -p
```

### Network Interface Settings

```bash
# Set MTU
ip link set dev eth0 mtu 9000    # Jumbo frames

# Disable IPv6 (if not used)
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf

# Enable IP forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
```

## Kernel Tuning

### Sysctl Parameters

**General Performance:**
```bash
# /etc/sysctl.conf

# Kernel
kernel.pid_max = 4194304                # Max PIDs

# File system
fs.file-max = 1000000                   # Max open files
fs.inotify.max_user_watches = 524288    # inotify watches

# Virtual memory
vm.dirty_ratio = 10                     # Start writing at 10%
vm.dirty_background_ratio = 5           # Background write at 5%
vm.swappiness = 10                      # Avoid swap
vm.vfs_cache_pressure = 50              # Keep dentry/inode cache

# Network
net.core.netdev_max_backlog = 5000
net.core.rmem_default = 262144
net.core.wmem_default = 262144
```

### Interrupt Request (IRQ) Tuning

```bash
# Xem IRQ usage
cat /proc/interrupts

# IRQ affinity
cat /proc/irq/30/smp_affinity

# Set IRQ affinity to CPU 0-3
echo f > /proc/irq/30/smp_affinity      # f = 0xF = 1111 (CPU 0-3)

# Install irqbalance
sudo apt install irqbalance
sudo systemctl enable irqbalance
```

### Disable Unnecessary Services

```bash
# List all services
systemctl list-unit-files --type=service

# Disable bluetooth
systemctl disable bluetooth

# Disable cups (printing)
systemctl disable cups

# Common services to consider disabling:
systemctl disable avahi-daemon     # Network discovery
systemctl disable ModemManager     # Modem management
systemctl disable accounts-daemon  # User accounts
```

## Application-Level Tuning

### Web Servers

**Nginx Tuning:**
```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;              # = số CPU cores
worker_rlimit_nofile 65536;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Connection tuning
    keepalive_timeout 65;
    keepalive_requests 100;
    
    # Buffer tuning
    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;
    
    # Sendfile
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # Compression
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 256;
    gzip_types text/plain text/css application/json;
    
    # Open file cache
    open_file_cache max=10000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
}
```

**Apache Tuning:**
```apache
# /etc/apache2/apache2.conf

# MPM Prefork
<IfModule mpm_prefork_module>
    StartServers 5
    MinSpareServers 5
    MaxSpareServers 10
    MaxRequestWorkers 150
    MaxConnectionsPerChild 0
</IfModule>

# MPM Worker (better performance)
<IfModule mpm_worker_module>
    StartServers 2
    MinSpareThreads 25
    MaxSpareThreads 75
    ThreadsPerChild 25
    MaxRequestWorkers 150
    MaxConnectionsPerChild 0
</IfModule>

# Keep-Alive
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

### Database Tuning

**MySQL/MariaDB:**
```ini
# /etc/mysql/my.cnf

[mysqld]
# Connection
max_connections = 200
max_connect_errors = 1000
connect_timeout = 10

# Buffer pool (70-80% of RAM)
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8

# Log files
innodb_log_file_size = 512M
innodb_log_buffer_size = 16M
innodb_flush_log_at_trx_commit = 2

# Query cache (deprecated in MySQL 8.0)
query_cache_type = 1
query_cache_size = 128M

# Thread cache
thread_cache_size = 50
table_open_cache = 4000

# Temp tables
tmp_table_size = 64M
max_heap_table_size = 64M
```

**PostgreSQL:**
```ini
# /etc/postgresql/13/main/postgresql.conf

# Memory
shared_buffers = 2GB                    # 25% of RAM
effective_cache_size = 6GB              # 50-75% of RAM
work_mem = 64MB
maintenance_work_mem = 512MB

# Checkpoint
checkpoint_completion_target = 0.9
wal_buffers = 16MB

# Planner
random_page_cost = 1.1                  # For SSDs
effective_io_concurrency = 200

# Workers
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
```

### Redis Tuning

```bash
# /etc/redis/redis.conf

# Memory
maxmemory 2gb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000

# AOF
appendonly yes
appendfsync everysec

# Performance
tcp-backlog 511
timeout 300
tcp-keepalive 60

# Disable THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Overcommit memory
sysctl vm.overcommit_memory=1
```

## Monitoring and Profiling

### Performance Monitoring Tools

```bash
# Install performance tools
sudo apt install sysstat iotop iftop htop dstat

# Enable sysstat
sudo systemctl enable sysstat
sudo systemctl start sysstat

# View sar data
sar -u          # CPU
sar -r          # Memory
sar -d          # Disk
sar -n DEV      # Network

# Historical data (from /var/log/sysstat/)
sar -u -f /var/log/sysstat/sa$(date +%d)
```

### System-wide Profiling

```bash
# perf - Linux profiling tool
sudo apt install linux-tools-generic

# Record system-wide
perf record -a -g sleep 30
perf report

# Record specific process
perf record -p 1234 -g sleep 10
perf report

# CPU flame graph
perf record -F 99 -a -g -- sleep 30
perf script > out.perf
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flamegraph.svg
```

### Application Profiling

**Python:**
```python
# cProfile
python -m cProfile -o output.prof script.py

# Analyze
python -m pstats output.prof
>>> sort time
>>> stats 10
```

**Node.js:**
```bash
# Built-in profiler
node --prof app.js

# Analyze
node --prof-process isolate-*.log > processed.txt

# clinic.js
npm install -g clinic
clinic doctor -- node app.js
```

**Java:**
```bash
# JVM tuning
java -Xms2g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -jar app.jar
```

## Performance Benchmarking

### CPU Benchmark

```bash
# sysbench CPU
sudo apt install sysbench
sysbench cpu --cpu-max-prime=20000 run

# Multi-threaded
sysbench cpu --threads=4 --cpu-max-prime=20000 run
```

### Memory Benchmark

```bash
# sysbench memory
sysbench memory --memory-block-size=1K --memory-total-size=10G run

# Stream benchmark
sudo apt install stream
stream
```

### Disk Benchmark

```bash
# dd write test
dd if=/dev/zero of=/tmp/test bs=1M count=1024 oflag=direct

# dd read test
dd if=/tmp/test of=/dev/null bs=1M count=1024 iflag=direct

# fio - Flexible I/O tester
sudo apt install fio

# Random read
fio --name=random-read --ioengine=libaio --iodepth=32 \
    --rw=randread --bs=4k --size=1G --numjobs=4 \
    --runtime=60 --group_reporting

# Random write
fio --name=random-write --ioengine=libaio --iodepth=32 \
    --rw=randwrite --bs=4k --size=1G --numjobs=4 \
    --runtime=60 --group_reporting
```

### Network Benchmark

```bash
# iperf3
sudo apt install iperf3

# Server
iperf3 -s

# Client
iperf3 -c server_ip -t 30

# UDP test
iperf3 -c server_ip -u -b 1G
```

## Complete Tuning Script

```bash
#!/bin/bash
# performance_optimize.sh - Complete system optimization

set -e

echo "=== Linux Performance Optimization ==="

# Backup original config
cp /etc/sysctl.conf /etc/sysctl.conf.backup

# CPU Tuning
echo "Tuning CPU..."
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Memory Tuning
echo "Tuning Memory..."
cat >> /etc/sysctl.conf << 'SYSCTL'

# Memory Management
vm.swappiness = 10
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.vfs_cache_pressure = 50

# Network Tuning
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.core.netdev_max_backlog = 5000
net.core.somaxconn = 1024
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 8192

# File System
fs.file-max = 1000000
fs.inotify.max_user_watches = 524288
SYSCTL

sysctl -p

# Disable THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Ulimits
cat >> /etc/security/limits.conf << 'LIMITS'
* soft nofile 65536
* hard nofile 65536
* soft nproc 32768
* hard nproc 32768
LIMITS

# I/O Scheduler
echo deadline > /sys/block/sda/queue/scheduler

echo "Optimization complete! Please reboot for full effect."
```

## Troubleshooting Performance Issues

### High Load Average

```bash
# 1. Check what's using CPU
top -o %CPU
ps aux --sort=-%cpu | head

# 2. Check I/O wait
iostat -x 1

# 3. Check processes in D state
ps aux | grep " D "
```

### Memory Pressure

```bash
# 1. Check OOM logs
dmesg | grep -i "out of memory"

# 2. Identify memory hog
ps aux --sort=-%mem | head

# 3. Check if swap is thrashing
vmstat 1
```

### Slow Disk I/O

```bash
# 1. Find busy disk
iostat -x 1

# 2. Find process doing I/O
iotop -o

# 3. Check for disk errors
dmesg | grep -i error
smartctl -a /dev/sda
```

## Best Practices

### Do's
- Benchmark before and after changes
- Change one thing at a time
- Monitor impact of changes
- Document all modifications
- Test in staging first
- Use monitoring tools
- Set up alerts
- Regular performance reviews

### Don'ts
- Don't blindly copy configs
- Don't disable security features without understanding
- Don't tune prematurely
- Don't ignore monitoring
- Don't forget to backup configs
- Don't tune without baseline

## Tổng kết

Performance tuning = Art + Science:
- **Measure first** - Use profiling tools
- **Identify bottleneck** - CPU, memory, I/O, network
- **Apply targeted fixes** - Don't over-optimize
- **Monitor impact** - Verify improvements
- **Document changes** - For troubleshooting

**Performance là ongoing process**, không phải one-time task!
