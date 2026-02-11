# Linux Cơ Bản

## 🐧 Linux Là Gì?

**Linux** là một hệ điều hành mã nguồn mở dựa trên Unix, được tạo ra bởi Linus Torvalds năm 1991. Linux là nền tảng cho phần lớn servers, cloud infrastructure, và DevOps tools.

### **Tại Sao DevOps Cần Linux?**

- **96% servers** chạy Linux
-  **100% top 500** supercomputers dùng Linux  
- ☁️ **AWS, GCP, Azure** chủ yếu là Linux
- **Docker & Kubernetes** native trên Linux
- 💰 **Free & Open Source**

## Linux vs Windows vs MacOS

| Feature | Linux | Windows | MacOS |
|---------|-------|---------|-------|
| **Cost** | Free | $$$  | $$$ |
| **Open Source** | Yes | No | No |
| **Customization** | High | Low | Medium |
| **Server Market** | 96% | 3% | <1% |
| **Command Line** | Powerful | Limited | Good |
| **Package Manager** | Yes (apt, yum) | Limited | Homebrew |
| **Docker Native** | Yes | WSL2 | VM |
| **Learning Curve** | Medium | Low | Low |

## 📂 Linux Distributions (Distros)

### **Enterprise Distros**

#### **1. Red Hat Enterprise Linux (RHEL)**
```
- Enterprise support
- Stable, secure
- Paid subscription
- Used by: Fortune 500 companies
```

#### **2. Ubuntu Server**
```
- Most popular
- Large community
- LTS versions (5 years support)
- Used by: AWS, Startups
```

#### **3. CentOS / Rocky Linux / AlmaLinux**
```
- RHEL compatible
- Free
- Community support
- Used by: Web hosting
```

#### **4. Debian**
```
- Very stable
- Large package repository
- Base for Ubuntu
- Used by: Servers, DevOps
```

### **Desktop Distros**

- **Ubuntu Desktop**: User-friendly
- **Fedora**: Cutting edge
- **Linux Mint**: Windows-like
- **Arch Linux**: DIY, advanced

### **Specialized Distros**

- **Alpine Linux**: Minimal, Docker images (5MB)
- **CoreOS / Flatcar**: Container-optimized
- **Amazon Linux 2**: AWS optimized

**Khuyến nghị cho DevOps**: Ubuntu 22.04 LTS hoặc Rocky Linux 9

## Linux Architecture

```
┌─────────────────────────────────────────────┐
│         Applications & Users                │
│   (Web browser, editors, Docker, etc.)      │
├─────────────────────────────────────────────┤
│              Shell / CLI                    │
│        (bash, zsh, commands)                │
├─────────────────────────────────────────────┤
│            System Libraries                 │
│         (glibc, systemd, etc.)              │
├─────────────────────────────────────────────┤
│             Linux Kernel                    │
│   (Process mgmt, Memory, Drivers, FS)       │
├─────────────────────────────────────────────┤
│              Hardware                       │
│      (CPU, RAM, Disk, Network)              │
└─────────────────────────────────────────────┘
```

### **Kernel**

```bash
# Kiểm tra kernel version
uname -r
# Output: 5.15.0-91-generic

# Xem thông tin chi tiết
uname -a

# Kernel parameters
cat /proc/version
```

## 📁 Linux File System Hierarchy

```
/                          (Root - thư mục gốc)
├── bin/                   (Essential binaries: ls, cp, cat)
├── boot/                  (Boot loader files, kernel)
├── dev/                   (Device files: sda, tty)
├── etc/                   (Configuration files)
│   ├── passwd            (User accounts)
│   ├── hosts             (Host name mapping)
│   └── nginx/            (App configs)
├── home/                  (User home directories)
│   ├── user1/
│   └── user2/
├── lib/                   (Shared libraries)
├── media/                 (Removable media mount points)
├── mnt/                   (Temporary mount points)
├── opt/                   (Optional software packages)
├── proc/                  (Process information - virtual)
├── root/                  (Root user home directory)
├── run/                   (Runtime data)
├── sbin/                  (System binaries: reboot, fdisk)
├── srv/                   (Service data)
├── sys/                   (System information - virtual)
├── tmp/                   (Temporary files)
├── usr/                   (User programs)
│   ├── bin/              (User commands)
│   ├── lib/              (Libraries)
│   ├── local/            (Locally installed software)
│   └── share/            (Shared data)
└── var/                   (Variable data)
    ├── log/              (Log files)
    ├── www/              (Web server files)
    └── lib/              (State information)
```

