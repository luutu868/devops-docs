# Firewall và SSH Security

## iptables Fundamentals

### Cơ bản về iptables

iptables là công cụ quản lý tường lửa trên Linux, hoạt động ở layer 3 và 4 của OSI model.

**Kiến trúc iptables:**

```
┌─────────────────────────────────────────────┐
│             Tables                          │
├─────────────────────────────────────────────┤
│ filter   │ nat      │ mangle   │ raw       │
└─────────────────────────────────────────────┘
         │
         ├─── Chains
         │    ├─── INPUT    (đến server)
         │    ├─── OUTPUT   (từ server)
         │    ├─── FORWARD  (qua server)
         │    ├─── PREROUTING
         │    └─── POSTROUTING
         │
         └─── Rules (xử lý tuần tự từ trên xuống)
              ├─── Match criteria
              └─── Target (ACCEPT, DROP, REJECT, etc.)
```

**Tables:**
- **filter**: Lọc packets (default table)
- **nat**: Network Address Translation
- **mangle**: Thay đổi packet headers
- **raw**: Xử lý trước connection tracking

**Chains:**
- **INPUT**: Packets đến server
- **OUTPUT**: Packets từ server  
- **FORWARD**: Packets đi qua server (routing)
- **PREROUTING**: Packets trước routing
- **POSTROUTING**: Packets sau routing

**Targets:**
- **ACCEPT**: Cho phép packet
- **DROP**: Hủy packet (không phản hồi)
- **REJECT**: Từ chối packet (có phản hồi)
- **LOG**: Ghi log packet
- **MASQUERADE**: NAT động
- **SNAT/DNAT**: NAT tĩnh

### Cú pháp cơ bản

```bash
iptables [-t table] [action] [chain] [match] -j [target]
```

**Actions:**
- `-A`: Append (thêm rule vào cuối)
- `-I`: Insert (chèn rule vào đầu)
- `-D`: Delete (xóa rule)
- `-R`: Replace (thay thế rule)
- `-L`: List (liệt kê rules)
- `-F`: Flush (xóa tất cả rules)
- `-P`: Policy (đặt chính sách mặc định)

### Xem rules hiện tại

```bash
# List all rules
sudo iptables -L
sudo iptables -L -n  # No DNS resolution
sudo iptables -L -v  # Verbose
sudo iptables -L -n -v --line-numbers

# Specific chain
sudo iptables -L INPUT -n -v
sudo iptables -L OUTPUT -n -v

# NAT table
sudo iptables -t nat -L -n -v

# Mangle table
sudo iptables -t mangle -L -n -v

# Show rules as commands
sudo iptables-save
```

### Match criteria

```bash
# Source IP
-s 192.168.1.100
-s 192.168.1.0/24
-s ! 192.168.1.100  # NOT

# Destination IP
-d 10.0.0.1
-d 10.0.0.0/8

# Interface
-i eth0      # Input interface
-o eth0      # Output interface

# Protocol
-p tcp
-p udp
-p icmp
-p all

# Source port
--sport 22
--sport 1024:65535

# Destination port
--dport 80
--dport 443
--dport 8000:9000

# TCP flags
--tcp-flags SYN,ACK,FIN SYN  # SYN packet only

# Connection state
-m state --state NEW,ESTABLISHED,RELATED
-m conntrack --ctstate NEW,ESTABLISHED

# Multiple ports
-m multiport --dports 80,443,8080
-m multiport --sports 1024:65535

# IP range
-m iprange --src-range 192.168.1.100-192.168.1.200

# Rate limiting
-m limit --limit 5/min --limit-burst 10

# Recent connections
-m recent --name SSH --set
-m recent --name SSH --rcheck --seconds 60 --hitcount 4
```

## iptables Examples

### Basic Firewall Setup

```bash
#!/bin/bash

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (port 22)
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT

# Allow ping
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Log dropped packets
iptables -A INPUT -j LOG --log-prefix "INPUT DROP: " --log-level 4
iptables -A INPUT -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### SSH Brute Force Protection

```bash
# Method 1: Rate limiting
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --set --name SSH

iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH \
  -j LOG --log-prefix "SSH Brute Force: "

iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH \
  -j DROP

iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Method 2: Connection limit
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m connlimit --connlimit-above 3 --connlimit-mask 32 \
  -j REJECT --reject-with tcp-reset

# Method 3: Hashlimit
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m hashlimit --hashlimit-name ssh \
  --hashlimit-above 3/minute --hashlimit-burst 5 \
  --hashlimit-mode srcip -j DROP
