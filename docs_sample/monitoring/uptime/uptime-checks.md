# Uptime Monitoring

## 🎯 Uptime Monitoring Overview

**Uptime Monitoring** = kiểm tra liên tục xem service có đang hoạt động hay không.

```
┌────────────────────────────────────────────┐
│      UPTIME MONITORING WORKFLOW            │
├────────────────────────────────────────────┤
│                                            │
│  [Monitor] ──► HTTP GET ──► [Your API]    │
│      ↓                          │          │
│  Every 1-5 min                  │          │
│      ↓                          ▼          │
│  Check Response          200 OK / Error    │
│      ↓                          │          │
│  Record Status                  │          │
│      ↓                          │          │
│  Alert if Down ────────────────┘           │
│                                            │
└────────────────────────────────────────────┘
```

**Key Metrics:**
- ✅ **Uptime %**: 99.9% = "three nines" = 43 minutes downtime/month
- ✅ **Response Time**: Average, P95, P99 latency
- ✅ **Availability**: Success rate of health checks
- ✅ **Geographic Coverage**: Multi-region monitoring

---

## 📊 Uptime Percentage (SLA)

### **Availability Table**

| Availability | Downtime/Year | Downtime/Month | Downtime/Week | Downtime/Day |
|--------------|---------------|----------------|---------------|--------------|
| **90%** (one nine) | 36.5 days | 72 hours | 16.8 hours | 2.4 hours |
| **99%** (two nines) | 3.65 days | 7.2 hours | 1.68 hours | 14.4 min |
| **99.9%** (three nines) | 8.76 hours | 43.2 min | 10.1 min | 1.44 min |
| **99.99%** (four nines) | 52.6 min | 4.32 min | 1.01 min | 8.64 sec |
| **99.999%** (five nines) | 5.26 min | 25.9 sec | 6.05 sec | 0.86 sec |

**Example:**
```
99.9% uptime = 43.2 minutes downtime per 30 days
= 8.76 hours downtime per year

If your API was down:
- 1 hour in January
- 2 hours in March  
- 3 hours in June
- 1 hour in September

Total = 7 hours/year = 99.92% uptime ✅
```

---

## 🔍 Health Check Endpoints

### **1. Basic Health Check**

```javascript
// Node.js Express
const express = require('express');
const app = express();

app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'ok',
    timestamp: new Date().toISOString()
  });
});

app.listen(3000);
```

### **2. Comprehensive Health Check**

```javascript
app.get('/health', async (req, res) => {
  const checks = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.APP_VERSION || '1.0.0',
    checks: {}
  };

  // Check database
  try {
    await db.query('SELECT 1');
    checks.checks.database = { status: 'ok' };
  } catch (error) {
    checks.status = 'error';
    checks.checks.database = {
      status: 'error',
      error: error.message
    };
  }

  // Check Redis
  try {
    await redis.ping();
    checks.checks.redis = { status: 'ok' };
  } catch (error) {
    checks.status = 'error';
    checks.checks.redis = {
      status: 'error',
      error: error.message
    };
  }

  // Check disk space
  const diskUsage = await checkDiskUsage();
  if (diskUsage > 90) {
    checks.status = 'warning';
    checks.checks.disk = {
      status: 'warning',
      usage: `${diskUsage}%`
    };
  } else {
    checks.checks.disk = { status: 'ok', usage: `${diskUsage}%` };
  }

  const statusCode = checks.status === 'ok' ? 200 : 503;
  res.status(statusCode).json(checks);
});
```

### **3. Kubernetes Liveness & Readiness**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:1.0.0
          ports:
            - containerPort: 3000
          
          # Liveness: Is container alive?
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Readiness: Is container ready to serve traffic?
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
```

**Implementation:**
```javascript
// Liveness: Simple check (is app running?)
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// Readiness: Check dependencies (can app serve traffic?)
app.get('/health/ready', async (req, res) => {
  try {
    // Check database connection
    await db.query('SELECT 1');
    
    // Check Redis connection
    await redis.ping();
    
    // All checks passed
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    // Not ready to serve traffic
    res.status(503).json({
      status: 'not_ready',
      error: error.message
    });
  }
});
```

---

## 🛠️ Uptime Monitoring Tools

### **1. Prometheus Blackbox Exporter**

```yaml
# docker-compose.yml
version: '3.8'

