# Network Commands

## Interface Management

### ip command (Modern)

```bash
# View all interfaces
ip link show
ip link

# View IP addresses
ip addr show
ip addr
ip a

# View specific interface
ip addr show eth0
ip link show eth0

# Interface statistics
ip -s link show eth0
ip -s -s link show eth0  # More details
```

**Output fields explained:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    │    │
    │    └─ Interface name
    └────── Interface index

    link/ether 00:1a:2b:3c:4d:5e
               │
               └─ MAC address

    inet 192.168.1.10/24 brd 192.168.1.255
         │                    │
         │                    └─ Broadcast address
         └────────────────────── IP address/netmask
```

### ifconfig (Legacy)

```bash
# View all interfaces
ifconfig
ifconfig -a  # Including down interfaces

# View specific interface
ifconfig eth0

# Interface statistics
ifconfig -s
```

### Bring Interface Up/Down

```bash
# Using ip
sudo ip link set eth0 up
sudo ip link set eth0 down

# Using ifconfig
sudo ifconfig eth0 up
sudo ifconfig eth0 down

# Using ifup/ifdown
sudo ifup eth0
sudo ifdown eth0
```

### Assign IP Address

```bash
# Assign IP (temporary)
sudo ip addr add 192.168.1.10/24 dev eth0

# Add secondary IP
sudo ip addr add 192.168.1.11/24 dev eth0

# Remove IP
sudo ip addr del 192.168.1.10/24 dev eth0

# Flush all IPs on interface
sudo ip addr flush dev eth0

# Using ifconfig (legacy)
sudo ifconfig eth0 192.168.1.10 netmask 255.255.255.0
```

### Change MAC Address

```bash
# View current MAC
ip link show eth0 | grep link/ether

# Change MAC (temporary)
sudo ip link set eth0 down
sudo ip link set eth0 address 00:11:22:33:44:55
sudo ip link set eth0 up

# Using ifconfig
sudo ifconfig eth0 down
sudo ifconfig eth0 hw ether 00:11:22:33:44:55
sudo ifconfig eth0 up
```

### Set MTU

```bash
# View current MTU
ip link show eth0 | grep mtu

# Change MTU
sudo ip link set eth0 mtu 9000  # Jumbo frames

# Using ifconfig
sudo ifconfig eth0 mtu 9000
```

## Routing

### View Routing Table

```bash
# Modern way
ip route show
ip route
ip r

# Legacy way
route -n
netstat -rn

# Default route only
ip route show default
```

### Add/Delete Routes

```bash
# Add default gateway
sudo ip route add default via 192.168.1.1
sudo ip route add default via 192.168.1.1 dev eth0

# Add static route
sudo ip route add 10.0.0.0/8 via 192.168.1.254
sudo ip route add 172.16.0.0/16 via 192.168.1.254 dev eth0

# Add route via interface (no gateway)
sudo ip route add 192.168.2.0/24 dev eth1

# Delete route
sudo ip route del 10.0.0.0/8
sudo ip route del default via 192.168.1.1

# Flush routing table
sudo ip route flush all
sudo ip route flush cache
```

### Routing Policy

```bash
# Multiple routing tables
# /etc/iproute2/rt_tables
100 isp1
200 isp2

# Add rule
sudo ip rule add from 192.168.1.0/24 table isp1

# Add route to table
sudo ip route add default via 10.0.1.1 table isp1

# View rules
ip rule list

# View specific table
ip route show table isp1
```

## DNS

### nslookup

```bash
# Basic lookup
nslookup google.com

# Query specific server
nslookup google.com 8.8.8.8

# Interactive mode
nslookup
> server 8.8.8.8
> google.com
> exit

# Query specific record type
nslookup -type=MX google.com
nslookup -type=NS google.com
nslookup -type=TXT google.com
```

### dig (Domain Information Groper)

```bash
# Basic query
dig google.com

# Short answer only
dig google.com +short

# Query specific record type
dig google.com A
dig google.com AAAA
dig google.com MX
dig google.com NS
dig google.com TXT
dig google.com ANY

# Query specific DNS server
dig @8.8.8.8 google.com
dig @1.1.1.1 google.com

# Reverse DNS lookup
dig -x 8.8.8.8
dig -x 142.250.185.46 +short

# Trace DNS path
dig +trace google.com

# Show only answer section
dig google.com +noall +answer

# Batch queries
dig +noall +answer google.com facebook.com twitter.com
```

### host

```bash
# Simple lookup
host google.com

