# Monitoring Basics

## Monitoring Overview

**Monitoring** là quá trình thu thập, phân tích và hành động dựa trên dữ liệu từ hệ thống.

### **Why Monitoring?**

```
┌─────────────────────────────────────────┐
│       THE MONITORING PYRAMID            │
├─────────────────────────────────────────┤
│   Business Metrics (Revenue, Users)     │  ← What matters to business
├─────────────────────────────────────────┤
│   Application Metrics (Requests, Errors)│  ← What users experience
├─────────────────────────────────────────┤
│   Infrastructure Metrics (CPU, Memory)  │  ← What keeps systems running
├─────────────────────────────────────────┤
│   Logs & Events (Debug, Audit)         │  ← What actually happened
└─────────────────────────────────────────┘
```

**Key Benefits:**
- **Detect issues** before users report them
- **Reduce downtime** with faster incident response
- **Optimize performance** based on data
- **Capacity planning** for scaling
- **Compliance** and audit trails

---

## Monitoring Types

### **1. Infrastructure Monitoring**

**What to Monitor:**
```yaml
# Server/VM Metrics
cpu:
  - usage_percent
  - load_average_1m
  - load_average_5m
  - load_average_15m

memory:
  - used_bytes
  - available_bytes
  - usage_percent
  - swap_used

disk:
  - used_bytes
  - available_bytes
  - io_read_bytes
  - io_write_bytes
  - io_utilization_percent

network:
  - bytes_received
  - bytes_sent
  - packets_dropped
  - connection_count
```

**Example: Node Exporter (Prometheus)**
```bash
# Install Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/

# Create systemd service
sudo tee /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

# Start service
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter

# Verify
curl http://localhost:9100/metrics
```

### **2. Application Monitoring**

**Golden Signals (Google SRE):**
```
┌──────────────────────────────────────────┐
│         THE FOUR GOLDEN SIGNALS          │
├──────────────────────────────────────────┤
│ 1. LATENCY                               │
│    - Request/response time               │
│    - 50th, 95th, 99th percentile        │
│                                          │
│ 2. TRAFFIC                               │
│    - Requests per second (RPS)           │
│    - Concurrent users                    │
│                                          │
│ 3. ERRORS                                │
│    - Error rate (%)                      │
│    - 4xx, 5xx HTTP codes                 │
│                                          │
│ 4. SATURATION                            │
│    - Resource utilization                │
│    - Queue depth, thread pool usage      │
└──────────────────────────────────────────┘
```

**Example: Instrument Node.js App**
```javascript
// app.js
const express = require('express');
const promClient = require('prom-client');

const app = express();

// Create metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const activeConnections = new promClient.Gauge({
  name: 'http_active_connections',
  help: 'Number of active HTTP connections'
});

// Middleware to track metrics
app.use((req, res, next) => {
  const start = Date.now();
  activeConnections.inc();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const labels = {
      method: req.method,
      route: req.route?.path || req.path,
      status_code: res.statusCode
    };

    httpRequestDuration.observe(labels, duration);
    httpRequestTotal.inc(labels);
    activeConnections.dec();
  });

  next();
});

// Business logic endpoints
app.get('/api/users', (req, res) => {
  // ... fetch users
  res.json({ users: [] });
});

app.post('/api/orders', (req, res) => {
  // ... create order
  res.json({ orderId: 123 });
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
  console.log('Metrics available at http://localhost:3000/metrics');
});
```

**Verify Metrics:**
```bash
# Start app
node app.js

# Generate traffic
for i in {1..100}; do
  curl -s http://localhost:3000/api/users > /dev/null
done

# View metrics
curl http://localhost:3000/metrics

# Output:
# http_request_duration_seconds_bucket{method="GET",route="/api/users",status_code="200",le="0.01"} 95
# http_request_duration_seconds_bucket{method="GET",route="/api/users",status_code="200",le="0.05"} 100
# http_request_duration_seconds_sum{method="GET",route="/api/users",status_code="200"} 0.523
# http_request_duration_seconds_count{method="GET",route="/api/users",status_code="200"} 100
#
# http_requests_total{method="GET",route="/api/users",status_code="200"} 100
# http_active_connections 0
```

### **3. Log Aggregation**

**Log Levels:**
```python
# Python logging example
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s - %(message)s'
)

logger = logging.getLogger(__name__)

# Different log levels
logger.debug("Detailed info for debugging")        # DEBUG
logger.info("General informational messages")      # INFO
logger.warning("Warning: something unexpected")    # WARNING
logger.error("Error: operation failed")            # ERROR
logger.critical("Critical: system unstable")       # CRITICAL
```

