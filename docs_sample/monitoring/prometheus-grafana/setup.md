# Prometheus & Grafana

## Overview

**Prometheus** = Metrics collection & storage  
**Grafana** = Visualization & dashboards

```
┌─────────────────────────────────────────────────────┐
│          PROMETHEUS + GRAFANA STACK                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Applications] ──┐                                 │
│  [Node Exporter] ─┤                                 │
│  [cAdvisor]      ─┼──► [Prometheus] ──► [Grafana]  │
│  [Blackbox]      ─┤      │                          │
│  [Custom]        ─┘      │                          │
│                          ▼                          │
│                    [Alertmanager] ──► Notifications │
└─────────────────────────────────────────────────────┘
```

---

##  Prometheus Setup

### **1. Docker Compose Installation**

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    restart: unless-stopped
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    ports:
      - "9100:9100"
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    restart: unless-stopped
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk:/dev/disk:ro
    devices:
      - /dev/kmsg
    privileged: true
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.3
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus_data: {}
  alertmanager_data: {}
  grafana_data: {}

networks:
  monitoring:
    driver: bridge
```

### **2. Prometheus Configuration**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s      # Scrape targets every 15s
  evaluation_interval: 15s  # Evaluate rules every 15s
  external_labels:
    cluster: 'production'
    environment: 'prod'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load alerting rules
rule_files:
  - "alerts.yml"

# Scrape configurations
scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (system metrics)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          instance: 'server-01'

  # cAdvisor (container metrics)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # Application metrics
  - job_name: 'myapp'
    static_configs:
      - targets: ['myapp:3000']
        labels:
          service: 'api'
          env: 'production'
    metrics_path: '/metrics'
    scrape_interval: 10s

  # Kubernetes service discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use custom port if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}:${__meta_kubernetes_pod_annotation_prometheus_io_port}
      # Add pod labels
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: kubernetes_pod_name

  # Blackbox exporter (endpoint probing)
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### **3. Start Stack**

```bash
# Create directories
mkdir -p prometheus grafana/{dashboards,datasources}

# Create configs
cat > alertmanager.yml << EOF
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'slack'

receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
EOF

# Start services
docker-compose up -d

# Verify
docker-compose ps
curl http://localhost:9090/-/healthy   # Prometheus
curl http://localhost:3000/api/health  # Grafana
```

### **4. Prometheus Queries (PromQL)**

**Basic Queries:**
```promql
# Simple metric query
up

# Filter by label
up{job="node-exporter"}

# Regex matching
http_requests_total{status=~"5.."}

# Range vector (last 5 minutes)
http_requests_total[5m]

# Rate function (per-second average)
rate(http_requests_total[5m])

# Sum by label
sum(rate(http_requests_total[5m])) by (method, status)
```

**Advanced Queries:**
```promql
# CPU usage percentage
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage percentage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100

