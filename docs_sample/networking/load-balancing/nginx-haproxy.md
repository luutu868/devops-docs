# Load Balancing với Nginx và HAProxy

## Load Balancing Concepts

### Tại sao cần Load Balancer?

**Vấn đề với single server:**
```
Client → Single Server
         (Nếu server down = Service down!)
```

**Giải pháp với Load Balancer:**
```
                    ┌─→ Backend Server 1
Client → Load Balancer ├─→ Backend Server 2
                    ├─→ Backend Server 3
                    └─→ Backend Server 4
```

**Lợi ích:**
- **High Availability**: Nếu 1 server down, traffic chuyển sang server khác
- **Scalability**: Dễ dàng thêm/bớt servers
- **Performance**: Phân tán traffic đều
- **Maintenance**: Zero-downtime updates

### Load Balancing Algorithms

**1. Round Robin (Vòng tròn)**
```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1 (lặp lại)
```
- Phân phối đều requests
- Không quan tâm load hiện tại

**2. Least Connections (Ít kết nối nhất)**
```
Server 1: 5 connections
Server 2: 3 connections  ← Request goes here
Server 3: 7 connections
```
- Gửi request tới server có ít connections nhất
- Tốt cho long-lived connections

**3. IP Hash (Hash theo IP)**
```
Hash(Client IP) % Number of Servers
192.168.1.100 → Always Server 1
192.168.1.101 → Always Server 2
```
- Cùng client luôn đến cùng server
- Tốt cho session persistence

**4. Weighted (Có trọng số)**
```
Server 1: weight=3 (75% traffic)
Server 2: weight=1 (25% traffic)
```
- Phân phối theo capacity của server
- Server mạnh hơn nhận nhiều traffic hơn

**5. Least Response Time**
- Gửi tới server có response time thấp nhất
- Cân nhắc cả connections và latency

## Nginx Load Balancer

### Cài đặt Nginx

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# CentOS/RHEL
sudo yum install epel-release
sudo yum install nginx

# Start service
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Basic Load Balancing

**/etc/nginx/conf.d/load-balancer.conf:**

```nginx
upstream backend {
    # Default: Round robin
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Test and reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Load Balancing Methods

**Least Connections:**
```nginx
upstream backend {
    least_conn;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}
```

**IP Hash:**
```nginx
upstream backend {
    ip_hash;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}