```

### Port Forwarding (DNAT)

```bash
# Enable forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Forward port 8080 to internal server
iptables -t nat -A PREROUTING -p tcp --dport 8080 \
  -j DNAT --to-destination 192.168.1.100:80

iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 80 \
  -j ACCEPT

# Forward range of ports
iptables -t nat -A PREROUTING -p tcp --dport 8000:9000 \
  -j DNAT --to-destination 192.168.1.100:8000-9000
```

### Source NAT (SNAT)

```bash
# SNAT for outgoing traffic
iptables -t nat -A POSTROUTING -o eth0 -s 192.168.1.0/24 \
  -j SNAT --to-source 203.0.113.5

# MASQUERADE (dynamic IP)
iptables -t nat -A POSTROUTING -o ppp0 -j MASQUERADE

# MASQUERADE with port range
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE \
  --to-ports 1024-65535
```

### Block Specific IP/Network

```bash
# Block single IP
iptables -I INPUT -s 192.168.1.100 -j DROP

# Block network
iptables -I INPUT -s 10.0.0.0/8 -j DROP

# Block multiple IPs
iptables -I INPUT -s 192.168.1.100 -j DROP
iptables -I INPUT -s 192.168.1.101 -j DROP

# Using ipset (efficient for many IPs)
ipset create blacklist hash:ip
ipset add blacklist 192.168.1.100
ipset add blacklist 192.168.1.101
iptables -I INPUT -m set --match-set blacklist src -j DROP
```

### Allow from Specific IP Only

```bash
# SSH only from admin IP
iptables -A INPUT -p tcp --dport 22 -s 203.0.113.10 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP

# Multiple admin IPs with ipset
ipset create admin hash:ip
ipset add admin 203.0.113.10
ipset add admin 203.0.113.11
iptables -A INPUT -p tcp --dport 22 \
  -m set --match-set admin src -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

### Block Port Scanning

```bash
# Block NULL packets
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP

# Block SYN flood
iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP

# Block XMAS packets
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP

# Block invalid packets
iptables -A INPUT -m state --state INVALID -j DROP

# Limit SYN packets
iptables -A INPUT -p tcp --syn -m limit --limit 1/s \
  --limit-burst 3 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP
```

### Save and Restore Rules

```bash
# Save rules (Debian/Ubuntu)
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6

# Auto-load on boot
sudo apt install iptables-persistent

# Restore rules manually
sudo iptables-restore < /etc/iptables/rules.v4

# Save current rules (RHEL/CentOS)
sudo service iptables save
# or
sudo /sbin/iptables-save > /etc/sysconfig/iptables
```

## firewalld

### Giới thiệu firewalld

firewalld là firewall quản lý động, sử dụng zones và services.

**Zones (theo mức độ tin cậy):**
- **drop**: Hủy tất cả incoming
- **block**: Reject tất cả incoming
- **public**: Public networks (default)
- **external**: External networks with NAT
- **dmz**: DMZ zone
- **work**: Work networks
- **home**: Home networks
- **internal**: Internal networks
- **trusted**: Tin cậy tất cả

### Basic Commands

```bash
# Start/stop firewalld
sudo systemctl start firewalld
sudo systemctl stop firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld

# Check if running
sudo firewall-cmd --state

# Get default zone
sudo firewall-cmd --get-default-zone

# List all zones
sudo firewall-cmd --get-zones

# List active zones
sudo firewall-cmd --get-active-zones

# Set default zone
sudo firewall-cmd --set-default-zone=public
```

### Zone Management

```bash
# List zone configuration
sudo firewall-cmd --zone=public --list-all

# List all zones config
sudo firewall-cmd --list-all-zones

# Add interface to zone
sudo firewall-cmd --zone=public --add-interface=eth0
sudo firewall-cmd --zone=public --add-interface=eth0 --permanent

# Change interface zone
sudo firewall-cmd --zone=work --change-interface=eth0

# Remove interface from zone
sudo firewall-cmd --zone=public --remove-interface=eth0
```

### Service Management

```bash
# List available services
sudo firewall-cmd --get-services

# List services in zone
sudo firewall-cmd --zone=public --list-services

# Add service
sudo firewall-cmd --zone=public --add-service=http
sudo firewall-cmd --zone=public --add-service=https
sudo firewall-cmd --zone=public --add-service=ssh

# Add permanent
sudo firewall-cmd --zone=public --add-service=http --permanent

# Remove service
sudo firewall-cmd --zone=public --remove-service=http
sudo firewall-cmd --zone=public --remove-service=http --permanent

# Multiple services
sudo firewall-cmd --zone=public --add-service={http,https,ssh}
```

