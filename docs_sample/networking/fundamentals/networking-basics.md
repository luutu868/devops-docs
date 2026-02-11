# Networking Basics

## Tổng quan về Networking

### Mô hình OSI (7 layers)

```
┌─────────────────────────────────────┐
│ 7. Application (HTTP, FTP, SSH)     │
├─────────────────────────────────────┤
│ 6. Presentation (SSL/TLS, ASCII)    │
├─────────────────────────────────────┤
│ 5. Session (NetBIOS, RPC)           │
├─────────────────────────────────────┤
│ 4. Transport (TCP, UDP)             │
├─────────────────────────────────────┤
│ 3. Network (IP, ICMP, ARP)          │
├─────────────────────────────────────┤
│ 2. Data Link (Ethernet, MAC)        │
├─────────────────────────────────────┤
│ 1. Physical (Cable, Fiber)          │
└─────────────────────────────────────┘
```

### Mô hình TCP/IP (4 layers)

```
┌─────────────────────────────────────┐
│ Application (HTTP, DNS, SSH)        │
├─────────────────────────────────────┤
│ Transport (TCP, UDP)                │
├─────────────────────────────────────┤
│ Internet (IP, ICMP)                 │
├─────────────────────────────────────┤
│ Network Access (Ethernet, WiFi)     │
└─────────────────────────────────────┘
```

## IP Addressing

### IPv4 Addresses

**Format:** 4 octets (32 bits)
```
192.168.1.100
│   │   │ │
│   │   │ └─ Host ID
│   │   └─── Subnet
│   └─────── Network
└─────────── Class
```

**Classes:**

| Class | Range | Default Mask | Usage |
|-------|-------|--------------|-------|
| A | 1.0.0.0 - 126.255.255.255 | 255.0.0.0 (/8) | Large networks |
| B | 128.0.0.0 - 191.255.255.255 | 255.255.0.0 (/16) | Medium networks |
| C | 192.0.0.0 - 223.255.255.255 | 255.255.255.0 (/24) | Small networks |
| D | 224.0.0.0 - 239.255.255.255 | N/A | Multicast |
| E | 240.0.0.0 - 255.255.255.255 | N/A | Experimental |

**Private IP Ranges:**
- Class A: 10.0.0.0 - 10.255.255.255 (10/8)
- Class B: 172.16.0.0 - 172.31.255.255 (172.16/12)
- Class C: 192.168.0.0 - 192.168.255.255 (192.168/16)

**Special Addresses:**
- Loopback: 127.0.0.1
- Broadcast: x.x.x.255
- Network: x.x.x.0

### Subnetting

**CIDR Notation:**
```
192.168.1.0/24
           │
           └─ Subnet mask bits
```

**Common Subnet Masks:**

| CIDR | Mask | Hosts | Networks |
|------|------|-------|----------|
| /8 | 255.0.0.0 | 16,777,214 | 128 |
| /16 | 255.255.0.0 | 65,534 | 16,384 |
| /24 | 255.255.255.0 | 254 | 65,536 |
| /25 | 255.255.255.128 | 126 | 131,072 |
| /26 | 255.255.255.192 | 62 | 262,144 |
| /27 | 255.255.255.224 | 30 | 524,288 |
| /28 | 255.255.255.240 | 14 | 1,048,576 |
| /30 | 255.255.255.252 | 2 | 4,194,304 |

**Example: 192.168.1.0/26**
```
Network:    192.168.1.0
First Host: 192.168.1.1
Last Host:  192.168.1.62
Broadcast:  192.168.1.63
Hosts:      62
```

**Subnet Calculator:**
```bash
# Number of hosts = 2^(32-prefix) - 2
# /24 = 2^8 - 2 = 254 hosts
# /25 = 2^7 - 2 = 126 hosts
# /26 = 2^6 - 2 = 62 hosts
```

### IPv6 Addresses

**Format:** 8 groups of 4 hex digits (128 bits)
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334