services:
  blackbox-exporter:
    image: prom/blackbox-exporter:v0.24.0
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox.yml:/etc/blackbox_exporter/config.yml
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:v2.48.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

**Blackbox Configuration:**
```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: "ip4"
  
  http_post:
    prober: http
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"check": "health"}'
  
  tcp_connect:
    prober: tcp
    timeout: 5s
  
  icmp:
    prober: icmp
    timeout: 5s
  
  dns:
    prober: dns
    dns:
      query_name: "example.com"
      query_type: "A"
```

**Prometheus Configuration:**
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com/health
          - https://app.example.com/status
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

**Alert Rules:**
```yaml
# alerts.yml
groups:
  - name: uptime
    interval: 30s
    rules:
      - alert: EndpointDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been unreachable for 2 minutes"
      
      - alert: SlowResponse
        expr: probe_http_duration_seconds > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow response from {{ $labels.instance }}"
          description: "Response time is {{ $value }}s (threshold: 5s)"
      
      - alert: SSLCertificateExpiring
        expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate expiring soon"
          description: "SSL certificate for {{ $labels.instance }} expires in {{ $value | humanizeDuration }}"
```

### **2. UptimeRobot (Free Tier)**

```bash
# UptimeRobot API
curl -X POST https://api.uptimerobot.com/v2/newMonitor \
  -H "Content-Type: application/json" \
  -d '{
    "api_key": "your-api-key",
    "friendly_name": "My Website",
    "url": "https://example.com",
    "type": 1,
    "interval": 300
  }'

# Monitor Types:
# 1 = HTTP(s)
# 2 = Keyword
# 3 = Ping
# 4 = Port

# Get monitor status
curl -X POST https://api.uptimerobot.com/v2/getMonitors \
  -H "Content-Type: application/json" \
  -d '{"api_key": "your-api-key"}'
```

### **3. Status Page**

