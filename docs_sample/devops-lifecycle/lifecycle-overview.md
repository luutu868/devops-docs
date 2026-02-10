# DevOps Lifecycle

## 🔄 Tổng Quan

DevOps Lifecycle là một quy trình liên tục (infinite loop) bao gồm 8 giai đoạn chính, từ lập kế hoạch đến giám sát và phản hồi.

```
        ┌──────────────────────┐
        │      PLAN            │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │      CODE            │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │      BUILD           │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │      TEST            │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │      RELEASE         │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │      DEPLOY          │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │      OPERATE         │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │      MONITOR         │
        └──────────┬───────────┘
                   │
                   └──→ Back to PLAN
```

## 1️⃣ PLAN (Lập Kế Hoạch)

### **Mục Đích**
- Xác định requirements và features
- Lập kế hoạch sprints/iterations
- Prioritize backlog
- Define success metrics

### **Activities**
- User story creation
- Sprint planning
- Risk assessment
- Resource allocation

### **Tools**
- **Project Management**: Jira, Trello, Azure Boards
- **Documentation**: Confluence, Notion, Wiki
- **Communication**: Slack, Microsoft Teams

### **Example: Sprint Planning**
```markdown
Sprint 15 - Q1 2026
Duration: 2 weeks

User Stories:
1. [HIGH] As a user, I want to login with SSO
   - Story Points: 8
   - Acceptance Criteria: ...

2. [MED] As admin, I want to view audit logs
   - Story Points: 5
   - Acceptance Criteria: ...

Technical Tasks:
- Setup monitoring for new service
- Update Docker base images
- Database migration for audit table
```

## 2️⃣ CODE (Phát Triển)

### **Mục Đích**
- Phát triển features theo plan
- Code review và collaboration
- Maintain code quality

### **Activities**
- Feature development
- Bug fixes
- Code reviews
- Pair programming

### **Tools**
- **IDE**: VS Code, IntelliJ, PyCharm
- **Version Control**: Git, GitHub, GitLab, Bitbucket
- **Code Review**: GitHub PRs, GitLab MRs, Gerrit

### **Best Practices**
```bash
# Feature branch workflow
git checkout -b feature/user-login
# ... code changes ...
git add .
git commit -m "feat: implement SSO login"
git push origin feature/user-login
# Create Pull Request
```

### **Code Review Checklist**
- [ ] Code follows style guide
- [ ] Tests are included
- [ ] Documentation updated
- [ ] No security vulnerabilities
- [ ] Performance considered
- [ ] Error handling in place

## 3️⃣ BUILD (Xây Dựng)

### **Mục Đích**
- Compile source code
- Manage dependencies
- Create artifacts (binaries, containers)
- Run static analysis

### **Activities**
- Compilation
- Dependency resolution
- Packaging
- Code quality checks

### **Tools**
- **Build Tools**: Maven, Gradle, npm, pip, Make
- **CI Servers**: Jenkins, GitLab CI, GitHub Actions, CircleCI
- **Artifact Repository**: Nexus, Artifactory, Docker Hub

### **Example: Maven Build**
```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <source>11</source>
                <target>11</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### **Example: Docker Build**
```dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
docker build -t myapp:v1.0.0 .
docker push registry.company.com/myapp:v1.0.0
```

## 4️⃣ TEST (Kiểm Thử)

### **Mục Đích**
- Verify functionality
- Ensure code quality
- Catch bugs early
- Validate performance

### **Testing Pyramid**
```
         /\         
        /  \        E2E Tests (10%)
       /____\       
      /      \      Integration Tests (30%)
     /________\     
    /          \    Unit Tests (60%)
   /____________\   
```

### **Test Types**

#### **Unit Tests**
```python
# test_calculator.py
import unittest
from calculator import add

class TestCalculator(unittest.TestCase):
    def test_add_positive_numbers(self):
        self.assertEqual(add(2, 3), 5)
    
    def test_add_negative_numbers(self):
        self.assertEqual(add(-2, -3), -5)
