# Logging Best Practices

## 🎯 Logging Fundamentals

**Good logging** = observable, debuggable, maintainable systems

```
┌─────────────────────────────────────────┐
│      LOGGING MATURITY LEVELS            │
├─────────────────────────────────────────┤
│ Level 1: Print statements               │
│ Level 2: Basic logging (file)           │
│ Level 3: Structured logging (JSON)      │
│ Level 4: Centralized logging            │
│ Level 5: Observability (logs+metrics)   │
└─────────────────────────────────────────┘
```

---

## 📝 Log Levels

### **Standard Log Levels**

```
TRACE   → Very detailed, every step
DEBUG   → Detailed diagnostic info
INFO    → General informational messages
WARN    → Warning messages, potential issues
ERROR   → Error events, but app continues
FATAL   → Severe errors, app terminates
```

### **When to Use Each Level**

```python
import logging

logger = logging.getLogger(__name__)

# TRACE/DEBUG - Development only
logger.debug(f"Processing user {user_id}, order {order_id}")
logger.debug(f"Query parameters: {params}")

# INFO - Important business events
logger.info(f"User {user_id} logged in successfully")
logger.info(f"Order {order_id} created: ${total_amount}")
logger.info("Cron job 'daily_report' started")

# WARNING - Unexpected but recoverable
logger.warning(f"API rate limit approaching: {current_rate}/{max_rate}")
logger.warning(f"Deprecated endpoint /api/v1/users called by {client_id}")
logger.warning(f"Retry attempt {retry_count} for payment {payment_id}")

# ERROR - Something failed
logger.error(f"Failed to process payment {payment_id}: {error_msg}")
logger.error(f"Database connection lost: {db_host}")
logger.error(f"Email delivery failed for order {order_id}")

# FATAL/CRITICAL - System crash
logger.critical("Database unavailable, shutting down")
logger.critical(f"Out of memory: {memory_used}/{memory_total}")
```

---

## 🏗️ Structured Logging

### **❌ Bad: Unstructured Logs**

```python
# DON'T do this
print(f"User John logged in at 2026-02-10 10:30:00")
print(f"Order #123 total: $50.99")

# Problems:
# - Hard to parse
# - No context
# - Can't filter/aggregate
# - No metadata
```

### **✅ Good: Structured Logs (JSON)**

```python
import json
import logging
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'context': {
                'function': record.funcName,
                'line': record.lineno,
                'file': record.pathname
            }
        }
        
        # Add extra fields
        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id
        if hasattr(record, 'trace_id'):
            log_data['trace_id'] = record.trace_id
        
        # Add exception info
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_data)

# Configure
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger = logging.getLogger(__name__)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Usage
logger.info(
    "User logged in successfully",
    extra={
        'user_id': 123,
        'username': 'john@example.com',
        'request_id': 'req-abc-123',
        'ip_address': '192.168.1.1',
        'user_agent': 'Mozilla/5.0'
    }
)

# Output:
# {
#   "timestamp": "2026-02-10T10:30:00.123Z",
#   "level": "INFO",
#   "logger": "auth.views",
#   "message": "User logged in successfully",
#   "user_id": 123,
#   "username": "john@example.com",
#   "request_id": "req-abc-123",
#   "ip_address": "192.168.1.1",
#   "user_agent": "Mozilla/5.0",
#   "context": {
#     "function": "login_view",
#     "line": 42,
#     "file": "/app/auth/views.py"
#   }
# }
```

### **Node.js Structured Logging (Winston)**

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'api-gateway',
    environment: process.env.NODE_ENV
  },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Usage
logger.info('User login', {
  user_id: 123,
  username: 'john@example.com',
  request_id: 'req-abc-123',
  duration_ms: 145
});

logger.error('Payment failed', {
  user_id: 123,
  order_id: 456,
  amount: 50.99,
  error_code: 'INSUFFICIENT_FUNDS',
  payment_method: 'credit_card'
});
```

---

## 🔍 Context and Correlation

### **Request ID / Trace ID**

```python
# Flask middleware
import uuid
from flask import Flask, g, request