# Query specific type
host -t MX google.com
host -t NS google.com
host -t AAAA google.com

# Reverse lookup
host 8.8.8.8

# Query specific server
host google.com 8.8.8.8

# Verbose output
host -v google.com
```

### resolvectl (systemd-resolved)

```bash
# Show DNS settings
resolvectl status

# Query domain
resolvectl query google.com

# Flush DNS cache
resolvectl flush-caches

# Show statistics
resolvectl statistics
```

## Connectivity Testing

### ping

```bash
# Basic ping
ping google.com
ping 8.8.8.8

# Ping N times
ping -c 4 google.com

# Ping interval (seconds)
ping -i 0.5 google.com  # Every 0.5 seconds

# Ping with specific packet size
ping -s 1024 google.com

# Ping with deadline (timeout)
ping -w 10 google.com  # Stop after 10 seconds

# Flood ping (root only)
sudo ping -f google.com

# Ping specific interface
ping -I eth0 google.com

# IPv6 ping
ping6 google.com

# Don't fragment
ping -M do -s 1472 google.com  # Find MTU
```

### traceroute / tracepath

```bash
# Trace route
traceroute google.com

# Max hops
traceroute -m 20 google.com

# Use ICMP instead of UDP
traceroute -I google.com

# Use TCP
traceroute -T -p 80 google.com

# Tracepath (no root needed)
tracepath google.com
tracepath -n google.com  # No DNS resolution

# IPv6
traceroute6 google.com
```

### mtr (My Traceroute)

```bash
# Interactive mode
mtr google.com

# Report mode (run 10 cycles)
mtr -c 10 -report google.com

# No DNS resolution
mtr -n google.com

# Use TCP instead of ICMP
mtr -T -P 443 google.com

# Wide report
mtr -w google.com

# CSV output
mtr --csv google.com
```

## Port and Service Testing

### netcat (nc)

```bash
# Test TCP connection
nc -zv google.com 80
nc -zv 192.168.1.1 22

# Scan port range
nc -zv google.com 80-443

# Listen on port (server)
nc -l 8080

# Connect to port (client)
nc localhost 8080

# Send file
nc -l 8080 > received_file  # Receiver
nc localhost 8080 < file    # Sender

# UDP mode
nc -u localhost 53

# Timeout
nc -w 5 -zv google.com 80
```

### telnet

```bash
# Test connection
telnet google.com 80

# HTTP request via telnet
telnet google.com 80
GET / HTTP/1.1
Host: google.com
[Enter twice]

# Test SSH
telnet 192.168.1.1 22

# Test SMTP
telnet mail.example.com 25
```

### curl

```bash
# Basic GET request
curl https://google.com

# Show headers only
curl -I https://google.com

# Verbose output
curl -v https://google.com

# Follow redirects
curl -L https://google.com

# Download file
curl -O https://example.com/file.zip

# Custom headers
curl -H "User-Agent: MyApp" https://api.example.com

# POST request
curl -X POST -d "key=value" https://api.example.com

# JSON POST
curl -X POST -H "Content-Type: application/json" \
  -d '{"key":"value"}' https://api.example.com

# Basic auth
curl -u username:password https://api.example.com

# Save cookies
curl -c cookies.txt https://example.com

# Use cookies
curl -b cookies.txt https://example.com

# Timeout
curl --connect-timeout 5 --max-time 10 https://example.com
```

### wget

```bash
# Download file
wget https://example.com/file.zip

# Save as different name
wget -O newname.zip https://example.com/file.zip

# Continue download
wget -c https://example.com/large-file.iso

# Download recursively
wget -r https://example.com/

# Mirror website
wget -m https://example.com/

# Background download
wget -b https://example.com/file.zip

# Limit bandwidth
wget --limit-rate=200k https://example.com/file.zip

# Timeout
wget --timeout=10 https://example.com/file.zip
```

## Network Statistics

### netstat

```bash
# All connections
netstat -a

# TCP connections only
netstat -t

# UDP connections only
netstat -u

# Listening ports
netstat -l
netstat -lt  # TCP listening
netstat -lu  # UDP listening

# Show PID/Program
netstat -p
netstat -tlnp  # Common usage

# Numeric (no DNS)
netstat -n
netstat -an

# Routing table
netstat -r
netstat -rn

# Interface statistics
netstat -i
netstat -ie  # Detailed

