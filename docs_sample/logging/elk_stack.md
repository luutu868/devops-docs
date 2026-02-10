# ELK Stack (Elasticsearch, Logstash, Kibana)

## 🎯 ELK Stack Overview

**ELK Stack** = Elasticsearch + Logstash + Kibana (+ Beats)

```
┌──────────────────────────────────────────────────────┐
│              ELK STACK ARCHITECTURE                  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  [Apps] ─┐                                          │
│  [Nginx]─┤                                          │
│  [Syslog]┼─► [Filebeat] ──┐                         │
│  [Docker]┘                 │                         │
│                            ▼                         │
│                      [Logstash]                      │
│                     (Parse, Filter)                  │
│                            │                         │
│                            ▼                         │
│                   [Elasticsearch]                    │
│                    (Store, Index)                    │
│                            │                         │
│                            ▼                         │
│                       [Kibana]                       │
│                   (Visualize, Search)                │
└──────────────────────────────────────────────────────┘
```

**Components:**
- **Elasticsearch**: Search and analytics engine (storage)
- **Logstash**: Log processing pipeline (parsing, filtering)
- **Kibana**: Visualization and search UI
- **Beats**: Lightweight data shippers (Filebeat, Metricbeat)

---

## 🚀 Installation

### **1. Docker Compose Setup**

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.3
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - elk
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200"]
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.3
    container_name: logstash
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      - "LS_JAVA_OPTS=-Xms256m -Xmx256m"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.3
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk
    depends_on:
      - elasticsearch
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5601/api/status"]
      interval: 30s
      timeout: 10s
      retries: 5

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.3
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - filebeat_data:/usr/share/filebeat/data
    command: filebeat -e -strict.perms=false
    networks:
      - elk
    depends_on:
      - elasticsearch
      - logstash

volumes:
  elasticsearch_data: {}
  filebeat_data: {}

networks:
  elk:
    driver: bridge
```

### **2. Logstash Pipeline Configuration**

```ruby
# logstash/pipeline/logstash.conf
input {
  # TCP input for applications
  tcp {
    port => 5000
    codec => json
    tags => ["tcp"]
  }

  # UDP input for syslog
  udp {
    port => 5000
    type => "syslog"
    tags => ["udp", "syslog"]
  }

  # Beats input (Filebeat)
  beats {
    port => 5044
    tags => ["beats"]
  }
}

filter {
  # Parse JSON logs
  if [type] == "json" {
    json {
      source => "message"
    }
  }

  # Parse Nginx access logs
  if [log_type] == "nginx_access" {
    grok {
      match => {
        "message" => '%{IPORHOST:remote_addr} - %{DATA:remote_user} \[%{HTTPDATE:time_local}\] "%{WORD:request_method} %{DATA:request_path} HTTP/%{NUMBER:http_version}" %{NUMBER:status} %{NUMBER:body_bytes_sent} "%{DATA:http_referer}" "%{DATA:http_user_agent}"'
      }
    }

    # Convert fields to appropriate types
    mutate {
      convert => {
        "status" => "integer"
        "body_bytes_sent" => "integer"
      }
    }

    # Parse timestamp
    date {
      match => ["time_local", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
    }

    # GeoIP lookup
    geoip {
      source => "remote_addr"
      target => "geoip"
    }
  }

  # Parse application logs
  if [app] == "myapp" {
    # Extract log level
    grok {
      match => {
        "message" => "\[%{LOGLEVEL:log_level}\]%{GREEDYDATA:log_message}"
      }
    }

    # Add custom fields
    mutate {
      add_field => {
        "environment" => "production"
      }
    }
  }

  # Parse Docker container logs
  if [container_name] {
    # Extract container info
    mutate {
      add_field => {
        "container_type" => "docker"
      }
    }
  }

  # Drop debug logs (optional)
  if [log_level] == "DEBUG" {
    drop { }
  }

  # Remove unnecessary fields
  mutate {
    remove_field => ["agent", "ecs", "host"]
  }
}

output {
  # Output to Elasticsearch
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[app]}-%{+YYYY.MM.dd}"
    
    # Create index template
    manage_template => true
    template_name => "logs"
    template_overwrite => true
  }

  # Output to stdout for debugging (optional)
  # stdout {
  #   codec => rubydebug
  # }
}
```

### **3. Logstash Config**

```yaml
# logstash/config/logstash.yml
http.host: "0.0.0.0"
xpack.monitoring.enabled: false
```

### **4. Filebeat Configuration**

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
  # Docker container logs
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

  # Application logs
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log
    fields:
      app: myapp
      log_type: application
    multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
    multiline.negate: true
    multiline.match: after

  # Nginx access logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      log_type: nginx_access
      app: nginx

  # Nginx error logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    fields:
      log_type: nginx_error
      app: nginx

# Processors
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~

# Output to Logstash
output.logstash:
  hosts: ["logstash:5044"]
  
# OR output directly to Elasticsearch
# output.elasticsearch:
#   hosts: ["elasticsearch:9200"]
#   index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

# Logging
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

### **5. Start ELK Stack**

```bash
# Create directories
mkdir -p logstash/{pipeline,config} filebeat