### **Important Directories**

| Directory | Purpose | Example |
|-----------|---------|---------|
| `/etc` | Configuration | `/etc/nginx/nginx.conf` |
| `/var/log` | Log files | `/var/log/syslog` |
| `/home` | User data | `/home/vnpt/projects` |
| `/tmp` | Temp files | Cleared on reboot |
| `/opt` | Third-party apps | `/opt/google/chrome` |
| `/usr/local` | Locally compiled | `/usr/local/bin/myapp` |

## 👤 Users & Groups

### **Types of Users**

1. **Root (superuser)**
   ```bash
   UID: 0
   Home: /root
   Shell: /bin/bash
   ```

2. **System users**
   ```bash
   UID: 1-999
   For services: www-data, mysql, docker
   ```

3. **Regular users**
   ```bash
   UID: 1000+
   Home: /home/username
   ```

### **User Management**

```bash
# View current user
whoami

# View user info
id
# uid=1000(vnpt) gid=1000(vnpt) groups=1000(vnpt),27(sudo),999(docker)

# View all users
cat /etc/passwd
# Format: username:x:UID:GID:comment:home:shell

# Create user
sudo useradd -m -s /bin/bash newuser
sudo passwd newuser

# Add user to group
sudo usermod -aG docker vnpt

# Delete user
sudo userdel -r username

# Switch user
su - username

# Run as root
sudo command
```

### **Groups**

```bash
# View groups
groups

# View all groups
cat /etc/group

# Create group
sudo groupadd developers

# Add user to group
sudo usermod -aG developers vnpt
```

## File Permissions

### **Permission Structure**

```
-rwxrw-r-- 1 vnpt developers 1024 Feb 09 10:30 script.sh
│││││││││  │  │      │       │    │             │
│││││││││  │  │      │       │    │             └─ Filename
│││││││││  │  │      │       │    └─ Timestamp
│││││││││  │  │      │       └─ Size (bytes)
│││││││││  │  │      └─ Group
│││││││││  │  └─ Owner
│││││││││  └─ Number of links
│││││││││
││││││││└─ Other: r-- (read only)
│││││└─└── Group: rw- (read, write)
││└└└───── Owner: rwx (read, write, execute)
└───────── File type: - (regular), d (directory), l (link)
```

### **Permission Values**

```
r (read)    = 4
w (write)   = 2
x (execute) = 1

rwx = 4+2+1 = 7
rw- = 4+2   = 6
r-x = 4+1   = 5
r-- = 4     = 4
```

### **Common Permissions**

```bash
# 755: Owner full, others read+execute
-rwxr-xr-x  (Typical for executables)

# 644: Owner rw, others read
-rw-r--r--  (Typical for files)

# 600: Owner only
-rw-------  (Private files, SSH keys)

# 777: Everyone full (DANGEROUS!)
-rwxrwxrwx  (Avoid this!)
```

### **Changing Permissions**

```bash
# Chmod - Change mode
chmod 755 script.sh
chmod u+x script.sh        # Add execute for user
chmod g-w file.txt         # Remove write for group
chmod o=r file.txt         # Set other to read only

# Chown - Change owner
sudo chown vnpt file.txt
sudo chown vnpt:developers file.txt

# Recursive
sudo chmod -R 755 /var/www/html
sudo chown -R www-data:www-data /var/www
```

## Package Management

### **APT (Debian/Ubuntu)**

```bash
# Update package list
sudo apt update

# Upgrade packages
sudo apt upgrade

# Install package
sudo apt install nginx

# Remove package
sudo apt remove nginx
sudo apt purge nginx  # Also remove configs

# Search package
apt search docker

# Show package info
apt show docker.io

# List installed
apt list --installed

# Clean cache
sudo apt clean
sudo apt autoclean
sudo apt autoremove
```

### **YUM/DNF (RHEL/CentOS/Fedora)**