**Structured Logging (JSON):**
```python
import json
import logging

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'function': record.funcName,
            'line': record.lineno
        }
        
        # Add extra fields
        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id
        
        return json.dumps(log_data)

# Configure
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())

logger = logging.getLogger(__name__)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Usage
logger.info("User logged in", extra={'user_id': 123, 'request_id': 'abc-123'})

# Output:
# {"timestamp": "2026-02-10 10:00:00", "level": "INFO", "logger": "__main__", 
#  "message": "User logged in", "function": "<module>", "line": 35, 
#  "user_id": 123, "request_id": "abc-123"}
```

### **4. Distributed Tracing**

**Trace Context:**
```
┌────────────────────────────────────────────────────────────┐
│              DISTRIBUTED TRACE EXAMPLE                     │
├────────────────────────────────────────────────────────────┤
│  Request: GET /api/orders/123                              │
│  Trace ID: 7f3c8a9b2e4d1f6a                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  [API Gateway]                              Span ID: 001   │
│  Duration: 250ms                                           │
│  ├─ [Auth Service]                          Span ID: 002   │
│  │  Duration: 50ms                                         │
│  │  └─ [Database Query]                     Span ID: 003   │
│  │     Duration: 20ms                                      │
│  │                                                         │
│  ├─ [Order Service]                         Span ID: 004   │
│  │  Duration: 180ms                                        │
│  │  ├─ [Database Query]                     Span ID: 005   │
│  │  │  Duration: 80ms                                      │
│  │  └─ [Payment Service]                    Span ID: 006   │
│  │     Duration: 90ms                                      │
│  │     └─ [External API]                    Span ID: 007   │
│  │        Duration: 60ms                                   │
│  │                                                         │
│  └─ [Notification Service]                  Span ID: 008   │
│     Duration: 30ms (async)                                 │
└────────────────────────────────────────────────────────────┘
```

---

## 📈 Key Metrics

### **RED Method (for Services)**

```yaml
# Rate - Requests per second
http_requests_per_second:
  query: rate(http_requests_total[5m])
  alert_threshold: > 10000

# Errors - Error rate percentage
http_error_rate:
  query: |
    rate(http_requests_total{status=~"5.."}[5m]) / 
    rate(http_requests_total[5m]) * 100
  alert_threshold: > 5  # 5% error rate

# Duration - Response time percentiles
http_request_duration_p95:
  query: histogram_quantile(0.95, http_request_duration_seconds_bucket)
  alert_threshold: > 1.0  # 1 second
```

### **USE Method (for Resources)**

```yaml
# Utilization - Percentage of time resource is busy
cpu_utilization:
  query: 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  alert_threshold: > 80

# Saturation - Amount of work resource cannot service (queued)
disk_io_queue:
  query: node_disk_io_time_weighted_seconds_total
  alert_threshold: > 10

# Errors - Count of error events
disk_read_errors:
  query: rate(node_disk_read_errors_total[5m])
  alert_threshold: > 0
```

---

## 🚨 Alerting Best Practices

### **Alert Severity Levels**

```yaml
alerting:
  severity_levels:
    - name: critical
      description: "Service down, immediate action required"
      notification: "PagerDuty, Phone, SMS"
      response_time: "< 5 minutes"
      examples:
        - "Production service down"
        - "Database unavailable"
        - "Critical data loss"
    
    - name: warning
      description: "Service degraded, investigate soon"
      notification: "Slack, Email"
      response_time: "< 1 hour"
      examples:
        - "High error rate (> 5%)"
        - "High latency (> 1s)"
        - "Disk usage > 80%"
    
    - name: info
      description: "Informational, no action needed"
      notification: "Dashboard only"
      response_time: "Next business day"
      examples:
        - "Deployment completed"
        - "Backup finished"
        - "Certificate renewed"
```

### **Alert Rules Example (Prometheus)**