```

**Hash (Generic):**
```nginx
upstream backend {
    hash $request_uri consistent;  # Hash by URL
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

upstream backend_cookie {
    hash $cookie_sessionid;  # Hash by cookie
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

**Random:**
```nginx
upstream backend {
    random two least_conn;  # Pick 2 random, choose least_conn
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}
```

### Server Weights

```nginx
upstream backend {
    server 192.168.1.101:8080 weight=3;  # 60% traffic
    server 192.168.1.102:8080 weight=1;  # 20% traffic
    server 192.168.1.103:8080 weight=1;  # 20% traffic
}
```

### Server Parameters

```nginx
upstream backend {
    server 192.168.1.101:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.102:8080 weight=2;
    server 192.168.1.103:8080 backup;           # Backup server
    server 192.168.1.104:8080 down;             # Temporarily disabled
    server 192.168.1.105:8080 max_conns=100;    # Max connections
}
```

**Parameters:**
- `weight=N`: Server weight (default 1)
- `max_fails=N`: Max failed attempts (default 1)
- `fail_timeout=time`: Time to consider server down (default 10s)
- `backup`: Backup server (used when primaries fail)
- `down`: Permanently disable server
- `max_conns=N`: Max concurrent connections
- `slow_start=time`: Gradually increase weight after recovery

### Health Checks

**Basic (passive) health check:**
```nginx
upstream backend {
    server 192.168.1.101:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.102:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.103:8080 max_fails=3 fail_timeout=30s;
}
```

**Active health check (Nginx Plus):**
```nginx
upstream backend {
    zone backend 64k;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=10s fails=3 passes=2;
    }
}
```

**Custom health check endpoint:**
```nginx
location / {
    proxy_pass http://backend;
    health_check uri=/health interval=5s;
}
```

### Session Persistence (Sticky Sessions)

**IP Hash method:**
```nginx
upstream backend {
    ip_hash;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

**Using cookie (Nginx Plus):**
```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

**Using hash:**
```nginx
upstream backend {
    hash $cookie_jsessionid;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

### SSL/TLS Termination

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### Content-Based Routing

```nginx
upstream api_backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

upstream web_backend {
    server 192.168.1.201:8080;
    server 192.168.1.202:8080;
}

server {
    listen 80;
    server_name example.com;

    location /api/ {
        proxy_pass http://api_backend;
    }

    location / {
        proxy_pass http://web_backend;
    }

    # Route by file extension
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        proxy_pass http://static_backend;
    }
}
```

### Rate Limiting

```nginx
# Define rate limit zone
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        limit_req zone=mylimit burst=20 nodelay;
        proxy_pass http://backend;
    }

    # API rate limiting
    location /api/ {
        limit_req zone=mylimit burst=5 nodelay;
        proxy_pass http://backend;
    }
}
```

### Connection Limits

```nginx
# Limit connections per IP
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    location / {
        limit_conn addr 10;  # Max 10 concurrent connections per IP
        proxy_pass http://backend;
    }
}
```

### Complete Production Example

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'upstream: $upstream_addr '
                    'response_time: $upstream_response_time';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 20M;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # Upstream
    upstream backend_app {
        least_conn;
        server 192.168.1.101:8080 max_fails=3 fail_timeout=30s weight=1;
        server 192.168.1.102:8080 max_fails=3 fail_timeout=30s weight=1;
        server 192.168.1.103:8080 max_fails=3 fail_timeout=30s weight=1;
        server 192.168.1.104:8080 backup;
        
        keepalive 32;
    }

    server {
        listen 80;
        server_name example.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name example.com;

        ssl_certificate /etc/nginx/ssl/example.com.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000" always;

        location / {
            limit_req zone=general burst=20 nodelay;
            limit_conn conn_limit 10;

            proxy_pass http://backend_app;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        location /api/ {
            limit_req zone=api burst=50 nodelay;
            proxy_pass http://backend_app;
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
    }
}
```

## HAProxy Load Balancer

### Cài đặt HAProxy

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install haproxy

# CentOS/RHEL
sudo yum install haproxy

# Start service
sudo systemctl start haproxy
sudo systemctl enable haproxy
```

### HAProxy Configuration Structure

**/etc/haproxy/haproxy.cfg:**

```
global      # Global settings
  ├── defaults  # Default settings
  ├── frontend  # Entry points (listeners)
  ├── backend   # Server pools
  └── listen    # Combined frontend + backend
```

### Basic Configuration

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check
    server web3 192.168.1.103:8080 check
```

Test and restart:
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```

### Load Balancing Algorithms

```haproxy
backend http_back
    # Round robin (default)
    balance roundrobin
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check

backend leastconn_back
    # Least connections
    balance leastconn
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check

backend source_back
    # Source IP hash
    balance source
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check

backend uri_back
    # URI hash
    balance uri
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check

backend hdr_back
    # Header hash
    balance hdr(X-Session-ID)
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check
```

### Server Options

```haproxy
backend http_back
    server web1 192.168.1.101:8080 check weight 100 maxconn 500
    server web2 192.168.1.102:8080 check weight 50 maxconn 300
    server web3 192.168.1.103:8080 check backup
    server web4 192.168.1.104:8080 check disabled

    # Options:
    # check          - Enable health checks
    # weight N       - Server weight (default 1)
    # maxconn N      - Max connections
    # backup         - Backup server
    # disabled       - Temporarily disabled
    # inter N        - Health check interval (default 2s)
    # rise N         - Checks before marking up (default 2)
    # fall N         - Checks before marking down (default 3)
```

### Health Checks

**Basic HTTP check:**
```haproxy
backend http_back
    option httpchk GET /health
    http-check expect status 200
    server web1 192.168.1.101:8080 check inter 5s rise 2 fall 3
    server web2 192.168.1.102:8080 check inter 5s rise 2 fall 3