# Create configs (see above)

# Start services
docker-compose up -d

# Wait for Elasticsearch to be ready
until curl -s http://localhost:9200 > /dev/null; do
  echo "Waiting for Elasticsearch..."
  sleep 5
done

# Verify services
docker-compose ps
curl http://localhost:9200         # Elasticsearch
curl http://localhost:9200/_cat/indices?v  # Indices
curl http://localhost:5601/api/status  # Kibana
curl http://localhost:9600         # Logstash

# Access Kibana
open http://localhost:5601
```

---

## 📊 Kibana Configuration

### **1. Create Index Pattern**

```bash
# In Kibana UI:
# 1. Go to Management → Stack Management → Index Patterns
# 2. Click "Create index pattern"
# 3. Enter pattern: logs-*
# 4. Select time field: @timestamp
# 5. Click "Create index pattern"
```

**Via API:**
```bash
curl -X POST "http://localhost:5601/api/saved_objects/index-pattern" \
  -H 'kbn-xsrf: true' \
  -H 'Content-Type: application/json' \
  -d '{
    "attributes": {
      "title": "logs-*",
      "timeFieldName": "@timestamp"
    }
  }'
```

### **2. Create Visualizations**

**Log Volume Over Time:**
```json
{
  "title": "Log Volume",
  "visState": {
    "type": "histogram",
    "params": {
      "type": "histogram",
      "grid": {"categoryLines": false},
      "categoryAxes": [{
        "type": "category",
        "position": "bottom",
        "show": true,
        "title": {"text": "Time"}
      }],
      "valueAxes": [{
        "type": "value",
        "position": "left",
        "show": true,
        "title": {"text": "Count"}
      }]
    },
    "aggs": [
      {
        "type": "date_histogram",
        "schema": "segment",
        "params": {
          "field": "@timestamp",
          "interval": "auto"
        }
      },
      {
        "type": "count",
        "schema": "metric"
      }
    ]
  }
}
```

**Top Error Messages:**
```json
{
  "title": "Top Errors",
  "visState": {
    "type": "table",
    "aggs": [
      {
        "type": "terms",
        "schema": "bucket",
        "params": {
          "field": "message.keyword",
          "size": 10,
          "order": "desc",
          "orderBy": "1"
        }
      },
      {
        "type": "count",
        "schema": "metric"
      }
    ]
  }
}
```

### **3. Create Dashboard**

```bash
# In Kibana:
# 1. Go to Dashboard
# 2. Click "Create dashboard"
# 3. Add visualizations
# 4. Arrange panels
# 5. Save dashboard

# Dashboard layout suggestion:
┌────────────────────────────────────────────┐
│  Log Volume (Line Chart)                   │
├────────────────────────────────────────────┤
│  Error Rate │ Top Errors │ Status Codes    │
├────────────────────────────────────────────┤
│  Application Breakdown │ Geographic Map    │
├────────────────────────────────────────────┤
│  Recent Logs (Table)                       │
└────────────────────────────────────────────┘
```

---

## 🔍 Kibana Query Language (KQL)

### **Basic Queries**

```kql
# Search for term
error

# Field equals value
status:500
log_level:ERROR
app:myapp

# Multiple fields (AND)
status:500 AND app:myapp

# Multiple fields (OR)
status:500 OR status:502

# NOT
NOT status:200

# Wildcard
message:*exception*

# Range
status >= 400 AND status < 500
@timestamp >= "2026-02-10T00:00:00"

# Exists
_exists_:error_code

# Null/missing
NOT _exists_:user_id
```

### **Advanced Queries**

```kql
# Regex
message:/user_id:[0-9]+/

# Multiple values
status:(500 OR 502 OR 503)

# IP range
remote_addr:"192.168.1.0/24"

# Complex query
(status >= 500 AND app:myapp) OR (log_level:ERROR AND NOT message:*test*)

# Time-based
@timestamp >= now-1h
@timestamp >= now-1d/d  # Start of today

# Phrase search
message:"database connection failed"
```

---

## 📈 Log Analysis Examples

### **1. Nginx Access Log Analysis**

```kql
# Find all 5xx errors
status >= 500

# Top 10 URLs with errors
# Visualization: Terms aggregation on request_path field

# Average response time by endpoint
# Visualization: Average metric on request_time, grouped by request_path

# Geographic distribution of requests
# Visualization: Coordinate map using geoip.location

# Traffic by hour
# Visualization: Date histogram with 1h interval
```

### **2. Application Error Tracking**

```kql
# All errors and criticals
log_level:(ERROR OR CRITICAL)

# Errors by service
# Terms aggregation on service field

# Error trend over time
# Date histogram with count metric

# Most common error messages
# Terms aggregation on message field

# Recent exceptions
message:*Exception* OR message:*Error*
```

### **3. Security Analysis**

```kql
# Failed login attempts
message:*failed*login* OR message:*authentication*failed*

# Suspicious activity (brute force)
# Count of failed attempts per IP
# Terms aggregation on remote_addr where message contains "failed"

# Unauthorized access attempts
status:401 OR status:403