```

#### **Integration Tests**
```python
# test_api.py
def test_user_registration():
    response = client.post('/api/register', json={
        'username': 'testuser',
        'password': 'Test123!'
    })
    assert response.status_code == 201
    assert 'user_id' in response.json()
```

#### **End-to-End Tests**
```javascript
// test_login.spec.js
describe('User Login', () => {
  it('should login successfully with valid credentials', () => {
    cy.visit('/login')
    cy.get('#username').type('testuser')
    cy.get('#password').type('password123')
    cy.get('#login-btn').click()
    cy.url().should('include', '/dashboard')
  })
})
```

### **Tools**
- **Unit Testing**: JUnit, pytest, Jest, Mocha
- **Integration**: TestNG, pytest, Postman
- **E2E**: Selenium, Cypress, Playwright
- **Performance**: JMeter, Gatling, k6
- **Security**: OWASP ZAP, SonarQube, Snyk

## 5️⃣ RELEASE (Phát Hành)

### **Mục Đích**
- Prepare for deployment
- Version management
- Release notes
- Approval workflows

### **Activities**
- Semantic versioning
- Release note generation
- Change management
- Approval gates

### **Semantic Versioning**
```
Version: MAJOR.MINOR.PATCH

Example: 2.1.3
- MAJOR: Breaking changes (2.x.x)
- MINOR: New features, backward compatible (x.1.x)
- PATCH: Bug fixes (x.x.3)
```

### **Release Checklist**
```markdown
## Release v2.1.0

- [ ] All tests passing
- [ ] Security scan completed
- [ ] Documentation updated
- [ ] Release notes written
- [ ] Stakeholders notified
- [ ] Rollback plan ready
- [ ] Monitoring prepared
- [ ] Production approval obtained
```

### **Tools**
- **Release Management**: GitHub Releases, GitLab Releases
- **Change Management**: ServiceNow, Jira Service Desk
- **Approval**: GitHub protected branches, GitLab approval rules

## 6️⃣ DEPLOY (Triển Khai)

### **Mục Đích**
- Deploy to production
- Minimize downtime
- Ensure reliability
- Enable rollback

### **Deployment Strategies**

#### **Blue-Green Deployment**
```
┌─────────────┐         ┌─────────────┐
│   Blue      │         │   Green     │
│  (Current)  │ ----->  │   (New)     │
│   v1.0.0    │ Switch  │   v2.0.0    │
└─────────────┘         └─────────────┘
```

#### **Canary Deployment**
```
v1.0.0: ████████████████████░░  95% traffic
v2.0.0: ██░░░░░░░░░░░░░░░░░░░░   5% traffic

Monitor → If OK → Increase v2.0.0 traffic gradually
```

#### **Rolling Deployment**
```
Step 1: [v1] [v1] [v1] [v1]
Step 2: [v2] [v1] [v1] [v1]
Step 3: [v2] [v2] [v1] [v1]
Step 4: [v2] [v2] [v2] [v1]
Step 5: [v2] [v2] [v2] [v2]
```

### **Tools**
- **CD Platforms**: ArgoCD, Spinnaker, Flux
- **Configuration**: Ansible, Chef, Puppet
- **Orchestration**: Kubernetes, Docker Swarm
- **Load Balancers**: Nginx, HAProxy, AWS ELB

### **Example: Kubernetes Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0.0
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
```

## 7️⃣ OPERATE (Vận Hành)

### **Mục Đích**
- Maintain system health
- Manage infrastructure
- Handle incidents
- Capacity planning

### **Activities**
- Incident response
- Performance tuning
- Backup & recovery
- Security patching
- Scaling operations

### **SRE Practices**

#### **Error Budgets**
```
SLO: 99.9% uptime
Error Budget: 0.1% = 43 minutes/month

Current Month:
Downtime: 25 minutes
Remaining Budget: 18 minutes

✅ Status: Within budget
```

#### **On-Call Rotation**
```
Week 1: Engineer A (Primary) + Engineer B (Secondary)
Week 2: Engineer C (Primary) + Engineer A (Secondary)
Week 3: Engineer B (Primary) + Engineer C (Secondary)
```