```

**Advanced HTTP check:**
```haproxy
backend http_back
    option httpchk
    http-check send meth GET uri /health ver HTTP/1.1 hdr Host example.com
    http-check expect status 200
    http-check expect string "OK"
    server web1 192.168.1.101:8080 check
```

**TCP check:**
```haproxy
backend tcp_back
    mode tcp
    option tcp-check
    tcp-check connect port 8080
    server web1 192.168.1.101:8080 check
```

**Custom check script:**
```haproxy
backend http_back
    option external-check
    external-check command /usr/local/bin/check_server.sh
    server web1 192.168.1.101:8080 check
```

### Session Persistence (Sticky Sessions)

**Cookie-based:**
```haproxy
backend http_back
    balance roundrobin
    cookie SERVERID insert indirect nocache
    server web1 192.168.1.101:8080 check cookie web1
    server web2 192.168.1.102:8080 check cookie web2
    server web3 192.168.1.103:8080 check cookie web3
```

**Source IP:**
```haproxy
backend http_back
    balance source
    hash-type consistent
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check
```

**App session cookie:**
```haproxy
backend http_back
    appsession JSESSIONID len 52 timeout 3h request-learn
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check
```

### SSL/TLS Termination

```haproxy
frontend https_front
    bind *:443 ssl crt /etc/haproxy/ssl/example.com.pem
    default_backend http_back

backend http_back
    balance roundrobin
    server web1 192.168.1.101:8080 check
    server web2 192.168.1.102:8080 check
```

**SSL passthrough (no termination):**
```haproxy
frontend https_front
    mode tcp
    bind *:443
    default_backend https_back

backend https_back
    mode tcp
    balance roundrobin
    server web1 192.168.1.101:443 check
    server web2 192.168.1.102:443 check
```

**Redirect HTTP to HTTPS:**
```haproxy
frontend http_front
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/haproxy/ssl/example.com.pem
    default_backend http_back
```

### ACLs (Access Control Lists)

```haproxy
frontend http_front
    bind *:80

    # Define ACLs
    acl is_api path_beg /api/
    acl is_admin path_beg /admin/
    acl is_static path_end .jpg .jpeg .png .gif .css .js
    acl allowed_ips src 203.0.113.0/24 198.51.100.0/24

    # Use ACLs
    use_backend api_back if is_api
    use_backend admin_back if is_admin allowed_ips
    use_backend static_back if is_static
    default_backend web_back

    # Block if not allowed
    http-request deny if is_admin !allowed_ips

backend api_back
    server api1 192.168.1.101:8080 check

backend admin_back
    server admin1 192.168.1.201:8080 check

backend static_back
    server static1 192.168.1.301:8080 check

backend web_back
    server web1 192.168.1.111:8080 check
    server web2 192.168.1.112:8080 check
```

### Rate Limiting

```haproxy
frontend http_front
    bind *:80

    # Track client requests
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src

    # Rate limit: 10 req/10s
    acl too_fast sc_http_req_rate(0) gt 10
    http-request deny if too_fast

    default_backend http_back
```

### Connection Limits

```haproxy
backend http_back
    balance roundrobin
    # Max 500 connections per server
    server web1 192.168.1.101:8080 check maxconn 500
    server web2 192.168.1.102:8080 check maxconn 500

frontend http_front
    bind *:80
    # Max 5000 total connections
    maxconn 5000
    default_backend http_back
```

### Statistics Page

```haproxy
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 5s
    stats auth admin:password
    stats admin if TRUE
```

Access: http://your-server:8404/stats

### Complete Production Example

```haproxy
global
    log /dev/log local0 info
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # SSL
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    # Performance
    maxconn 10000
    tune.ssl.default-dh-param 2048

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  http-server-close
    option  forwardfor except 127.0.0.0/8
    option  redispatch
    retries 3
    timeout connect 5s
    timeout client  50s
    timeout server  50s
    timeout http-request 10s
    timeout http-keep-alive 10s
    timeout queue 30s