### Port Management

```bash
# Add port
sudo firewall-cmd --zone=public --add-port=8080/tcp
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent

# Port range
sudo firewall-cmd --zone=public --add-port=8000-9000/tcp

# List ports
sudo firewall-cmd --zone=public --list-ports

# Remove port
sudo firewall-cmd --zone=public --remove-port=8080/tcp
```

### Rich Rules

```bash
# Allow SSH from specific IP
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4" source address="203.0.113.10"
  service name="ssh" accept'

# Block IP
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4" source address="192.168.1.100" reject'

# Port forward
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4"
  forward-port port="80" protocol="tcp" to-port="8080"'

# Rate limit SSH
sudo firewall-cmd --zone=public --add-rich-rule='
  rule service name="ssh"
  limit value="3/m" accept'

# Log dropped packets
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4" source address="0.0.0.0/0"
  log prefix="FIREWALL-DROP: " level="info" limit value="5/m"
  drop'
```

### Port Forwarding

```bash
# Enable masquerading (NAT)
sudo firewall-cmd --zone=public --add-masquerade
sudo firewall-cmd --zone=public --add-masquerade --permanent

# Forward port
sudo firewall-cmd --zone=public --add-forward-port=\
port=8080:proto=tcp:toport=80:toaddr=192.168.1.100

# Permanent
sudo firewall-cmd --zone=public --add-forward-port=\
port=8080:proto=tcp:toport=80:toaddr=192.168.1.100 --permanent
```

### Reload and Runtime

```bash
# Reload firewall
sudo firewall-cmd --reload

# Apply runtime rules permanently
sudo firewall-cmd --runtime-to-permanent

# Panic mode (block all)
sudo firewall-cmd --panic-on
sudo firewall-cmd --panic-off
sudo firewall-cmd --query-panic
```

## UFW (Uncomplicated Firewall)

### Cài đặt và Kích hoạt

```bash
# Install (Ubuntu)
sudo apt install ufw

# Enable/disable
sudo ufw enable
sudo ufw disable

# Status
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered

# Reset to default
sudo ufw reset
```

### Default Policies

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed

# Allow/deny forwarding
sudo ufw default allow routed
```

### Allow/Deny Rules

```bash
# Allow service
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Allow port
sudo ufw allow 22
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8000:9000/tcp

# Allow from specific IP
sudo ufw allow from 203.0.113.10

# Allow from IP to specific port
sudo ufw allow from 203.0.113.10 to any port 22

# Allow from subnet
sudo ufw allow from 192.168.1.0/24

# Deny
sudo ufw deny 23
sudo ufw deny from 192.168.1.100
```

### Delete Rules

```bash
# By rule specification
sudo ufw delete allow 80
sudo ufw delete allow from 192.168.1.100

# By rule number
sudo ufw status numbered
sudo ufw delete 3
```

### Application Profiles

```bash
# List applications
sudo ufw app list

# Show app info
sudo ufw app info 'Nginx Full'

# Allow application
sudo ufw allow 'Nginx Full'
sudo ufw allow 'OpenSSH'
```

### Rate Limiting

```bash
# Rate limit SSH (max 6 conn in 30 sec)
sudo ufw limit ssh
sudo ufw limit 22/tcp

# Custom rate limit (requires ufw app)
sudo ufw limit proto tcp from any to any port 80
```

### Logging

```bash
# Enable logging
sudo ufw logging on
sudo ufw logging off

# Log levels
sudo ufw logging low
sudo ufw logging medium
sudo ufw logging high
sudo ufw logging full

# View logs
sudo tail -f /var/log/ufw.log
```

## SSH Security

### SSH Configuration

**/etc/ssh/sshd_config:**

```bash
# Basic security
Port 2222                         # Change default port
PermitRootLogin no                # Disable root login
PasswordAuthentication no         # Disable password auth
PubkeyAuthentication yes          # Enable key auth
PermitEmptyPasswords no           # No empty passwords
MaxAuthTries 3                    # Max login attempts
MaxSessions 2                     # Max concurrent sessions

# Allowed users/groups
AllowUsers alice bob admin
AllowGroups sshusers
# or
DenyUsers baduser
DenyGroups noremote

# Protocol and ciphers
Protocol 2                        # Only SSH protocol 2
Ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org

