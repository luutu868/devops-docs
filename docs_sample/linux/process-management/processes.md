# Quản Lý Tiến Trình Linux (Process Management)

## Tổng quan về Process

### Process là gì?

Process (tiến trình) là một chương trình đang chạy trong bộ nhớ. Mỗi process có:
- **PID** (Process ID): Định danh duy nhất
- **PPID** (Parent Process ID): ID của process cha
- **UID/GID**: User và Group ID sở hữu process
- **Memory**: Vùng nhớ riêng biệt
- **File Descriptors**: Các file đang mở
- **State**: Trạng thái hiện tại

### Vòng đời của Process

```
     fork()          exec()         exit()
[Parent] ──→ [Child] ──→ [Running] ──→ [Zombie] ──→ [Terminated]
                              ↓
                          wait()
                              ↓
                          [Reaped]
```

### Process States

| State | Ký hiệu | Mô tả |
|-------|---------|-------|
| Running | R | Đang chạy hoặc sẵn sàng chạy |
| Sleeping | S | Chờ sự kiện (interruptible) |
| Uninterruptible | D | Chờ I/O, không thể interrupt |
| Stopped | T | Bị dừng bởi signal |
| Zombie | Z | Đã kết thúc nhưng chưa được reaped |

## Xem Thông Tin Process

### ps - Process Status

```bash
# Xem process của user hiện tại
ps

# Xem tất cả process (BSD style)
ps aux

# Xem tất cả process (Unix style)
ps -ef

# Xem process dạng tree
ps auxf
ps -ejH

# Filter process
ps aux | grep nginx
ps -ef | grep java

# Custom output format
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -10

# Xem threads
ps -eLf
ps -p 1234 -L
```

**Giải thích columns:**
```
USER    PID  %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root      1   0.0  0.1  169416 13612 ?     Ss   Jan01   0:45 /sbin/init
```

- **VSZ**: Virtual memory size (KB)
- **RSS**: Resident Set Size - RAM thực tế (KB)
- **STAT**: Process state
  - `S`: Sleeping
  - `s`: Session leader
  - `+`: Foreground process
  - `<`: High priority
  - `N`: Low priority

### top - Real-time Monitoring

```bash
# Chạy top
top

# Top với user cụ thể
top -u nginx

# Update mỗi 2 giây
top -d 2

# Batch mode (không interactive)
top -b -n 1

# Sort by memory
top -o %MEM

# Sort by CPU
top -o %CPU
```

**Phím tắt trong top:**
- `M`: Sort by memory
- `P`: Sort by CPU
- `k`: Kill process
- `r`: Renice process
- `f`: Fields management
- `1`: Toggle CPU cores
- `q`: Quit

### htop - Interactive Process Viewer

```bash
# Install htop
sudo apt install htop    # Ubuntu/Debian
sudo yum install htop    # CentOS/RHEL

# Chạy htop
htop

# Filter by user
htop -u nginx

# Sort by memory
htop -s PERCENT_MEM
```

**Phím tắt htop:**
- `F3`: Search
- `F4`: Filter
- `F5`: Tree view
- `F6`: Sort
- `F9`: Kill
- `F10`: Quit

### Lệnh khác

```bash
# pstree - Xem process tree
pstree
pstree -p        # Hiển thị PID
pstree -u        # Hiển thị user
pstree 1234      # Tree từ PID cụ thể

# pidof - Tìm PID của process
pidof nginx
pidof -s nginx   # Chỉ 1 PID

# pgrep - Pattern-based process search
pgrep nginx
pgrep -u root
pgrep -l nginx   # Hiển thị cả tên

# jobs - Xem background jobs
jobs
jobs -l          # Hiển thị PID
```

## Kiểm Soát Process

### Foreground và Background

```bash
# Chạy command foreground
sleep 100

# Chạy command background
sleep 100 &

# Đưa process ra background (Ctrl+Z rồi bg)
sleep 100
# Press Ctrl+Z
bg

# Đưa process lên foreground
fg
fg %1            # Job number 1

# Danh sách jobs
jobs
```

### Signals

```bash
# Gửi signal đến process
kill -SIGNAL PID

# Kill process (SIGTERM)
kill 1234

# Force kill (SIGKILL)
kill -9 1234
kill -KILL 1234

# Reload config (SIGHUP)
kill -HUP 1234

# Stop process (SIGSTOP)
kill -STOP 1234

# Continue process (SIGCONT)
kill -CONT 1234

# Terminate process (SIGTERM)
kill -TERM 1234
kill -15 1234
```