frontend http_front
    bind *:80
    # Redirect to HTTPS
    redirect scheme https code 301 if !{ ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/haproxy/ssl/example.com.pem
    
    # Security headers
    http-response set-header X-Frame-Options SAMEORIGIN
    http-response set-header X-Content-Type-Options nosniff
    http-response set-header X-XSS-Protection "1; mode=block"
    http-response set-header Strict-Transport-Security "max-age=31536000"

    # ACLs
    acl is_api path_beg /api/
    acl is_health path /health
    acl is_admin path_beg /admin/
    acl admin_ips src 203.0.113.0/24

    # Rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    acl too_fast sc_http_req_rate(0) gt 50
    http-request deny deny_status 429 if too_fast !is_health

    # Routing
    use_backend api_backend if is_api
    use_backend admin_backend if is_admin admin_ips
    http-request deny if is_admin !admin_ips
    default_backend web_backend

backend web_backend
    balance leastconn
    option httpchk GET /health
    http-check expect status 200
    cookie SERVERID insert indirect nocache
    
    server web1 192.168.1.101:8080 check cookie web1 weight 100 maxconn 500
    server web2 192.168.1.102:8080 check cookie web2 weight 100 maxconn 500
    server web3 192.168.1.103:8080 check cookie web3 weight 50 maxconn 250
    server web4 192.168.1.104:8080 check backup

backend api_backend
    balance roundrobin
    option httpchk GET /api/health
    http-check expect status 200
    
    server api1 192.168.1.201:8080 check inter 5s rise 2 fall 3
    server api2 192.168.1.202:8080 check inter 5s rise 2 fall 3

backend admin_backend
    balance source
    option httpchk GET /admin/health
    
    server admin1 192.168.1.301:8080 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:SecurePassword123
    stats admin if TRUE
```

## High Availability Setup

### keepalived for HA

**Master HAProxy (192.168.1.10):**

```bash
# /etc/keepalived/keepalived.conf
vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    interface eth0
    state MASTER
    virtual_router_id 51
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass SecretPassword
    }

    virtual_ipaddress {
        192.168.1.100/24
    }

    track_script {
        check_haproxy
    }
}
```

**Backup HAProxy (192.168.1.11):**
```bash
# Same config but:
state BACKUP
priority 100
```

Traffic goes to: 192.168.1.100 (Virtual IP)

## Load Balancer Best Practices

1. **Health Checks:**
   - Always enable health checks
   - Use appropriate check intervals
   - Test check endpoints regularly

2. **Session Persistence:**
   - Use sticky sessions when needed
   - Consider stateless architecture
   - Implement shared session storage

3. **SSL/TLS:**
   - Use latest TLS versions (1.2, 1.3)
   - Implement strong ciphers
   - Consider SSL offloading at LB

4. **Monitoring:**
   - Monitor backend health
   - Track response times
   - Set up alerts for failures
   - Use statistics dashboards

5. **Capacity Planning:**
   - Size backends appropriately
   - Use weights for different capacities
   - Keep backup servers ready

6. **Security:**
   - Implement rate limiting
   - Use ACLs for access control
   - Add security headers
   - Filter malicious requests

7. **High Availability:**
   - Deploy multiple load balancers
   - Use keepalived or similar
   - Test failover scenarios

8. **Performance:**
   - Enable keepalive connections
   - Use appropriate timeouts
   - Optimize buffer sizes
   - Enable compression

**Nginx vs HAProxy:**

| Feature | Nginx | HAProxy |
|---------|-------|---------|
| HTTP/HTTPS | Excellent | Excellent |
| TCP/UDP | Good | Excellent |
| Layer 7 LB | Excellent | Excellent |
| Layer 4 LB | Good | Excellent |
| Config complexity | Moderate | Moderate |
| Web server | Yes | No |
| Stats dashboard | Basic | Excellent |
| Active health checks | Plus only | Free |
| SSL termination | Excellent | Excellent |

Choose Nginx for: Web serving + Load balancing
Choose HAProxy for: Pure load balancing, complex routing

Master load balancing for scalable infrastructure!