# Protocol statistics
netstat -s
netstat -st  # TCP stats
netstat -su  # UDP stats

# Continuous monitoring
netstat -c
```

### ss (Socket Statistics)

```bash
# All sockets
ss -a

# Listening sockets
ss -l
ss -lt  # TCP listening
ss -lu  # UDP listening

# Established connections
ss -t
ss -ta  # All TCP

# Show process
ss -p
ss -tlnp  # Common usage

# Numeric
ss -n

# Summary statistics
ss -s

# Filter by state
ss state established
ss state listening
ss state time-wait

# Filter by port
ss sport = :80
ss dport = :443

# Filter by address
ss dst 192.168.1.0/24
ss src 10.0.0.0/8

# Show timer information
ss -o

# Extended info
ss -e
```

### lsof

```bash
# Network connections
lsof -i

# Specific protocol
lsof -i tcp
lsof -i udp

# Specific port
lsof -i :80
lsof -i :22

# Specific host
lsof -i @192.168.1.1

# TCP state
lsof -i tcp -s TCP:LISTEN

# By user
lsof -u nginx -i

# By process
lsof -p 1234 -i

# Combine options
lsof -u nginx -i tcp:80
```

## ARP (Address Resolution Protocol)

### View ARP Cache

```bash
# View ARP table
arp -a
arp -n  # Numeric

# Modern way
ip neigh show
ip n

# Specific interface
ip neigh show dev eth0
```

### Modify ARP Cache

```bash
# Add static entry
sudo arp -s 192.168.1.100 00:11:22:33:44:55
sudo ip neigh add 192.168.1.100 lladdr 00:11:22:33:44:55 dev eth0

# Delete entry
sudo arp -d 192.168.1.100
sudo ip neigh del 192.168.1.100 dev eth0

# Flush ARP cache
sudo ip neigh flush all
sudo ip neigh flush dev eth0
```

### ARP Scan

```bash
# Scan local network
sudo arp-scan --localnet
sudo arp-scan 192.168.1.0/24

# Specific interface
sudo arp-scan -I eth0 192.168.1.0/24
```

## Network Scanning

### nmap

```bash
# Scan single host
nmap 192.168.1.1

# Scan range
nmap 192.168.1.1-254
nmap 192.168.1.0/24

# Specific ports
nmap -p 80,443 192.168.1.1
nmap -p 1-1000 192.168.1.1
nmap -p- 192.168.1.1  # All ports

# Service version detection
nmap -sV 192.168.1.1

# OS detection
sudo nmap -O 192.168.1.1

# Aggressive scan
sudo nmap -A 192.168.1.1

# Stealth scan
sudo nmap -sS 192.168.1.1

# UDP scan
sudo nmap -sU 192.168.1.1

# Fast scan (top 100 ports)
nmap -F 192.168.1.1

# Output to file
nmap -oN output.txt 192.168.1.1
nmap -oX output.xml 192.168.1.1

# Scan with script
nmap --script=vuln 192.168.1.1
```

## Packet Capture

### tcpdump

```bash
# Capture on interface
sudo tcpdump -i eth0

# Capture N packets
sudo tcpdump -c 100 -i eth0

# Don't resolve hostnames
sudo tcpdump -n -i eth0

# Verbose output
sudo tcpdump -v -i eth0
sudo tcpdump -vv -i eth0
sudo tcpdump -vvv -i eth0

# Save to file
sudo tcpdump -i eth0 -w capture.pcap

# Read from file
tcpdump -r capture.pcap

# Filter by host
sudo tcpdump -i eth0 host 192.168.1.1
sudo tcpdump -i eth0 src 192.168.1.1
sudo tcpdump -i eth0 dst 192.168.1.1

# Filter by port
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 src port 80
sudo tcpdump -i eth0 dst port 443

# Filter by protocol
sudo tcpdump -i eth0 tcp
sudo tcpdump -i eth0 udp
sudo tcpdump -i eth0 icmp

# Complex filters
sudo tcpdump -i eth0 'tcp port 80 and host 192.168.1.1'
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'
sudo tcpdump -i eth0 'icmp[icmptype] = icmp-echo'

# Show packet contents
sudo tcpdump -X -i eth0  # ASCII + Hex
sudo tcpdump -A -i eth0  # ASCII only
```

## Network Configuration

### Network Manager (nmcli)

```bash
# Show all connections
nmcli connection show

# Show devices
nmcli device status