**Các signal quan trọng:**

| Signal | Number | Mô tả | Có thể catch? |
|--------|--------|-------|---------------|
| SIGHUP | 1 | Hangup, reload config | Có |
| SIGINT | 2 | Interrupt (Ctrl+C) | Có |
| SIGQUIT | 3 | Quit (Ctrl+\) | Có |
| SIGKILL | 9 | Force kill | Không |
| SIGTERM | 15 | Graceful termination | Có |
| SIGSTOP | 19 | Stop process | Không |
| SIGCONT | 18 | Continue stopped process | Có |

```bash
# Xem tất cả signals
kill -l

# pkill - Kill by name
pkill nginx
pkill -9 java
pkill -u www-data

# killall - Kill all processes by name
killall nginx
killall -9 php-fpm
```

### Process Priority - Nice và Renice

**Nice value:**
- Range: -20 (highest) đến 19 (lowest)
- Default: 0
- Chỉ root mới set negative nice

```bash
# Chạy process với nice value
nice -n 10 tar -czf backup.tar.gz /data

# Chạy với priority thấp nhất
nice -n 19 ./slow-job.sh

# Xem nice value
ps -el | grep PID
top

# Renice - Thay đổi priority của process đang chạy
renice -n 5 -p 1234
renice +10 1234
renice -5 -u nginx

# Renice tất cả process của user
renice +5 -u developer
```

### nohup - Chạy Process Khi Logout

```bash
# Chạy command với nohup
nohup ./long-running-script.sh &

# Output redirect
nohup ./script.sh > output.log 2>&1 &

# Multiple commands
nohup bash -c 'command1; command2; command3' &
```

### screen và tmux - Terminal Multiplexer

**Screen:**
```bash
# Tạo screen session
screen -S mysession

# Detach (Ctrl+A D)

# List sessions
screen -ls

# Reattach
screen -r mysession

# Kill session
screen -X -S mysession quit
```

**tmux:**
```bash
# Tạo session
tmux new -s mysession

# Detach (Ctrl+B D)

# List sessions
tmux ls

# Attach
tmux attach -t mysession

# Kill session
tmux kill-session -t mysession

# Split panes
# Ctrl+B % (vertical)
# Ctrl+B " (horizontal)
```

## Giám Sát Process Chi Tiết

### /proc Filesystem

```bash
# Xem thông tin process
cat /proc/1234/status      # Tổng quan
cat /proc/1234/cmdline     # Command line
cat /proc/1234/environ     # Environment variables
cat /proc/1234/limits      # Resource limits
cat /proc/1234/fd/         # File descriptors
cat /proc/1234/maps        # Memory mappings

# CPU và Memory usage
cat /proc/1234/stat
cat /proc/1234/statm

# Process cwd và exe
ls -l /proc/1234/cwd
ls -l /proc/1234/exe
```

### lsof - List Open Files

```bash
# Install lsof
sudo apt install lsof

# Xem tất cả files đang mở
lsof

# Files mở bởi process cụ thể
lsof -p 1234

# Files mở bởi user
lsof -u nginx
lsof -u ^root    # Exclude root

# Process using specific file
lsof /var/log/nginx/access.log

# Network connections
lsof -i           # Tất cả
lsof -i :80       # Port 80
lsof -i TCP       # TCP only
lsof -i TCP:8080

# Combine options
lsof -u nginx -i :80
```

### strace - System Call Tracer

```bash
# Install strace
sudo apt install strace

# Trace process đang chạy
strace -p 1234

# Trace command mới
strace ls -la

# Save to file
strace -o output.txt ls

# Trace specific syscalls
strace -e open,read,write cat file.txt
strace -e trace=network curl google.com

# Count syscalls
strace -c ls

# Trace with timestamps
strace -t ls
strace -tt ls    # Microseconds

# Follow child processes
strace -f python script.py

# Trace file operations only
strace -e trace=file ls
```

**Ví dụ debug nginx:**
```bash
# Tìm master process
ps aux | grep nginx

# Trace nginx worker
sudo strace -p 1234 -f -e trace=network 2>&1 | grep -A 5 "accept\|connect"
```

### pidstat - Process Statistics

```bash
# Install sysstat
sudo apt install sysstat

# CPU usage mỗi giây
pidstat 1

# Memory usage
pidstat -r 1

# I/O statistics
pidstat -d 1

# Specific process
pidstat -p 1234 1

# Show threads
pidstat -t -p 1234 1

# Context switches
pidstat -w 1
```

