# Application Performance Monitoring (APM)

## 🎯 APM Overview

**APM (Application Performance Monitoring)** = theo dõi hiệu suất ứng dụng từ góc độ end-user.

```
┌─────────────────────────────────────────────────┐
│           APM MONITORING SCOPE                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  [User Browser] ──► [Frontend] ──► [Backend]   │
│                          │              │       │
│                          ▼              ▼       │
│                     [API Calls]   [Database]    │
│                          │              │       │
│                          ▼              ▼       │
│                   [External Services]           │
│                                                 │
│  APM tracks:                                    │
│  - Request/Response times                       │
│  - Error rates                                  │
│  - Throughput                                   │
│  - Database query performance                   │
│  - External API latency                         │
│  - Resource utilization                         │
└─────────────────────────────────────────────────┘
```

**Key Capabilities:**
- ✅ **Distributed Tracing**: Track requests across microservices
- ✅ **Real User Monitoring (RUM)**: Actual user experience
- ✅ **Transaction Tracing**: End-to-end request flow
- ✅ **Error Tracking**: Exceptions and stack traces
- ✅ **Performance Profiling**: CPU, memory, slow queries
- ✅ **Service Maps**: Visualize service dependencies

---

## 📊 Popular APM Tools

### **Comparison Table**

| Tool | Best For | Pricing | Key Features |
|------|----------|---------|--------------|
| **New Relic** | Full-stack monitoring | $99+/month | Real-time monitoring, AI insights |
| **Datadog** | Cloud-native apps | $15+/host/month | 600+ integrations, logs+metrics |
| **Dynatrace** | Enterprise, AI-powered | $69+/host/month | Auto-discovery, root cause analysis |
| **AppDynamics** | Large enterprises | Custom | Business transaction monitoring |
| **Elastic APM** | ELK users | Free (open source) | Integrates with Elasticsearch |
| **Jaeger** | Microservices tracing | Free (CNCF) | OpenTracing, Kubernetes native |
| **Zipkin** | Distributed tracing | Free (open source) | Simple, lightweight |
| **Sentry** | Error tracking | Free tier | Real-time alerts, release tracking |

---

## 🔍 Elastic APM Setup

### **1. Architecture**

```
┌────────────────────────────────────────────┐
│          ELASTIC APM STACK                 │
├────────────────────────────────────────────┤
│                                            │
│  [Application]                             │
│       │                                    │
│       │ APM Agent                          │
│       ▼                                    │
│  [APM Server] ──► [Elasticsearch]         │
│                         │                  │
│                         ▼                  │
│                    [Kibana APM UI]         │
└────────────────────────────────────────────┘
```

### **2. Docker Compose**

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.3
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    networks:
      - elastic

  apm-server:
    image: docker.elastic.co/apm/apm-server:8.11.3
    ports:
      - "8200:8200"
    environment:
      - output.elasticsearch.hosts=["elasticsearch:9200"]
      - apm-server.rum.enabled=true
    networks:
      - elastic
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.3
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elastic
    depends_on:
      - elasticsearch

networks:
  elastic:
    driver: bridge
```

### **3. Instrument Node.js Application**

```javascript
// app.js
// Initialize APM at the very beginning!
require('elastic-apm-node').start({
  serviceName: 'my-app',
  secretToken: '',
  serverUrl: 'http://localhost:8200',
  environment: 'production'
});

const express = require('express');
const app = express();