# Shortened (remove leading zeros):
2001:db8:85a3:0:0:8a2e:370:7334

# Further shortened (:: for consecutive zeros):
2001:db8:85a3::8a2e:370:7334
```

**Special IPv6 Addresses:**
- Loopback: ::1
- Unspecified: ::
- Link-local: fe80::/10
- Unique local: fc00::/7
- Multicast: ff00::/8

## Network Protocols

### TCP (Transmission Control Protocol)

**Characteristics:**
- Connection-oriented
- Reliable delivery
- Ordered packets
- Flow control
- Error checking

**3-Way Handshake:**
```
Client                    Server
  │                         │
  ├──── SYN ───────────────>│
  │                         │
  │<──── SYN-ACK ───────────┤
  │                         │
  ├──── ACK ───────────────>│
  │                         │
  │    Connection Established
  │                         │
```

**TCP Header:**
```
0                   16                  31
├───────────────────┼───────────────────┤
│   Source Port     │ Destination Port  │
├───────────────────┴───────────────────┤
│          Sequence Number              │
├───────────────────────────────────────┤
│       Acknowledgment Number           │
├───┬───┬───────────┬───────────────────┤
│Hdr│Res│  Flags    │   Window Size     │
├───┴───┴───────────┼───────────────────┤
│    Checksum       │  Urgent Pointer   │
└───────────────────┴───────────────────┘
```

### UDP (User Datagram Protocol)

**Characteristics:**
- Connectionless
- Unreliable (no guarantee)
- No ordering
- Faster than TCP
- Lower overhead

**Use cases:**
- DNS queries
- Video streaming
- VoIP
- Gaming
- DHCP

**UDP Header:**
```
0                   16                  31
├───────────────────┼───────────────────┤
│   Source Port     │ Destination Port  │
├───────────────────┼───────────────────┤
│     Length        │    Checksum       │
└───────────────────┴───────────────────┘
```

### ICMP (Internet Control Message Protocol)

**Common ICMP Types:**
- Echo Request/Reply (ping)
- Destination Unreachable
- Time Exceeded
- Redirect

**Ping Example:**
```bash
ping google.com
# ICMP Type 8 (Echo Request)
# ICMP Type 0 (Echo Reply)
```

### ARP (Address Resolution Protocol)

**Purpose:** Map IP to MAC address

```
Host A                         Host B
IP: 192.168.1.10              IP: 192.168.1.20
MAC: AA:BB:CC:DD:EE:FF        MAC: ?

1. Broadcast: Who has 192.168.1.20?
2. Host B replies: 192.168.1.20 is at 11:22:33:44:55:66
3. Host A caches MAC address
```

**View ARP cache:**
```bash
arp -a
ip neigh show
```

## DNS (Domain Name System)

### DNS Hierarchy

```
                    Root (.)
                       │
        ┌──────────────┼──────────────┐
       .com           .net           .org
        │              │               │
     example        example         example
        │              │               │
    www/mail       www/ftp         www/blog
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | example.com → 192.168.1.1 |
| AAAA | IPv6 address | example.com → 2001:db8::1 |
| CNAME | Canonical name | www → example.com |
| MX | Mail exchange | mail.example.com |
| NS | Name server | ns1.example.com |
| TXT | Text record | SPF, DKIM records |
| PTR | Reverse DNS | 1.1.168.192.in-addr.arpa |
| SOA | Start of authority | Zone info |

### DNS Query Process

```
1. Client → Local Cache
2. Client → Resolver (ISP DNS)
3. Resolver → Root Server
4. Resolver → TLD Server (.com)
5. Resolver → Authoritative Server (example.com)
6. Response cached and returned
```

**DNS Lookup:**
```bash
# Query A record
dig example.com

# Query specific record
dig example.com MX
dig example.com AAAA

# Query specific DNS server
dig @8.8.8.8 example.com

# Reverse lookup
dig -x 8.8.8.8

# Trace query path
dig +trace example.com
```