# SQL injection attempts
message:*SELECT*FROM* OR message:*UNION*SELECT*
```

---

## 🚨 Alerts in Kibana

### **Create Alert Rule**

```json
{
  "name": "High Error Rate",
  "tags": ["errors", "production"],
  "enabled": true,
  "throttle": "5m",
  "schedule": {
    "interval": "1m"
  },
  "params": {
    "index": ["logs-*"],
    "timeField": "@timestamp",
    "aggType": "count",
    "groupBy": "all",
    "threshold": [100],
    "thresholdComparator": ">",
    "timeWindowSize": 5,
    "timeWindowUnit": "m"
  },
  "conditions": {
    "query": {
      "bool": {
        "must": [
          {"match": {"log_level": "ERROR"}}
        ]
      }
    }
  },
  "actions": [
    {
      "group": "threshold met",
      "id": "slack-connector",
      "params": {
        "message": "High error rate detected: {{context.value}} errors in 5 minutes"
      }
    }
  ]
}
```

---

## 🔧 Elasticsearch Management

### **Index Management**

```bash
# List all indices
curl "http://localhost:9200/_cat/indices?v"

# Create index with mapping
curl -X PUT "http://localhost:9200/logs-myapp" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "message": {"type": "text"},
      "log_level": {"type": "keyword"},
      "service": {"type": "keyword"},
      "trace_id": {"type": "keyword"},
      "duration_ms": {"type": "integer"},
      "status_code": {"type": "integer"}
    }
  }
}
'

# Delete old indices
curl -X DELETE "http://localhost:9200/logs-*-2025.01.*"

# Reindex
curl -X POST "http://localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": {"index": "old-logs"},
  "dest": {"index": "new-logs"}
}
'
```

### **Index Lifecycle Management (ILM)**

```bash
# Create ILM policy
curl -X PUT "http://localhost:9200/_ilm/policy/logs-policy" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "50gb"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
'

# Apply policy to index template
curl -X PUT "http://localhost:9200/_index_template/logs-template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}
'
```

### **Search API**

```bash
# Simple search
curl "http://localhost:9200/logs-*/_search?q=error"

# Query DSL
curl -X GET "http://localhost:9200/logs-*/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        {"match": {"log_level": "ERROR"}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  },
  "size": 100,
  "sort": [{"@timestamp": {"order": "desc"}}]
}
'

# Aggregations
curl -X GET "http://localhost:9200/logs-*/_search" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "interval": "1h"
      },
      "aggs": {
        "error_count": {
          "filter": {"term": {"log_level": "ERROR"}}
        }
      }
    }
  }
}
'
```

---

## ☸️  Kubernetes Deployment

### **Elasticsearch StatefulSet**

```yaml
# elasticsearch-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:8.11.3
          env:
            - name: cluster.name
              value: "k8s-logs"
            - name: node.name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: discovery.seed_hosts
              value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
            - name: cluster.initial_master_nodes
              value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
            - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
          ports:
            - containerPort: 9200
              name: http
            - containerPort: 9300
              name: transport
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 1000m
              memory: 2Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

### **ECK (Elastic Cloud on Kubernetes)**

```bash
# Install ECK operator
kubectl create -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml

# Deploy Elasticsearch
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: logs
  namespace: logging
spec:
  version: 8.11.3
  nodeSets:
    - name: default
      count: 3
      config:
        node.store.allow_mmap: false
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 50Gi
EOF

# Deploy Kibana
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: logs
  namespace: logging
spec:
  version: 8.11.3
  count: 1
  elasticsearchRef:
    name: logs
EOF

# Deploy Filebeat
cat <<EOF | kubectl apply -f -
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: filebeat
  namespace: logging
spec:
  type: filebeat
  version: 8.11.3
  elasticsearchRef:
    name: logs
  config:
    filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
  daemonSet:
    podTemplate:
      spec:
        dnsPolicy: ClusterFirstWithHostNet
        hostNetwork: true
        containers:
          - name: filebeat
            volumeMounts:
              - name: varlogcontainers
                mountPath: /var/log/containers
              - name: varlogpods
                mountPath: /var/log/pods
        volumes:
          - name: varlogcontainers
            hostPath:
              path: /var/log/containers
          - name: varlogpods
            hostPath:
              path: /var/log/pods
EOF
```

---

## 🎓 Summary

**✅ ELK Stack Components:**
- **Elasticsearch**: Distributed search and analytics
- **Logstash**: Log processing pipeline
- **Kibana**: Visualization and exploration
- **Beats**: Lightweight data shippers

**📊 Key Features:**
- Centralized logging
- Real-time search and analysis
- Powerful visualizations
- Alerting and monitoring
- Scalable architecture

**🔧 Management:**
```bash
# Elasticsearch
curl http://localhost:9200/_cluster/health
curl http://localhost:9200/_cat/indices?v

# Logstash
curl http://localhost:9600/_node/stats

# Kibana
open http://localhost:5601
```

**🔗 Next Steps:**
- [Logging Best Practices →](../monitoring/logging/log-management.md)
- [APM Tools →](../monitoring/apm/application-monitoring.md)
- [Monitoring Basics ←](../monitoring/fundamentals/monitoring-basics.md)