app = Flask(__name__)

@app.before_request
def before_request():
    # Generate or use existing request ID
    g.request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
    g.trace_id = request.headers.get('X-Trace-ID', str(uuid.uuid4()))

# Custom logger
class RequestLogger:
    def __init__(self, logger):
        self.logger = logger
    
    def _log(self, level, message, **kwargs):
        extra = {
            'request_id': getattr(g, 'request_id', None),
            'trace_id': getattr(g, 'trace_id', None),
            'user_id': getattr(g, 'user_id', None),
            **kwargs
        }
        getattr(self.logger, level)(message, extra=extra)
    
    def info(self, message, **kwargs):
        self._log('info', message, **kwargs)
    
    def error(self, message, **kwargs):
        self._log('error', message, **kwargs)

# Usage
req_logger = RequestLogger(logger)

@app.route('/api/orders', methods=['POST'])
def create_order():
    req_logger.info('Creating order', order_data=request.json)
    
    try:
        order = process_order(request.json)
        req_logger.info('Order created successfully', order_id=order.id)
        return {'order_id': order.id}, 201
    except Exception as e:
        req_logger.error('Order creation failed', error=str(e))
        return {'error': 'Order creation failed'}, 500
```

### **Distributed Tracing with OpenTelemetry**

```python
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, BatchSpanProcessor

# Setup tracer
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(ConsoleSpanExporter())
)

# Instrument Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

# Get tracer
tracer = trace.get_tracer(__name__)

@app.route('/api/orders/<order_id>')
def get_order(order_id):
    # Create span
    with tracer.start_as_current_span("get_order") as span:
        span.set_attribute("order_id", order_id)
        
        # Log with trace context
        ctx = trace.get_current_span().get_span_context()
        logger.info(
            f"Fetching order {order_id}",
            extra={
                'trace_id': f"{ctx.trace_id:032x}",
                'span_id': f"{ctx.span_id:016x}"
            }
        )
        
        order = fetch_order_from_db(order_id)
        
        if order:
            span.set_attribute("order_found", True)
            return order
        else:
            span.set_attribute("order_found", False)
            logger.warning(f"Order {order_id} not found")
            return {'error': 'Not found'}, 404
```

---

## 🎨 Log Format Standards

### **Common Fields**

```json
{
  "timestamp": "2026-02-10T10:30:00.123Z",  // ISO 8601 format
  "level": "INFO",                         // Log level
  "logger": "app.service.user",            // Logger name
  "message": "User created successfully",  // Human-readable message
  
  // Context
  "request_id": "req-abc-123",
  "trace_id": "trace-xyz-789",
  "span_id": "span-def-456",
  
  // User context
  "user_id": 123,
  "session_id": "sess-abc-123",
  
  // Request context
  "method": "POST",
  "path": "/api/users",
  "status_code": 201,
  "duration_ms": 145,
  "ip_address": "192.168.1.1",
  
  // Application context
  "service": "api-gateway",
  "environment": "production",
  "version": "1.2.3",
  "hostname": "api-01",
  
  // Error context (if error)
  "error": {
    "type": "ValidationError",
    "message": "Email is required",
    "stack_trace": "...",
    "code": "VAL_001"
  },
  
  // Custom fields
  "order_id": 456,
  "payment_method": "credit_card",
  "amount": 50.99
}
```

---

## 📦 What to Log

### **✅ DO Log**

```python
# 1. Authentication events
logger.info("User logged in", extra={'user_id': user.id, 'method': 'oauth'})
logger.warning("Failed login attempt", extra={'username': username, 'ip': ip})

# 2. Business transactions
logger.info("Order created", extra={
    'order_id': order.id,
    'user_id': user.id,
    'total_amount': order.total,
    'items_count': len(order.items)
})