## DHCP (Dynamic Host Configuration Protocol)

### DHCP Process (DORA)

```
Client                    DHCP Server
  │                           │
  ├──── DISCOVER (broadcast)─>│
  │                           │
  │<──── OFFER ───────────────┤
  │                           │
  ├──── REQUEST ─────────────>│
  │                           │
  │<──── ACK ─────────────────┤
  │                           │
```

**DHCP Lease Info:**
- IP Address
- Subnet Mask
- Default Gateway
- DNS Servers
- Lease Time

**View DHCP lease:**
```bash
# Linux
dhclient -v
cat /var/lib/dhcp/dhclient.leases

# Release and renew
sudo dhclient -r  # Release
sudo dhclient     # Renew
```

## Routing

### Routing Table

```bash
# View routing table
ip route show
route -n

# Example output:
Destination     Gateway         Genmask         Iface
0.0.0.0         192.168.1.1     0.0.0.0         eth0    # Default route
192.168.1.0     0.0.0.0         255.255.255.0   eth0    # Local network
```

### Default Gateway

**Purpose:** Route traffic to external networks

```bash
# View default gateway
ip route | grep default

# Add default gateway
sudo ip route add default via 192.168.1.1

# Delete default gateway
sudo ip route del default
```

### Static Routes

```bash
# Add static route
sudo ip route add 10.0.0.0/8 via 192.168.1.254

# Add route via specific interface
sudo ip route add 172.16.0.0/16 dev eth1

# Delete static route
sudo ip route del 10.0.0.0/8

# Persistent route (systemd)
# /etc/systemd/network/10-static-route.network
[Route]
Destination=10.0.0.0/8
Gateway=192.168.1.254
```

## Network Address Translation (NAT)

### NAT Types

**1. Static NAT (1-to-1)**
```
Private: 192.168.1.10 ←→ Public: 203.0.113.10
```

**2. Dynamic NAT (Pool)**
```
Private: 192.168.1.0/24 ←→ Public: 203.0.113.10-20
```

**3. PAT (Port Address Translation)**
```
192.168.1.10:5000 ←→ 203.0.113.1:50000
192.168.1.11:5000 ←→ 203.0.113.1:50001
```

### NAT with iptables

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# SNAT (Source NAT)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# DNAT (Destination NAT) - Port forwarding
iptables -t nat -A PREROUTING -p tcp --dport 80 \
  -j DNAT --to-destination 192.168.1.10:80

# View NAT rules
iptables -t nat -L -n -v
```

## VLANs (Virtual LANs)

### VLAN Basics

**Purpose:** Segment network logically

```
Physical Switch
┌──────────────────────────────┐
│ VLAN 10: Sales (Ports 1-8)   │
│ VLAN 20: IT (Ports 9-16)     │
│ VLAN 30: Guest (Ports 17-24) │
└──────────────────────────────┘
```

### VLAN Configuration (Linux)

```bash
# Install vlan package
sudo apt install vlan

# Load 8021q module
sudo modprobe 8021q

# Create VLAN interface
sudo ip link add link eth0 name eth0.10 type vlan id 10

# Assign IP to VLAN
sudo ip addr add 192.168.10.1/24 dev eth0.10

# Bring up VLAN interface
sudo ip link set eth0.10 up

# View VLANs
cat /proc/net/vlan/config
```

## Network Troubleshooting

### Connectivity Test

```bash
# Ping (ICMP)
ping 8.8.8.8
ping -c 4 google.com

# Traceroute
traceroute google.com
mtr google.com  # Better than traceroute

# TCP connectivity
telnet google.com 80
nc -zv google.com 80

# Path MTU discovery
ping -M do -s 1472 google.com
```

### DNS Troubleshooting

```bash
# Test DNS resolution
nslookup google.com
dig google.com
host google.com