```bash
# Update
sudo yum update
sudo dnf update  # Fedora

# Install
sudo yum install nginx

# Remove
sudo yum remove nginx

# Search
yum search docker

# Info
yum info docker

# List installed
yum list installed
```

## 🔧 System Information

```bash
# OS Information
cat /etc/os-release
lsb_release -a
hostnamectl

# CPU Info
lscpu
cat /proc/cpuinfo
nproc  # Number of cores

# Memory Info
free -h
cat /proc/meminfo

# Disk Space
df -h
du -sh /var/log

# System Uptime
uptime

# Load Average
top
htop  # Better version

# Hardware Info
lshw
lspci  # PCI devices
lsusb  # USB devices
lsblk  # Block devices
```

##  Essential Commands

### **Navigation**

```bash
pwd           # Print working directory
cd /path      # Change directory
cd ~          # Home directory
cd -          # Previous directory
ls            # List files
ls -la        # List all with details
tree          # Tree view (install first)
```

### **File Operations**

```bash
touch file.txt              # Create empty file
cat file.txt                # View file content
less file.txt               # View file (paginated)
head -n 10 file.txt         # First 10 lines
tail -n 10 file.txt         # Last 10 lines
tail -f /var/log/syslog     # Follow log file

cp source dest              # Copy
mv source dest              # Move/rename
rm file.txt                 # Remove
rm -rf directory/           # Remove directory recursively

mkdir new_dir               # Create directory
mkdir -p path/to/new_dir    # Create parent dirs
rmdir empty_dir             # Remove empty directory
```

## DevOps Use Cases

### **1. Setting Up Web Server**

```bash
# Install Nginx
sudo apt update
sudo apt install nginx -y

# Start service
sudo systemctl start nginx
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx

# Configure
sudo vi /etc/nginx/sites-available/default

# Test config
sudo nginx -t

# Reload
sudo systemctl reload nginx
```

### **2. Managing Services**

```bash
# systemd commands
sudo systemctl start service
sudo systemctl stop service
sudo systemctl restart service
sudo systemctl reload service
sudo systemctl enable service
sudo systemctl disable service
sudo systemctl status service

# List all services
systemctl list-units --type=service

# View logs
journalctl -u nginx
journalctl -u nginx -f  # Follow
```

### **3. Checking Logs**

```bash
# System logs
sudo tail -f /var/log/syslog
sudo tail -f /var/log/auth.log

# Application logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Using journalctl
sudo journalctl -xe
sudo journalctl --since "1 hour ago"
sudo journalctl -u docker.service
```

## Environment Variables

```bash
# View environment variables
env
printenv

# View specific variable
echo $PATH
echo $HOME

# Set temporary variable
export MY_VAR="value"

# Set permanent (add to ~/.bashrc)
echo 'export MY_VAR="value"' >> ~/.bashrc
source ~/.bashrc

# Common variables
$HOME     # User home directory
$USER     # Current username
$PATH     # Executable search path
$SHELL    # Current shell
$PWD      # Present working directory
```

## 🎓 Practice Labs

### **Lab 1: User Management**

```bash
# 1. Create a new user
sudo useradd -m -s /bin/bash devops_user

# 2. Set password
sudo passwd devops_user

# 3. Add to sudo group
sudo usermod -aG sudo devops_user

# 4. Test
su - devops_user
sudo whoami  # Should output: root
```

### **Lab 2: Permissions**

```bash
# 1. Create a script
echo '#!/bin/bash\necho "Hello DevOps"' > hello.sh

# 2. Make executable
chmod +x hello.sh

# 3. Run it
./hello.sh

# 4. Check permissions
ls -l hello.sh
```

## Checklist

- [ ] Cài đặt Ubuntu/Linux VM hoặc WSL2
- [ ] Hiểu cấu trúc thư mục Linux
- [ ] Thành thạo basic commands
- [ ] Hiểu permissions (chmod, chown)
- [ ] Biết quản lý users và groups
- [ ] Sử dụng package manager (apt/yum)
- [ ] Quản lý services với systemd
- [ ] Đọc và phân tích log files

---

**Tiếp theo**: [Linux Commands →](../commands/essential-commands.md)