## Quản Lý Resources

### ulimit - User Limits

```bash
# Xem tất cả limits
ulimit -a

# Xem max open files
ulimit -n

# Set soft limit (current session)
ulimit -n 4096

# Set hard limit (requires root)
ulimit -Hn 65536

# Unlimited core dumps
ulimit -c unlimited

# Max processes
ulimit -u 2048
```

**Cấu hình persistent trong /etc/security/limits.conf:**
```bash
# /etc/security/limits.conf
nginx    soft    nofile    4096
nginx    hard    nofile    65536
nginx    soft    nproc     2048
*        soft    core      unlimited
```

### cgroups - Control Groups

```bash
# Xem cgroups của process
cat /proc/1234/cgroup

# List cgroups
ls /sys/fs/cgroup/

# Tạo cgroup mới (memory limit)
sudo mkdir /sys/fs/cgroup/memory/mygroup
echo 512M > /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes

# Add process vào cgroup
echo 1234 > /sys/fs/cgroup/memory/mygroup/cgroup.procs

# CPU limit
mkdir /sys/fs/cgroup/cpu/mygroup
echo 50000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us
echo 100000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_period_us
```

**Sử dụng systemd cgroups:**
```bash
# Slice configuration
# /etc/systemd/system/myapp.slice
[Unit]
Description=My App Resource Limit

[Slice]
MemoryLimit=1G
CPUQuota=50%
```

### cpulimit - Limit CPU Usage

```bash
# Install cpulimit
sudo apt install cpulimit

# Limit PID to 50% CPU
cpulimit -p 1234 -l 50

# Limit command to 25% CPU
cpulimit -l 25 -- tar -czf backup.tar.gz /data

# Background mode
cpulimit -b -p 1234 -l 30
```

## Systemd Process Management

### systemctl Commands

```bash
# Xem service status
systemctl status nginx

# Start/Stop/Restart service
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx

# Enable/Disable auto-start
systemctl enable nginx
systemctl disable nginx

# Xem logs
journalctl -u nginx
journalctl -u nginx -f      # Follow
journalctl -u nginx --since "1 hour ago"

# List all services
systemctl list-units --type=service
systemctl list-units --type=service --state=running
```

### Tạo Systemd Service

**Ví dụ: Node.js application**
```bash
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js Application
After=network.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=myapp

# Environment
Environment=NODE_ENV=production
Environment=PORT=3000

# Resource limits
LimitNOFILE=65536
MemoryLimit=1G
CPUQuota=50%

[Install]
WantedBy=multi-user.target
```

**Reload và start:**
```bash
systemctl daemon-reload
systemctl start myapp
systemctl enable myapp
systemctl status myapp
```

## Process Troubleshooting

### Process Consuming High CPU

```bash
# 1. Identify process
top -o %CPU
ps aux --sort=-%cpu | head

# 2. Xem threads
top -H -p 1234
ps -eLf | grep 1234

# 3. Stack trace
sudo gdb -p 1234
(gdb) thread apply all bt
(gdb) quit

# 4. strace to see syscalls
strace -p 1234 -c
```

### Process Consuming High Memory

```bash
# 1. Identify process
top -o %MEM
ps aux --sort=-%mem | head

# 2. Memory map
pmap -x 1234
cat /proc/1234/smaps

# 3. Check for memory leaks
valgrind --leak-check=full ./myapp

# 4. Get heap dump (Java)
jmap -dump:format=b,file=heap.bin 1234
```

### Zombie Processes

```bash
# Tìm zombie processes
ps aux | grep 'Z'
ps -eo pid,ppid,state,cmd | grep '^[0-9]* [0-9]* Z'

# Xem parent process
ps -o ppid= -p <zombie_pid>

# Kill parent để reap zombie
kill -HUP <parent_pid>
```

### Process Không Response

```bash
# 1. Check process state
ps aux | grep myapp

# 2. Nếu state = D (uninterruptible sleep)
cat /proc/<PID>/stack      # Kernel stack trace
cat /proc/<PID>/wchan      # Waiting channel

# 3. Check I/O
iotop -p 1234
pidstat -d -p 1234 1

# 4. Tạo core dump
gcore 1234

# 5. Analyze with gdb
gdb /path/to/binary core.1234
```

### "Too Many Open Files" Error

```bash
# 1. Check current usage
lsof -p 1234 | wc -l

# 2. Check limit
cat /proc/1234/limits | grep "open files"

# 3. Increase limit temporarily
prlimit --pid 1234 --nofile=65536

# 4. Fix permanently
# /etc/security/limits.conf
myapp    soft    nofile    65536
myapp    hard    nofile    65536

# For systemd services
# Add to .service file:
LimitNOFILE=65536
```