```javascript
// Simple status page with Node.js
const express = require('express');
const axios = require('axios');

const app = express();

// Endpoints to monitor
const endpoints = [
  { name: 'Website', url: 'https://example.com', type: 'http' },
  { name: 'API', url: 'https://api.example.com/health', type: 'http' },
  { name: 'Database', host: 'localhost', port: 5432, type: 'tcp' }
];

// Check endpoint status
async function checkEndpoint(endpoint) {
  try {
    if (endpoint.type === 'http') {
      const start = Date.now();
      const response = await axios.get(endpoint.url, { timeout: 5000 });
      const latency = Date.now() - start;
      
      return {
        status: 'operational',
        latency: latency,
        statusCode: response.status
      };
    } else if (endpoint.type === 'tcp') {
      // TCP check implementation
      const net = require('net');
      const start = Date.now();
      
      return new Promise((resolve) => {
        const socket = new net.Socket();
        socket.setTimeout(5000);
        
        socket.on('connect', () => {
          const latency = Date.now() - start;
          socket.destroy();
          resolve({ status: 'operational', latency });
        });
        
        socket.on('timeout', () => {
          socket.destroy();
          resolve({ status: 'degraded', error: 'Timeout' });
        });
        
        socket.on('error', (err) => {
          resolve({ status: 'down', error: err.message });
        });
        
        socket.connect(endpoint.port, endpoint.host);
      });
    }
  } catch (error) {
    return {
      status: 'down',
      error: error.message
    };
  }
}

// Status page route
app.get('/status', async (req, res) => {
  const results = await Promise.all(
    endpoints.map(async (endpoint) => {
      const status = await checkEndpoint(endpoint);
      return {
        name: endpoint.name,
        ...status
      };
    })
  );
  
  // Calculate overall status
  const allOperational = results.every(r => r.status === 'operational');
  const anyDown = results.some(r => r.status === 'down');
  
  const overallStatus = anyDown ? 'down' : allOperational ? 'operational' : 'degraded';
  
  res.json({
    status: overallStatus,
    updated: new Date().toISOString(),
    services: results
  });
});

// HTML status page
app.get('/', async (req, res) => {
  const results = await Promise.all(
    endpoints.map(async (endpoint) => {
      const status = await checkEndpoint(endpoint);
      return {
        name: endpoint.name,
        ...status
      };
    })
  );
  
  const html = `
    <!DOCTYPE html>
    <html>
    <head>
      <title>System Status</title>
      <style>
        body { font-family: Arial, sans-serif; margin: 50px; }
        .service { padding: 20px; margin: 10px 0; border-radius: 5px; }
        .operational { background: #d4edda; }
        .degraded { background: #fff3cd; }
        .down { background: #f8d7da; }
      </style>
    </head>
    <body>
      <h1>System Status</h1>
      ${results.map(r => `
        <div class="service ${r.status}">
          <h3>${r.name}</h3>
          <p>Status: ${r.status.toUpperCase()}</p>
          ${r.latency ? `<p>Latency: ${r.latency}ms</p>` : ''}
          ${r.error ? `<p>Error: ${r.error}</p>` : ''}
        </div>
      `).join('')}
      <p>Last updated: ${new Date().toLocaleString()}</p>
    </body>
    </html>
  `;
  
  res.send(html);
});

app.listen(3000, () => {
  console.log('Status page: http://localhost:3000');
});
```

---

## 📊 Multi-Region Monitoring

### **Global Monitoring Setup**

```
┌──────────────────────────────────────────────────┐
│       MULTI-REGION MONITORING                    │
├──────────────────────────────────────────────────┤
│                                                  │
│  [US-East Monitor] ──┐                           │
│  [US-West Monitor] ──┤                           │
│  [EU Monitor]      ──┼──► [Your API] ◄─── Users │
│  [Asia Monitor]    ──┤                           │
│  [AU Monitor]      ──┘                           │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Benefits:**
- Detect regional outages
- Measure global latency
- Ensure CDN is working
- Geographic performance insights

### **Using AWS CloudWatch Synthetics**

```javascript
// CloudWatch Canary script
const synthetics = require('Synthetics');
const log = require('SyntheticsLogger');

const apiCanaryBlueprint = async function () {
  // Test homepage
  let page = await synthetics.getPage();
  const response = await page.goto('https://example.com', {
    waitUntil: 'networkidle0',
    timeout: 30000
  });
  
  // Verify response
  if (!response || response.status() !== 200) {
    throw new Error(`Failed to load page: ${response.status()}`);
  }
  
  log.info('Page loaded successfully');
  
  // Test API endpoint
  const apiResponse = await page.goto('https://api.example.com/health');
  if (apiResponse.status() !== 200) {
    throw new Error(`API health check failed: ${apiResponse.status()}`);
  }
  
  // Parse JSON response
  const json = await apiResponse.json();
  if (json.status !== 'ok') {
    throw new Error(`API unhealthy: ${JSON.stringify(json)}`);
  }
  
  log.info('API health check passed');
};

exports.handler = async () => {
  return await apiCanaryBlueprint();
};
```

---

## 🔔 Alerting Strategies

### **Alert Routing**

```yaml
alert_routing:
  # Critical: Service completely down
  critical:
    condition: "probe_success == 0 for 2 minutes"
    notify:
      - PagerDuty (phone call)
      - Slack #incidents
      - On-call engineer
    escalation:
      - 5 min: Team lead
      - 15 min: Engineering manager
  
  # Warning: Service degraded
  warning:
    condition: "probe_duration > 5s for 10 minutes"
    notify:
      - Slack #alerts
      - Email to team
    no_escalation: true
  
  # Info: Certificate expiring
  info:
    condition: "ssl_cert_expiry < 30 days"
    notify:
      - Email to DevOps team
    schedule: "09:00-17:00 weekdays"
```

### **Alert Fatigue Prevention**

```yaml
best_practices:
  # 1. Alert on symptoms, not causes
  good_alert: "API returning 500 errors"
  bad_alert: "CPU usage > 80%"
  
  # 2. Use appropriate thresholds
  transient_issues:
    check_interval: 1 minute
    alert_threshold: 3 consecutive failures
  
  # 3. Group related alerts
  single_alert: "Service X is down"
  not_multiple: 
    - "Health check failed"
    - "High error rate"
    - "No traffic"
  
  # 4. Include runbook links
  alert_message: |
    Service X is down
    Runbook: https://wiki.company.com/runbooks/service-x-down
    Dashboard: https://grafana.company.com/d/service-x
```

---

## 📈 Uptime Reporting

### **Calculate Uptime**

```python
from datetime import datetime, timedelta
import statistics

def calculate_uptime(checks):
    """
    Calculate uptime percentage from health check data
    
    checks = [
        {'timestamp': '2026-01-01 00:00:00', 'status': 'up'},
        {'timestamp': '2026-01-01 00:05:00', 'status': 'up'},
        {'timestamp': '2026-01-01 00:10:00', 'status': 'down'},
        ...
    ]
    """
    total_checks = len(checks)
    successful_checks = sum(1 for c in checks if c['status'] == 'up')
    
    uptime_percentage = (successful_checks / total_checks) * 100
    
    # Calculate downtime duration
    downtime_minutes = 0
    current_downtime_start = None
    
    for i, check in enumerate(checks):
        if check['status'] == 'down' and current_downtime_start is None:
            current_downtime_start = check['timestamp']
        elif check['status'] == 'up' and current_downtime_start is not None:
            # Downtime ended
            downtime_end = check['timestamp']
            duration = (downtime_end - current_downtime_start).total_seconds() / 60
            downtime_minutes += duration
            current_downtime_start = None
    
    return {
        'uptime_percentage': round(uptime_percentage, 3),
        'total_checks': total_checks,
        'successful_checks': successful_checks,
        'failed_checks': total_checks - successful_checks,
        'downtime_minutes': round(downtime_minutes, 2)
    }

# Calculate response time statistics
def calculate_latency_stats(checks):
    latencies = [c['latency_ms'] for c in checks if c.get('latency_ms')]
    
    return {
        'avg_latency': round(statistics.mean(latencies), 2),
        'p50_latency': round(statistics.median(latencies), 2),
        'p95_latency': round(statistics.quantiles(latencies, n=20)[18], 2),
        'p99_latency': round(statistics.quantiles(latencies, n=100)[98], 2),
        'max_latency': max(latencies),
        'min_latency': min(latencies)
    }
```

### **Monthly Report**

```python
def generate_monthly_report(service_name, checks):
    """Generate uptime report for the past month"""
    uptime_stats = calculate_uptime(checks)
    latency_stats = calculate_latency_stats(checks)
    
    report = f"""
    ═════════════════════════════════════════════════
    Monthly Uptime Report - {service_name}
    Period: {datetime.now().strftime('%B %Y')}
    ═════════════════════════════════════════════════
    
    UPTIME METRICS
    ─────────────────────────────────────────────────
    Uptime:              {uptime_stats['uptime_percentage']}%
    Total Checks:        {uptime_stats['total_checks']:,}
    Successful:          {uptime_stats['successful_checks']:,}
    Failed:              {uptime_stats['failed_checks']}
    Total Downtime:      {uptime_stats['downtime_minutes']} minutes
    
    LATENCY METRICS
    ─────────────────────────────────────────────────
    Average:             {latency_stats['avg_latency']} ms
    Median (P50):        {latency_stats['p50_latency']} ms
    P95:                 {latency_stats['p95_latency']} ms
    P99:                 {latency_stats['p99_latency']} ms
    Max:                 {latency_stats['max_latency']} ms
    
    SLA COMPLIANCE
    ─────────────────────────────────────────────────
    Target SLA:          99.9%
    Actual:              {uptime_stats['uptime_percentage']}%
    Status:              {'✅ MET' if uptime_stats['uptime_percentage'] >= 99.9 else '❌ MISSED'}
    
    ═════════════════════════════════════════════════
    """
    
    return report
```

---

## 🎓 Summary

**✅ Uptime Monitoring Best Practices:**

1. **Multiple Check Locations**: Monitor from different geographic regions
2. **Appropriate Intervals**: 1-5 minutes for critical services
3. **Comprehensive Health Checks**: Test dependencies (DB, Redis, etc.)
4. **Alert Fatigue Prevention**: Alert on real issues, not noise
5. **Status Page**: Keep users informed during incidents
6. **SLA Tracking**: Monitor and report uptime percentage
7. **Runbooks**: Include recovery steps in alerts

**📊 Key Metrics:**
- Uptime percentage (99.9% = 43 min downtime/month)
- Response time (avg, P95, P99)
- Check frequency and success rate
- Geographic latency distribution

**🛠️ Recommended Tools:**
- **Open Source**: Prometheus Blackbox Exporter
- **Free Tier**: UptimeRobot, StatusCake
- **Paid**: Pingdom, Datadog, New Relic
- **Cloud Native**: AWS CloudWatch Synthetics, GCP Uptime Checks

**🔗 Related Documentation:**
- [Monitoring Basics ←](../fundamentals/monitoring-basics.md)
- [Prometheus & Grafana ←](../prometheus-grafana/setup.md)
- [APM Tools ←](../apm/application-monitoring.md)