# 3. External API calls
logger.info("Calling payment gateway", extra={
    'provider': 'stripe',
    'amount': amount,
    'currency': 'USD'
})

# 4. State changes
logger.info("Order status changed", extra={
    'order_id': order.id,
    'old_status': 'pending',
    'new_status': 'confirmed'
})

# 5. Errors and exceptions
try:
    process_payment(payment)
except PaymentError as e:
    logger.error("Payment processing failed", extra={
        'payment_id': payment.id,
        'error_code': e.code,
        'error_message': str(e)
    }, exc_info=True)

# 6. Performance metrics
start_time = time.time()
result = expensive_operation()
duration = time.time() - start_time
logger.info("Expensive operation completed", extra={
    'operation': 'data_export',
    'duration_ms': duration * 1000,
    'records_processed': len(result)
})

# 7. Configuration changes
logger.info("Configuration reloaded", extra={
    'old_config': old_config,
    'new_config': new_config,
    'changed_keys': changed_keys
})
```

### **❌ DON'T Log**

```python
# 1. Sensitive data
logger.info(f"Password: {password}")  # ❌ NEVER
logger.info(f"Credit card: {credit_card}")  # ❌ NEVER
logger.info(f"SSN: {ssn}")  # ❌ NEVER
logger.info(f"API key: {api_key}")  # ❌ NEVER

# 2. Excessive debug info (in production)
for item in items:  # ❌ Don't log in loops
    logger.debug(f"Processing item {item}")

# 3. Redundant information
logger.info("Starting function")  # ❌ Too granular
do_something()
logger.info("Function completed")  # ❌ Too granular

# 4. Entire request/response bodies (unless necessary)
logger.info(f"Request: {request.json}")  # ❌ Too verbose

# ✅ Better: Log summary
logger.info("Request received", extra={
    'endpoint': '/api/users',
    'method': 'POST',
    'content_length': len(request.data)
})
```

---

## 🔒 Security and Privacy

### **Data Masking**

```python
import re

def mask_sensitive_data(data):
    """Mask sensitive information in logs"""
    if isinstance(data, dict):
        masked = {}
        for key, value in data.items():
            if key.lower() in ['password', 'secret', 'token', 'api_key']:
                masked[key] = '***REDACTED***'
            elif key.lower() in ['credit_card', 'card_number']:
                masked[key] = mask_credit_card(str(value))
            elif key.lower() == 'email':
                masked[key] = mask_email(str(value))
            elif isinstance(value, dict):
                masked[key] = mask_sensitive_data(value)
            else:
                masked[key] = value
        return masked
    return data

def mask_credit_card(card_number):
    """Mask credit card: 1234-5678-9012-3456 → ****-****-****-3456"""
    return re.sub(r'\d(?=\d{4})', '*', card_number)

def mask_email(email):
    """Mask email: john@example.com → j***@example.com"""
    if '@' in email:
        name, domain = email.split('@')
        return f"{name[0]}***@{domain}"
    return email

# Usage
user_data = {
    'username': 'john',
    'email': 'john@example.com',
    'password': 'secret123',
    'credit_card': '1234-5678-9012-3456'
}

logger.info("User data", extra=mask_sensitive_data(user_data))
# Output: {username: 'john', email: 'j***@example.com', 
#          password: '***REDACTED***', credit_card: '****-****-****-3456'}
```

---

## 📊 Log Aggregation Patterns

### **Centralized Logging Architecture**

```
┌──────────────────────────────────────────────────┐
│     Application Servers (3 nodes)                │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │  App 1  │  │  App 2  │  │  App 3  │         │
│  │         │  │         │  │         │         │
│  │ Filebeat│  │ Filebeat│  │ Filebeat│         │
│  └────┬────┘  └────┬────┘  └────┬────┘         │
└───────┼────────────┼────────────┼───────────────┘
        │            │            │
        └────────────┼────────────┘
                     ▼
         ┌─────────────────────┐
         │     Logstash         │
         │  (Parse & Filter)    │
         └──────────┬───────────┘
                    ▼
         ┌─────────────────────┐
         │   Elasticsearch      │
         │  (Store & Index)     │
         └──────────┬───────────┘
                    ▼
         ┌─────────────────────┐
         │      Kibana          │
         │  (Query & Visualize) │
         └─────────────────────┘