# Test specific DNS server
nslookup google.com 8.8.8.8
dig @1.1.1.1 google.com

# Check /etc/resolv.conf
cat /etc/resolv.conf
```

### Port Testing

```bash
# Check if port is open
nc -zv 192.168.1.1 22
telnet 192.168.1.1 22

# Scan ports with nmap
nmap 192.168.1.1
nmap -p 1-1000 192.168.1.1
nmap -sV 192.168.1.1  # Service version detection
```

### Network Interface Issues

```bash
# Check interface status
ip link show
ifconfig -a

# Check interface statistics
ip -s link show eth0
ifconfig eth0

# Check for errors
ethtool -S eth0 | grep error
netstat -i
```

### Packet Capture

```bash
# Capture with tcpdump
tcpdump -i eth0
tcpdump -i eth0 port 80
tcpdump -i eth0 host 192.168.1.1
tcpdump -i eth0 'tcp port 443'

# Save to file
tcpdump -i eth0 -w capture.pcap

# Read from file
tcpdump -r capture.pcap

# Wireshark (GUI)
wireshark
```

## Performance Monitoring

### Bandwidth Monitoring

```bash
# Install tools
sudo apt install iftop nethogs bmon

# iftop (interface traffic)
sudo iftop -i eth0

# nethogs (per-process bandwidth)
sudo nethogs eth0

# bmon (bandwidth monitor)
bmon

# vnstat (network statistics)
vnstat -i eth0
vnstat -h  # Hourly stats
vnstat -d  # Daily stats
```

### Network Statistics

```bash
# netstat
netstat -i      # Interface stats
netstat -s      # Protocol stats
netstat -tulnp  # Listening ports

# ss (modern netstat)
ss -s           # Summary
ss -tulnp       # Listening ports
ss -o state established  # Established connections
```

## Common Network Commands

### Interface Management

```bash
# View interfaces
ip link show
ip addr show

# Up/Down interface
sudo ip link set eth0 up
sudo ip link set eth0 down

# Assign IP
sudo ip addr add 192.168.1.10/24 dev eth0

# Remove IP
sudo ip addr del 192.168.1.10/24 dev eth0
```

### Connection Testing

```bash
# Ping
ping -c 4 8.8.8.8

# Traceroute
traceroute google.com
tracepath google.com

# MTR (My Traceroute)
mtr google.com

# Curl (HTTP)
curl -I https://google.com
curl -v https://google.com
```

### Network Configuration Files

**Ubuntu/Debian:**
```bash
# /etc/network/interfaces
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

**CentOS/RHEL:**
```bash
# /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.10
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=8.8.4.4
```

**Netplan (Ubuntu 18.04+):**
```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.10/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

## Best Practices

### Network Security
- Use private IP ranges internally
- Implement firewall rules
- Disable unused services
- Use VLANs for segmentation
- Enable encryption (TLS/SSL)
- Regular security audits

### Performance
- Monitor bandwidth usage
- Optimize MTU settings
- Use QoS for critical traffic
- Implement caching (DNS, web)
- Load balancing
- Regular performance testing

### Documentation
- Document network topology
- Maintain IP address inventory
- Record configuration changes
- Create runbooks for common issues
- Keep network diagrams updated

### Monitoring
- Set up network monitoring tools
- Configure alerts for issues
- Monitor interface errors
- Track bandwidth trends
- Log network events

## Tổng kết

**Key concepts:**
- IP addressing & subnetting
- TCP/UDP protocols
- DNS & DHCP
- Routing & NAT
- VLANs
- Network troubleshooting

**Essential skills:**
- Configure network interfaces
- Troubleshoot connectivity
- Analyze network traffic
- Implement security best practices
- Monitor network performance

**Tools to master:**
- ip, ifconfig
- ping, traceroute, mtr
- tcpdump, wireshark
- netstat, ss
- nmap, dig, nslookup

Networking knowledge là foundation cho DevOps!