# Other security settings
X11Forwarding no                  # Disable X11
AllowTcpForwarding no             # Disable port forwarding
ClientAliveInterval 300           # Timeout idle sessions
ClientAliveCountMax 2             # Max missed keepalives
LoginGraceTime 30                 # Login timeout
StrictModes yes                   # Check file permissions

# Banner
Banner /etc/ssh/banner
```

Apply changes:
```bash
sudo systemctl reload sshd
# or
sudo service ssh reload
```

### SSH Key Authentication

```bash
# Generate key pair
ssh-keygen -t ed25519 -C "user@host"
# or
ssh-keygen -t rsa -b 4096 -C "user@host"

# Copy public key to server
ssh-copy-id user@server
# or manually
cat ~/.ssh/id_ed25519.pub | ssh user@server \
  'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'

# Set permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# Test connection
ssh -i ~/.ssh/id_ed25519 user@server
```

### SSH Config File

**~/.ssh/config:**

```bash
# Global settings
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    Compression yes
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

# Specific host
Host webserver
    HostName 203.0.113.10
    User deploy
    Port 2222
    IdentityFile ~/.ssh/webserver_key
    ForwardAgent no

Host db-server
    HostName db.example.com
    User admin
    ProxyJump jumphost

# Jump host
Host jumphost
    HostName jump.example.com
    User jumper
    Port 22
```

Use with: `ssh webserver`

### SSH Tunneling

```bash
# Local port forward
ssh -L 8080:localhost:80 user@server
# Access via: http://localhost:8080

# Remote port forward
ssh -R 8080:localhost:3000 user@server
# Server can access your local:3000 via server:8080

# Dynamic port forward (SOCKS proxy)
ssh -D 9090 user@server
# Configure browser to use localhost:9090 as SOCKS5 proxy

# Multiple tunnels
ssh -L 8080:localhost:80 -L 3306:localhost:3306 user@server

# Background tunnel
ssh -f -N -L 8080:localhost:80 user@server
# -f: Background
# -N: No command execution
```

### fail2ban

Protect SSH from brute force attacks.

```bash
# Install
sudo apt install fail2ban

# Configuration
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Edit /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600          # Ban for 1 hour
findtime = 600          # Within 10 minutes
maxretry = 3            # 3 failed attempts
destemail = admin@example.com
sendername = Fail2Ban
action = %(action_mwl)s  # Ban and email with log

[sshd]
enabled = true
port = ssh,2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600

# Start fail2ban
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Unban IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# View banned IPs
sudo fail2ban-client get sshd banip

# Live log
sudo tail -f /var/log/fail2ban.log
```

### Port Knocking

Hidden SSH access using port sequence.

**Install knockd:**
```bash
sudo apt install knockd
```

**/etc/knockd.conf:**
```bash
[options]
    UseSyslog

[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 5
    command     = /usr/sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn

[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 5
    command     = /usr/sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn
```

**Usage:**
```bash
# From client
knock server.com 7000 8000 9000
ssh user@server.com

# Close
knock server.com 9000 8000 7000
```

### Two-Factor Authentication (2FA)

**Install Google Authenticator:**
```bash
sudo apt install libpam-google-authenticator
```

**Setup:**
```bash
# Run as user
google-authenticator

# Scan QR code with authenticator app
# Save backup codes
```

**/etc/pam.d/sshd:**
```bash
# Add at top
auth required pam_google_authenticator.so
```

**/etc/ssh/sshd_config:**
```bash
ChallengeResponseAuthentication yes
UsePAM yes
AuthenticationMethods publickey,keyboard-interactive
```

```bash
sudo systemctl reload sshd
```

Now SSH requires both key AND 2FA code.

## Network Security Best Practices

1. **Firewall Default Deny:**
   - Default policy: DROP incoming
   - Explicitly allow only needed services

2. **SSH Hardening:**
   - Change default port
   - Disable root login
   - Use key authentication
   - Enable fail2ban
   - Consider 2FA

3. **Regular Updates:**
   - Keep firewall rules updated
   - Update SSH version
   - Monitor security advisories

4. **Monitoring:**
   - Monitor firewall logs
   - Check SSH auth logs
   - Set up alerts for suspicious activity

5. **Network Segmentation:**
   - Use VLANs
   - Separate DMZ, internal, management networks
   - Restrict cross-network traffic

6. **Outbound Filtering:**
   - Control outgoing connections
   - Prevent data exfiltration
   - Limit egress to known services

7. **Documentation:**
   - Document firewall rules
   - Maintain IP whitelist/blacklist
   - Keep change log

Master firewall and SSH security for robust network defense!