# HTTP error rate (%)
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# Request latency (95th percentile)
histogram_quantile(0.95, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# Requests per second
sum(rate(http_requests_total[1m]))

# Top 5 slowest endpoints
topk(5, 
  histogram_quantile(0.95, 
    sum(rate(http_request_duration_seconds_bucket[5m])) by (endpoint, le)
  )
)

# Predict disk full (linear regression)
predict_linear(
  node_filesystem_avail_bytes{mountpoint="/"}[1h], 
  4 * 3600  # 4 hours
) < 0
```

**Aggregation Operators:**
```promql
# sum - Total across all dimensions
sum(http_requests_total)

# avg - Average
avg(node_cpu_seconds_total)

# min, max
min(http_request_duration_seconds)
max(http_request_duration_seconds)

# count - Number of time series
count(up == 1)

# topk, bottomk - Top/bottom N
topk(10, http_requests_total)

# quantile - Calculate quantiles
quantile(0.95, http_request_duration_seconds)
```

---

## Grafana Setup

### **1. Datasource Configuration**

```yaml
# grafana/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
      httpMethod: POST
```

### **2. Dashboard Provisioning**

```yaml
# grafana/dashboards/dashboard.yml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

### **3. Create Dashboard (JSON)**

```json
{
  "dashboard": {
    "title": "Application Metrics",
    "uid": "app-metrics",
    "tags": ["application", "monitoring"],
    "timezone": "browser",
    "refresh": "10s",
    
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (method)",
            "legendFormat": "{{ method }}"
          }
        ],
        "yaxes": [
          {"format": "reqps", "label": "Requests/sec"},
          {"format": "short"}
        ]
      },
      
      {
        "id": 2,
        "title": "Error Rate",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
            "legendFormat": "Error Rate %"
          }
        ],
        "alert": {
          "conditions": [
            {
              "type": "query",
              "query": {"params": ["A", "5m", "now"]},
              "reducer": {"type": "avg"},
              "evaluator": {"type": "gt", "params": [5]}
            }
          ],
          "frequency": "60s",
          "handler": 1,
          "name": "High Error Rate",
          "message": "Error rate is above 5%"
        }
      },
      
      {
        "id": 3,
        "title": "Response Time (P95)",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))",
            "legendFormat": "{{ endpoint }}"
          }
        ]
      },
      
      {
        "id": 4,
        "title": "Active Connections",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 12, "y": 8},
        "targets": [
          {
            "expr": "http_active_connections"
          }
        ],
        "options": {
          "colorMode": "background",
          "graphMode": "area",
          "orientation": "auto"
        }
      },
      
      {
        "id": 5,
        "title": "CPU Usage",
        "type": "gauge",
        "gridPos": {"h": 4, "w": 6, "x": 18, "y": 8},
        "targets": [
          {
            "expr": "100 - (avg(irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"
          }
        ],
        "options": {
          "showThresholdLabels": false,
          "showThresholdMarkers": true
        },
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                {"value": 0, "color": "green"},
                {"value": 70, "color": "yellow"},
                {"value": 90, "color": "red"}
              ]
            },
            "max": 100,
            "unit": "percent"
          }
        }
      }
    ],
    
    "templating": {
      "list": [
        {
          "name": "environment",
          "type": "query",
          "query": "label_values(up, environment)",
          "current": {"value": "production"}
        }
      ]
    }
  }
}
```

### **4. Import Popular Dashboards**

```bash
# Access Grafana UI
open http://localhost:3000

# Login: admin/admin123

# Import dashboards by ID:
# 1. Go to Dashboards → Import
# 2. Enter dashboard ID:

# Node Exporter Full
# ID: 1860
# https://grafana.com/grafana/dashboards/1860

# Docker Dashboard
# ID: 193
# https://grafana.com/grafana/dashboards/193

# Kubernetes Cluster Monitoring
# ID: 7249
# https://grafana.com/grafana/dashboards/7249
```

---

## 🚨 Alerting

### **1. Alert Rules**

```yaml
# alerts.yml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m])) /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High HTTP error rate"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
          dashboard: "https://grafana.example.com/d/app-metrics"
          runbook: "https://wiki.example.com/runbooks/high-error-rate"
      
      # Service down
      - alert: ServiceDown
        expr: up{job="myapp"} == 0
        for: 1m
        labels:
          severity: critical
          team: sre
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "Service has been unavailable for more than 1 minute"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1.0
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High request latency"
          description: "P95 latency is {{ $value }}s (threshold: 1s)"
      
      # Disk space low
      - alert: DiskSpaceLow
        expr: |
          (
            node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} /
            node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}
          ) < 0.1
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanizePercentage }} remaining on {{ $labels.mountpoint }}"

  - name: kubernetes_alerts
    interval: 30s
    rules:
      # Pod crash looping
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"
          description: "Pod has restarted {{ $value }} times in 15 minutes"
      
      # High memory usage
      - alert: PodMemoryHigh
        expr: |
          (
            container_memory_usage_bytes{pod!=""} /
            container_spec_memory_limit_bytes{pod!=""} * 100
          ) > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage in pod {{ $labels.pod }}"
          description: "Memory usage is {{ $value }}%"
```

### **2. Alertmanager Configuration**

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  
  routes:
    # Critical alerts → PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    
    - match:
        severity: critical
      receiver: 'slack-critical'
    
    # Warning alerts → Slack only
    - match:
        severity: warning
      receiver: 'slack-warnings'
    
    # Team-specific routing
    - match:
        team: backend
      receiver: 'slack-backend'
    
    - match:
        team: infrastructure
      receiver: 'slack-infrastructure'

# Inhibit rules (suppress alerts)
inhibit_rules:
  # If service is down, don't alert on high latency
  - source_match:
      severity: 'critical'
      alertname: 'ServiceDown'
    target_match:
      severity: 'warning'
      alertname: 'HighLatency'
    equal: ['instance']

# Receivers
receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
  
  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        color: 'danger'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Runbook:* {{ .Annotations.runbook }}
          {{ end }}
  
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        color: 'warning'
        title: '⚠️  WARNING: {{ .GroupLabels.alertname }}'
  
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}'
        details:
          severity: '{{ .CommonLabels.severity }}'
          description: '{{ .CommonAnnotations.description }}'
  
  - name: 'slack-backend'
    slack_configs:
      - channel: '#team-backend'
  
  - name: 'slack-infrastructure'
    slack_configs:
      - channel: '#team-infrastructure'
```

### **3. Test Alerts**

```bash
# Reload Prometheus config
curl -X POST http://localhost:9090/-/reload

# Check alert rules
curl http://localhost:9090/api/v1/rules | jq

# Check active alerts
curl http://localhost:9090/api/v1/alerts | jq

# Send test alert to Alertmanager
curl -XPOST http://localhost:9093/api/v1/alerts -d '[
  {
    "labels": {
      "alertname": "TestAlert",
      "severity": "warning"
    },
    "annotations": {
      "summary": "This is a test alert",
      "description": "Testing Alertmanager configuration"
    }
  }
]'

# Check Alertmanager status
curl http://localhost:9093/api/v1/status | jq
```

---

##  Kubernetes Deployment

### **Prometheus Operator**

```bash
# Install using Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=admin123

# Verify
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

### **ServiceMonitor for Applications**

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-metrics
  namespace: monitoring
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
      - default
      - production
```

### **PrometheusRule for Alerts**

```yaml
# prometheusrule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: myapp
      interval: 30s
      rules:
        - alert: HighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[5m])) /
              sum(rate(http_requests_total[5m]))
            ) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate in {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }}"
```

---

## 🎓 Summary

**Prometheus:**
- Time-series database for metrics
- Pull-based scraping model
- PromQL query language
- Built-in alerting

**Grafana:**
- Visualization and dashboards
- Multiple datasource support
- Alerting and notifications
- Template variables for dynamic dashboards

**Complete Stack:**
- Prometheus: Metrics storage
- Alertmanager: Alert routing
- Node Exporter: System metrics
- cAdvisor: Container metrics
- Grafana: Visualization

**Key Commands:**
```bash
# Prometheus
curl http://localhost:9090/api/v1/query?query=up
curl -X POST http://localhost:9090/-/reload

# Grafana
curl http://localhost:3000/api/health

# Docker Compose
docker-compose logs -f prometheus
docker-compose restart grafana
```

**🔗 Next:**
- [ELK Stack Guide →](../../logging/elk_stack.md)
- [Logging Best Practices →](../logging/log-management.md)
- [APM Tools →](../apm/application-monitoring.md)