# Connect to WiFi
nmcli dev wifi list
nmcli dev wifi connect SSID password PASSWORD

# Create connection
nmcli connection add type ethernet con-name mycon ifname eth0

# Modify connection
nmcli connection modify mycon ipv4.addresses 192.168.1.10/24
nmcli connection modify mycon ipv4.gateway 192.168.1.1
nmcli connection modify mycon ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify mycon ipv4.method manual

# Activate connection
nmcli connection up mycon

# Deactivate connection
nmcli connection down mycon

# Delete connection
nmcli connection delete mycon
```

### systemd-networkd

```bash
# Status
networkctl status
networkctl list

# Restart networkd
sudo systemctl restart systemd-networkd

# Configuration file
# /etc/systemd/network/20-wired.network
[Match]
Name=eth0

[Network]
DHCP=yes

# Or static IP
[Match]
Name=eth0

[Network]
Address=192.168.1.10/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=8.8.4.4
```

## Bandwidth Monitoring

### iftop

```bash
# Monitor interface
sudo iftop -i eth0

# No hostname resolution
sudo iftop -n -i eth0

# Show ports
sudo iftop -P -i eth0

# Filter by host
sudo iftop -f "host 192.168.1.1" -i eth0

# Filter by port
sudo iftop -f "port 80" -i eth0
```

### nethogs

```bash
# Monitor bandwidth per process
sudo nethogs

# Specific interface
sudo nethogs eth0

# Trace mode (log to file)
sudo nethogs -t eth0 > bandwidth.log
```

### vnstat

```bash
# Initialize database
vnstat -i eth0 --create

# Current stats
vnstat -i eth0

# Live monitor
vnstat -l -i eth0

# Hourly stats
vnstat -h -i eth0

# Daily stats
vnstat -d -i eth0

# Monthly stats
vnstat -m -i eth0

# Top 10 days
vnstat -t -i eth0
```

### bmon

```bash
# Interactive bandwidth monitor
bmon

# Specific interface
bmon -p eth0

# Output format
bmon -o ascii
bmon -o format:fmt='$(element:name) $(attr:rxrate:bytes) $(attr:txrate:bytes)\n'
```

## Firewall Commands

### iptables

```bash
# List rules
sudo iptables -L
sudo iptables -L -n -v

# List with line numbers
sudo iptables -L --line-numbers

# List NAT rules
sudo iptables -t nat -L -n -v

# Allow incoming SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow incoming HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Block IP
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# Delete rule by number
sudo iptables -D INPUT 5

# Flush all rules
sudo iptables -F

# Save rules
sudo iptables-save > /etc/iptables/rules.v4

# Restore rules
sudo iptables-restore < /etc/iptables/rules.v4
```

### ufw (Uncomplicated Firewall)

```bash
# Enable firewall
sudo ufw enable

# Disable firewall
sudo ufw disable

# Status
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered

# Allow service
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Allow port
sudo ufw allow 8080
sudo ufw allow 8000:9000/tcp

# Allow from IP
sudo ufw allow from 192.168.1.0/24

# Allow from IP to specific port
sudo ufw allow from 192.168.1.100 to any port 22

# Deny
sudo ufw deny 23

# Delete rule
sudo ufw delete allow 80
sudo ufw delete 3  # By number

# Reset
sudo ufw reset
```

## Troubleshooting Tools

### ethtool

```bash
# View interface info
ethtool eth0

# Statistics
ethtool -S eth0

# Driver info
ethtool -i eth0

# Speed and duplex
ethtool eth0 | grep -E 'Speed|Duplex'

# Set speed
sudo ethtool -s eth0 speed 1000 duplex full autoneg on
```

### mii-tool

```bash
# Check link status
mii-tool eth0

# Verbose output
mii-tool -v eth0
```

## Tổng kết

**Essential commands by category:**

**Interface Management:**
- ip, ifconfig
- ip link, ip addr

**Routing:**
- ip route, route
- traceroute, mtr

**DNS:**
- dig, nslookup, host
- resolvectl

**Connectivity:**
- ping, telnet, nc
- curl, wget

**Statistics:**
- ss, netstat, lsof
- iftop, nethogs, vnstat

**Scanning:**
- nmap, arp-scan

**Capture:**
- tcpdump, wireshark

**Troubleshooting:**
- ethtool, mii-tool

**NetworkManager:**
- nmcli, nmtui

**Firewall:**
- iptables, ufw

Master these commands for effective network administration!