### **Tools**
- **Configuration**: Ansible, Terraform
- **Orchestration**: Kubernetes, Docker Swarm
- **Service Mesh**: Istio, Linkerd
- **Incident Management**: PagerDuty, Opsgenie

## 8️⃣ MONITOR (Giám Sát)

### **Mục Đích**
- Track system health
- Detect issues early
- Gather insights
- Enable data-driven decisions

### **The Three Pillars of Observability**

#### **1. Metrics**
```
# CPU Usage
node_cpu_usage{instance="server1"} 75%

# Response Time
http_request_duration_seconds{method="GET", status="200"} 0.25

# Error Rate
http_requests_total{status="500"} / rate[5m]
```

#### **2. Logs**
```json
{
  "timestamp": "2026-02-09T10:30:45Z",
  "level": "ERROR",
  "service": "api-service",
  "message": "Database connection failed",
  "trace_id": "abc123",
  "user_id": "user456"
}
```

#### **3. Traces**
```
Request Flow:
API Gateway (50ms)
  └─> Auth Service (20ms)
  └─> User Service (100ms)
      └─> Database Query (80ms)
  └─> Response (30ms)

Total: 200ms
```

### **Key Metrics (DORA + SRE)**

```markdown
## Deployment Metrics
- Deployment Frequency: 10/day
- Lead Time: 2 hours
- Change Failure Rate: 5%
- MTTR: 15 minutes

## Performance Metrics
- Latency (p95): 200ms
- Throughput: 1000 req/s
- Error Rate: 0.1%
- Saturation: 60% capacity

## Business Metrics
- Active Users: 50,000
- Conversion Rate: 3.5%
- Revenue: $100k/day
```

### **Alerting Rules**
```yaml
# Prometheus alerting rule
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} for 5 minutes"
```

### **Tools**
- **Metrics**: Prometheus, Datadog, New Relic
- **Logging**: ELK Stack, Splunk, Loki
- **Tracing**: Jaeger, Zipkin, AWS X-Ray
- **Dashboards**: Grafana, Kibana
- **APM**: Datadog, New Relic, AppDynamics

## 🔁 Continuous Feedback Loop

```
Monitor → Analyze → Learn → Improve → Plan
   ↑                                      ↓
   └──────────────────────────────────────┘
```

### **Feedback Mechanisms**
1. **Real-time Monitoring**: Immediate alerts
2. **User Feedback**: Support tickets, surveys
3. **Analytics**: Usage patterns, behavior
4. **Retrospectives**: Team learnings
5. **Metrics Review**: Weekly/Monthly analysis

## 📊 Success Metrics

### **DORA Metrics (DevOps Research & Assessment)**

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| **Deployment Frequency** | Multiple/day | Weekly | Monthly | Yearly |
| **Lead Time** | < 1 hour | < 1 day | < 1 week | > 1 month |
| **MTTR** | < 1 hour | < 1 day | < 1 week | > 1 week |
| **Change Failure Rate** | 0-15% | 16-30% | 31-45% | > 45% |

## ✅ Best Practices

1. **Automate Everything**
   - CI/CD pipelines
   - Testing
   - Deployments
   - Monitoring setup

2. **Small, Frequent Changes**
   - Easier to debug
   - Faster rollback
   - Lower risk

3. **Shift Left**
   - Test early
   - Security from start
   - Quality gates

4. **Measure Everything**
   - Track metrics
   - Set baselines
   - Continuous improvement

5. **Blameless Culture**
   - Learn from failures
   - Share knowledge
   - Improve processes

## 📚 Đọc Thêm

- [Continuous Delivery Book](https://continuousdelivery.com/)
- [Google SRE Book](https://sre.google/books/)
- [Phoenix Project](https://www.amazon.com/Phoenix-Project-DevOps-Helping-Business/dp/0988262592)

---

**Tiếp theo**: [DevOps Tools Ecosystem →](../devops-tools/tools-overview.md)