## Automation Scripts

### Monitor Specific Process

```bash
#!/bin/bash
# monitor_process.sh

PROCESS_NAME="nginx"
EMAIL="admin@example.com"

while true; do
    if ! pgrep -x "$PROCESS_NAME" > /dev/null; then
        echo "$(date): $PROCESS_NAME is not running" | \
            mail -s "Alert: $PROCESS_NAME down" "$EMAIL"
        
        # Restart service
        systemctl start $PROCESS_NAME
    fi
    sleep 60
done
```

### Kill Process If High Memory

```bash
#!/bin/bash
# kill_high_memory.sh

THRESHOLD=80  # Percent
PROCESS_NAME="myapp"

USAGE=$(ps aux | grep "$PROCESS_NAME" | grep -v grep | awk '{print $4}')

if (( $(echo "$USAGE > $THRESHOLD" | bc -l) )); then
    PID=$(pgrep -o "$PROCESS_NAME")
    echo "$(date): Killing $PROCESS_NAME (PID: $PID) - Memory: $USAGE%"
    kill -9 $PID
    systemctl start $PROCESS_NAME
fi
```

### Export Process Metrics

```bash
#!/bin/bash
# export_metrics.sh

OUTPUT_FILE="/var/log/process_metrics.csv"

# Header
echo "timestamp,process,pid,cpu,mem,threads" > $OUTPUT_FILE

while true; do
    ps aux --no-headers | awk '{
        printf "%s,%s,%s,%s,%s\n", 
        strftime("%Y-%m-%d %H:%M:%S"),
        $11, $2, $3, $4
    }' >> $OUTPUT_FILE
    
    sleep 60
done
```

### Auto-restart Dead Process

```bash
#!/bin/bash
# auto_restart.sh

PROCESS_NAME="myapp"
CHECK_INTERVAL=30
MAX_RESTARTS=3
RESTART_COUNT=0

while true; do
    if ! systemctl is-active --quiet $PROCESS_NAME; then
        RESTART_COUNT=$((RESTART_COUNT + 1))
        
        if [ $RESTART_COUNT -le $MAX_RESTARTS ]; then
            echo "$(date): Restarting $PROCESS_NAME (attempt $RESTART_COUNT)"
            systemctl restart $PROCESS_NAME
        else
            echo "$(date): Max restart attempts reached" | \
                mail -s "Critical: $PROCESS_NAME failed" admin@example.com
            exit 1
        fi
    else
        RESTART_COUNT=0
    fi
    
    sleep $CHECK_INTERVAL
done
```

## Best Practices

### 1. Process Management
- Sử dụng systemd để quản lý services
- Set ulimits phù hợp cho từng application
- Monitor memory leaks định kỳ
- Sử dụng graceful shutdown (SIGTERM) thay vì force kill (SIGKILL)
- Configure proper restart policies

### 2. Monitoring
- Setup alerts cho high CPU/memory usage
- Log process restarts
- Track zombie processes
- Monitor file descriptor usage
- Use centralized logging (syslog, journald)

### 3. Resource Limits
- Set memory limits để prevent OOM
- Configure CPU quotas cho non-critical services
- Limit max file descriptors per process
- Use cgroups cho isolation
- Set core dump limits

### 4. Debugging
- Enable core dumps cho critical services
- Keep debug symbols
- Use strace sparingly (high overhead)
- Profile trước khi optimize
- Document troubleshooting steps

### 5. Security
- Run services with minimum required privileges
- Use dedicated users cho mỗi service
- Disable core dumps for sensitive processes
- Monitor unauthorized process creations
- Audit process capabilities

## Tổng kết

Process management là kỹ năng quan trọng cho:
- **DevOps Engineers**: Deploy và maintain applications
- **SREs**: Troubleshoot production issues
- **System Administrators**: Optimize server performance
- **Developers**: Debug application issues

**Key takeaways:**
- Hiểu process lifecycle và states
- Master các tools: ps, top, htop, lsof, strace
- Sử dụng systemd để quản lý services
- Set resource limits phù hợp
- Monitor và alert proactively
- Document recovery procedures

Kỹ năng process management sẽ giúp bạn:
- Tăng uptime và reliability
- Giảm MTTR (Mean Time To Recovery)
- Optimize resource utilization
- Debug issues nhanh chóng
- Automate operational tasks