```

### **Log Retention Policy**

```yaml
retention_policy:
  # Hot tier (SSD, fast queries)
  hot:
    duration: 7 days
    storage: SSD
    shards: 5
    replicas: 1
  
  # Warm tier (slower, compressed)
  warm:
    duration: 30 days
    storage: HDD
    shards: 1
    replicas: 0
    compression: enabled
  
  # Cold tier (archived)
  cold:
    duration: 90 days
    storage: S3/Glacier
    format: compressed_parquet
  
  # After 90 days: DELETE
  delete_after: 90 days

# Environment-specific retention
environments:
  production:
    retention: 90 days
  staging:
    retention: 30 days
  development:
    retention: 7 days
```

---

## 🚀 Performance Optimization

### **Async Logging**

```python
import logging
from logging.handlers import QueueHandler, QueueListener
from queue import Queue

# Create queue
log_queue = Queue(-1)  # No limit

# Create handlers that will process log records
file_handler = logging.FileHandler('app.log')
file_handler.setFormatter(JSONFormatter())

# Create queue listener
queue_listener = QueueListener(log_queue, file_handler, respect_handler_level=True)
queue_listener.start()

# Configure logger to use queue
queue_handler = QueueHandler(log_queue)
logger = logging.getLogger(__name__)
logger.addHandler(queue_handler)
logger.setLevel(logging.INFO)

# Now logging is async!
logger.info("This doesn't block")  # Returns immediately

# Cleanup on shutdown
import atexit
atexit.register(queue_listener.stop)
```

### **Sampling for High-Volume Logs**

```python
import random

class SamplingLogger:
    def __init__(self, logger, sample_rate=0.1):
        self.logger = logger
        self.sample_rate = sample_rate
    
    def info(self, message, force=False, **kwargs):
        if force or random.random() < self.sample_rate:
            self.logger.info(message, extra=kwargs)
    
    def error(self, message, **kwargs):
        # Always log errors
        self.logger.error(message, extra=kwargs)

# Usage
sampled_logger = SamplingLogger(logger, sample_rate=0.01)  # Log 1% of events

# High-volume endpoint
@app.route('/api/health')
def health_check():
    sampled_logger.info("Health check")  # Only logged 1% of the time
    return {'status': 'ok'}

# Important events always logged
@app.route('/api/orders', methods=['POST'])
def create_order():
    sampled_logger.info("Order created", force=True, order_id=order.id)
    return {'order_id': order.id}
```

---

## 🎓 Summary

**✅ Logging Best Practices:**

1. **Use structured logging** (JSON Format)
2. **Include context** (request_id, user_id, trace_id)
3. **Log appropriate level** (INFO for events, ERROR for failures)
4. **Don't log sensitive data** (passwords, tokens, PII)
5. **Use correlation IDs** for distributed systems
6. **Implement log rotation** and retention policies
7. **Aggregate centrally** (ELK, CloudWatch, Datadog)
8. **Monitor log volume** and optimize high-traffic endpoints
9. **Make logs actionable** (include runbooks, error codes)
10. **Test logging** (verify log format, test alerts)

**📊 Key Metrics to Track:**
- Log volume per service
- Error rate trends
- Performance of expensive operations
- Business metrics (orders, signups, payments)

**🔗 Related Documentation:**
- [ELK Stack Setup ←](../../logging/elk_stack.md)
- [Monitoring Basics ←](../fundamentals/monitoring-basics.md)
- [Prometheus & Grafana ←](../prometheus-grafana/setup.md)