```yaml
# prometheus-alerts.yml
groups:
  - name: application
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{status=~"5.."}[5m]) /
            rate(http_requests_total[5m])
          ) > 0.05
        for: 5m
        labels:
          severity: warning
          service: "{{ $labels.service }}"
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for service {{ $labels.service }}"
          runbook: "https://wiki.company.com/runbooks/high-error-rate"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High request latency"
          description: "P95 latency is {{ $value }}s"
      
      # Service down
      - alert: ServiceDown
        expr: up{job="myapp"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "Service has been down for more than 1 minute"
      
      # Disk space low
      - alert: DiskSpaceLow
        expr: |
          (
            node_filesystem_avail_bytes{fstype!="tmpfs"} /
            node_filesystem_size_bytes{fstype!="tmpfs"}
          ) < 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Only {{ $value | humanizePercentage }} disk space remaining"

  - name: kubernetes
    interval: 30s
    rules:
      # Pod restarts
      - alert: PodRestarting
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} is restarting"
          description: "Pod has restarted {{ $value }} times in the last 15 minutes"
      
      # Pod crash loop
      - alert: PodCrashLooping
        expr: kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"
          description: "Pod is in CrashLoopBackOff state"
```

---

## SLI, SLO, SLA

### **Definitions**

```
┌─────────────────────────────────────────┐
│   SLI → SLO → SLA                       │
├─────────────────────────────────────────┤
│                                         │
│  SLI (Service Level Indicator)          │
│  Quantitative measure of service level  │
│  Example: "Request latency"             │
│                                         │
│  SLO (Service Level Objective)          │
│  Target value for SLI                   │
│  Example: "95% of requests < 200ms"     │
│                                         │
│  SLA (Service Level Agreement)          │
│  Contract with consequences             │
│  Example: "99.9% uptime or refund"      │
└─────────────────────────────────────────┘
```

### **Common SLIs**

```yaml
# Availability SLI
availability:
  definition: "Percentage of successful requests"
  calculation: |
    (total_requests - failed_requests) / total_requests * 100
  good_events: "HTTP 200-399, 401, 404"
  bad_events: "HTTP 500-599, timeouts"

# Latency SLI
latency:
  definition: "Percentage of requests faster than threshold"
  calculation: |
    count(requests < 200ms) / count(requests) * 100
  threshold: "200ms for 95th percentile"

# Quality SLI
quality:
  definition: "Percentage of correct responses"
  calculation: |
    correct_responses / total_responses * 100
  examples:
    - "Search results accuracy"
    - "Transcription accuracy"
```

### **SLO Example**

```yaml
# Example: API Service SLO
service: api-gateway
period: 30 days

slos:
  - name: availability
    description: "API should be available for most requests"
    sli: (count(http_requests_total{status!~"5.."}) / count(http_requests_total)) * 100
    target: 99.9%  # "three nines"
    error_budget: 0.1%  # 43.2 minutes of downtime per 30 days
    
  - name: latency
    description: "API should respond quickly"
    sli: histogram_quantile(0.95, http_request_duration_seconds_bucket)
    target: "< 200ms"
    threshold: 95%  # 95% of requests
    
  - name: error_rate
    description: "API should have low error rate"
    sli: (count(http_requests_total{status=~"5.."}) / count(http_requests_total)) * 100
    target: "< 1%"

# Error budget calculation
error_budget_calculation: |
  # For 99.9% availability SLO over 30 days:
  total_minutes = 30 * 24 * 60 = 43,200 minutes
  allowed_downtime = 43,200 * 0.001 = 43.2 minutes
  
  # If already had 20 minutes of downtime:
  remaining_budget = 43.2 - 20 = 23.2 minutes
  budget_consumed = 20 / 43.2 = 46.3%
```

**Error Budget Policy:**
```yaml
error_budget_policy:
  when_budget_available:
    - "Deploy new features"
    - "Experiment with new tech"
    - "Planned maintenance"
  
  when_budget_low_50_percent:
    - "Reduce deployment frequency"
    - "Focus on reliability"
    - "Add monitoring"
  
  when_budget_exhausted:
    - "Freeze all deployments"
    - "All hands on stability"
    - "Investigate incidents"
    - "Improve testing"
```

---

## 🎓 Summary

**Key Monitoring Concepts:**

1. **Monitoring Types**: Infrastructure, Application, Logs, Traces
2. **Golden Signals**: Latency, Traffic, Errors, Saturation
3. **Methods**: RED (services), USE (resources)
4. **SLI/SLO/SLA**: Define and measure service quality
5. **Alerting**: Critical, Warning, Info with proper runbooks
6. **Error Budget**: Balance innovation with reliability

**Monitoring Stack:**
- **Metrics**: Prometheus + Grafana
- **Logs**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Traces**: Jaeger, Zipkin, OpenTelemetry
- **APM**: New Relic, Datadog, Dynatrace

**🔗 Next Steps:**
- [Prometheus & Grafana Setup →](../prometheus-grafana/setup.md)
- [ELK Stack Guide →](../../logging/elk_stack.md)
- [Logging Best Practices →](../logging/log-management.md)