// Your application code
app.get('/api/users', async (req, res) => {
  try {
    const users = await fetchUsers();
    res.json(users);
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

async function fetchUsers() {
  // APM automatically traces this
  const users = await db.query('SELECT * FROM users');
  return users;
}

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

```json
// package.json
{
  "dependencies": {
    "elastic-apm-node": "^4.0.0",
    "express": "^4.18.0"
  }
}
```

### **4. Instrument Python Application**

```python
# app.py
from elasticapm.contrib.flask import ElasticAPM
from flask import Flask

app = Flask(__name__)

# Configure APM
app.config['ELASTIC_APM'] = {
    'SERVICE_NAME': 'my-python-app',
    'SERVER_URL': 'http://localhost:8200',
    'ENVIRONMENT': 'production'
}

apm = ElasticAPM(app)

@app.route('/api/users')
def get_users():
    try:
        users = fetch_users_from_db()
        return {'users': users}
    except Exception as e:
        # APM captures this exception
        raise

@app.route('/api/slow')
def slow_endpoint():
    # APM will track this as slow transaction
    import time
    time.sleep(3)
    return {'status': 'done'}

if __name__ == '__main__':
    app.run(port=5000)
```

```txt
# requirements.txt
elastic-apm[flask]==6.19.0
Flask==3.0.0
```

---

## 🔍 Jaeger (Distributed Tracing)

### **1. Quick Start**

```bash
# Run Jaeger all-in-one
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.51

# Access UI
open http://localhost:16686
```

### **2. Instrument with OpenTelemetry**

```javascript
// Node.js with OpenTelemetry
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

// Configure tracer
const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'my-service',
  }),
});

// Configure Jaeger exporter
const exporter = new JaegerExporter({
  serviceName: 'my-service',
  endpoint: 'http://localhost:14268/api/traces',
});

provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();

// Auto-instrument HTTP and Express
registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});

// Your application
const express = require('express');
const app = express();

app.get('/api/users', async (req, res) => {
  // This will be automatically traced
  const users = await fetchUsers();
  res.json(users);
});

app.listen(3000);
```

### **3. Manual Instrumentation**

```javascript
const opentelemetry = require('@opentelemetry/api');

async function processOrder(orderId) {
  const tracer = opentelemetry.trace.getTracer('my-service');
  
  // Create a span
  const span = tracer.startSpan('process_order');
  span.setAttribute('order_id', orderId);
  
  try {
    // Validate order
    const validateSpan = tracer.startSpan('validate_order', {
      parent: span
    });
    await validateOrder(orderId);
    validateSpan.end();
    
    // Charge payment
    const paymentSpan = tracer.startSpan('charge_payment', {
      parent: span
    });
    await chargePayment(orderId);
    paymentSpan.setAttribute('amount', 99.99);
    paymentSpan.end();
    
    // Update inventory
    const inventorySpan = tracer.startSpan('update_inventory', {
      parent: span
    });
    await updateInventory(orderId);
    inventorySpan.end();
    
    span.setStatus({ code: opentelemetry.SpanStatusCode.OK });
  } catch (error) {
    span.setStatus({
      code: opentelemetry.SpanStatusCode.ERROR,
      message: error.message
    });
    span.recordException(error);
    throw error;
  } finally {
    span.end();
  }
}
```

---

## 🚨 Sentry (Error Tracking)

### **1. Setup**

```javascript
// Node.js
const Sentry = require('@sentry/node');

Sentry.init({
  dsn: 'https://your-dsn@sentry.io/project-id',
  environment: 'production',
  tracesSampleRate: 1.0, // 100% of transactions
  
  // Optional: Release tracking
  release: 'my-app@1.2.3',
  
  // Optional: Performance monitoring
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
  ],
});

const express = require('express');
const app = express();

// Request handler must be first
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// Your routes
app.get('/api/users', (req, res) => {
  // This error will be captured
  throw new Error('Something went wrong!');
});

// Error handler must be last
app.use(Sentry.Handlers.errorHandler());

app.listen(3000);
```

```python
# Python
import sentry_sdk
from sentry_sdk.integrations.flask import FlaskIntegration

sentry_sdk.init(
    dsn="https://your-dsn@sentry.io/project-id",
    integrations=[FlaskIntegration()],
    traces_sample_rate=1.0,
    environment="production",
    release="my-app@1.2.3"
)

from flask import Flask
app = Flask(__name__)

@app.route('/api/users')
def get_users():
    # This error will be captured
    division_by_zero = 1 / 0
    return users
```

### **2. Custom Context**

```javascript
// Add user context
Sentry.setUser({
  id: '123',
  email: 'john@example.com',
  username: 'john'
});

// Add tags
Sentry.setTag('page_locale', 'en-US');
Sentry.setTag('user_plan', 'premium');

// Add breadcrumbs
Sentry.addBreadcrumb({
  category: 'auth',
  message: 'User logged in',
  level: 'info',
});

// Capture custom error
try {
  processPayment();
} catch (error) {
  Sentry.captureException(error, {
    tags: {
      payment_method: 'credit_card',
    },
    extra: {
      order_id: 123,
      amount: 99.99
    }
  });
}

// Capture message
Sentry.captureMessage('Something important happened', 'warning');
```

---

## 📊 Key APM Metrics

### **1. Response Time Metrics**

```javascript
// Track custom metrics
const apm = require('elastic-apm-node');

async function processOrder(order) {
  const span = apm.startSpan('process_order');
  const start = Date.now();
  
  try {
    // Process order
    const result = await doProcessOrder(order);
    
    // Record custom metrics
    span.setLabel('order_id', order.id);
    span.setLabel('items_count', order.items.length);
    span.setLabel('total_amount', order.total);
    
    const duration = Date.now() - start;
    span.setLabel('duration_ms', duration);
    
    return result;
  } finally {
    span.end();
  }
}
```

### **2. Database Query Monitoring**

```python
from elasticapm import capture_span

@capture_span('database')
def fetch_users():
    with elasticapm.capture_span(
        'db.query',
        span_type='db',
        span_subtype='postgresql',
        span_action='query'
    ) as span:
        span.label(query='SELECT * FROM users WHERE active = true')
        result = db.execute('SELECT * FROM users WHERE active = true')
        span.label(rows_affected=len(result))
        return result
```

### **3. External API Monitoring**

```javascript
const apm = require('elastic-apm-node');
const axios = require('axios');

async function callExternalAPI(endpoint) {
  const span = apm.startSpan('external_api_call', 'external', 'http');
  
  try {
    span.setLabel('endpoint', endpoint);
    const start = Date.now();
    
    const response = await axios.get(endpoint);
    
    const duration = Date.now() - start;
    span.setLabel('duration_ms', duration);
    span.setLabel('status_code', response.status);
    
    return response.data;
  } catch (error) {
    span.setLabel('error', error.message);
    throw error;
  } finally {
    span.end();
  }
}
```

---

## 🔔 APM Alerting

### **Elastic APM Alerts**

```json
{
  "name": "High Error Rate",
  "tags": ["apm", "errors"],
  "schedule": {
    "interval": "1m"
  },
  "params": {
    "serviceName": "my-app",
    "environment": "production",
    "alertType": "errorRate",
    "threshold": 5,
    "windowSize": 5,
    "windowUnit": "m"
  },
  "actions": [
    {
      "group": "threshold met",
      "id": "slack-connector",
      "params": {
        "message": "Error rate exceeds 5% in service {{context.serviceName}}"
      }
    }
  ]
}
```

### **Sentry Alerts**

```yaml
# .sentry/alerts.yml
alerts:
  - name: High Error Rate
    conditions:
      - events: 100
        interval: 1m
    actions:
      - type: slack
        channel: "#alerts"
        message: "High error rate detected"
  
  - name: New Release Issues
    conditions:
      - new_issue
      - release: "latest"
    actions:
      - type: email
        recipients: ["team@example.com"]
```

---

## 📈 APM Best Practices

### **1. Sampling**

```javascript
// Don't trace 100% in high-traffic apps
const apm = require('elastic-apm-node').start({
  serviceName: 'my-app',
  transactionSampleRate: 0.1, // Sample 10% of requests
});
```

### **2. Custom Transactions**

```javascript
// Group similar operations
apm.setTransactionName('POST /api/orders/:id/items');

// Don't use dynamic values in names
// ❌ Bad
apm.setTransactionName(`GET /api/users/${userId}`);

// ✅ Good
apm.setTransactionName('GET /api/users/:id');
apm.setLabel('user_id', userId);
```

### **3. Context Propagation**

```javascript
// Pass trace context to background jobs
const apm = require('elastic-apm-node');

app.post('/api/orders', (req, res) => {
  const transaction = apm.currentTransaction;
  const traceContext = transaction.traceparent;
  
  // Send to queue with context
  queue.publish({
    orderId: order.id,
    traceContext: traceContext
  });
  
  res.json({ orderId: order.id });
});

// In worker
worker.process(async (job) => {
  // Continue trace from context
  const transaction = apm.startTransaction('process-order', 'worker', {
    childOf: job.data.traceContext
  });
  
  try {
    await processOrder(job.data.orderId);
    transaction.end('success');
  } catch (error) {
    transaction.end('failure');
    throw error;
  }
});
```

---

## 🎓 Summary

**✅ APM Key Features:**
- **Distributed Tracing**: Track requests across microservices
- **Real User Monitoring**: Actual user experience metrics
- **Error Tracking**: Exceptions with stack traces
- **Performance Profiling**: Identify bottlenecks
- **Service Maps**: Visualize dependencies

**📊 Popular Tools:**
- **Elastic APM**: Free, integrates with ELK
- **Jaeger**: CNCF, Kubernetes-native
- **Sentry**: Best for error tracking
- **New Relic/Datadog**: Full-featured commercial

**🔧 Quick Start:**
```bash
# Elastic APM
npm install elastic-apm-node

# Jaeger
docker run -d -p 16686:16686 jaegertracing/all-in-one

# Sentry
npm install @sentry/node
```

**🔗 Related:**
- [Monitoring Basics ←](../fundamentals/monitoring-basics.md)
- [Prometheus & Grafana ←](../prometheus-grafana/setup.md)
- [Logging Best Practices ←](../logging/log-management.md)
