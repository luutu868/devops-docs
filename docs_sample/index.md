<h1 align="center">
<img src="../images/career/graduate-cap.png" alt="DevOps Docs icon" width="170">
<br>Tài Liệu DevOps - VNPT VCI<br>
</h1>

[![GitHub](https://img.shields.io/badge/GitHub-luutu868%2Fdevops--docs-blue?logo=github)](https://github.com/luutu868/devops-docs)
[![MkDocs](https://img.shields.io/badge/MkDocs-ReadTheDocs-blue)](https://www.mkdocs.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](https://github.com/luutu868/devops-docs/blob/main/LICENSE)

---

## 📚 Giới Thiệu

Chào mừng bạn đến với tài liệu DevOps toàn diện của VNPT VCI! Đây là nguồn tài liệu được biên soạn kỹ lưỡng để giúp bạn từ người mới bắt đầu trở thành một DevOps Engineer chuyên nghiệp.

**Repository:** [https://github.com/luutu868/devops-docs](https://github.com/luutu868/devops-docs)

---

## 🚀 Quick Start

### Chạy với Docker Compose (Khuyến nghị)

```bash
# Clone repository
git clone https://github.com/luutu868/devops-docs.git
cd devops-docs

# Tạo network (nếu chưa có)
docker network create cloudflare_redisinsight_app-network

# Chạy MkDocs với Docker Compose
docker compose up -d

# Xem logs
docker compose logs -f mkdocs

# Truy cập tại: http://localhost:8000
```

### Các lệnh Docker Compose hữu ích

```bash
# Dừng service
docker compose stop

# Khởi động lại
docker compose restart mkdocs

# Xóa container
docker compose down

# Rebuild và chạy lại
docker compose up -d --build
```

### Chạy với Python (Alternative)

```bash
# Cài đặt dependencies
pip install -r requirements.txt

# Chạy development server
mkdocs serve -f mkdocs-sample.yml

# Build static site
mkdocs build -f mkdocs-sample.yml
```

---

## 📖 Nội Dung Tài Liệu

### 🎯 **Giới Thiệu DevOps**
- DevOps là gì?
- Văn hóa DevOps
- DevOps Lifecycle
- DevOps Tools Ecosystem

### 🔧 **Linux & Networking**
- Linux Basics & Commands
- Shell Scripting & Process Management
- Network Fundamentals & Security
- Load Balancing (Nginx, HAProxy)

### 📦 **Version Control & Git**
- Git Basics & Advanced Commands
- Git Workflows & Best Practices
- GitHub & GitLab

### 🐳 **Docker & Kubernetes**
- Docker Fundamentals, Images & Containers
- Docker Compose & Security
- Kubernetes Architecture & Objects
- K8s Networking, Storage & Advanced Topics
- Managed Kubernetes (EKS, GKE, AKS)

### 🔄 **CI/CD**
- CI/CD Fundamentals
- Jenkins Pipeline
- GitLab CI/CD & GitHub Actions
- ArgoCD & GitOps
- CI/CD Best Practices

### 🏗️ **Infrastructure as Code**
- Terraform
- Ansible Playbooks
- AWS CloudFormation
- IaC Best Practices

### 📊 **Monitoring & Logging**
- Monitoring Basics
- Prometheus & Grafana
- ELK Stack
- APM & Uptime Monitoring
- Log Management Best Practices

### ☁️ **Cloud Platforms**
- AWS Basics
- Google Cloud Platform
- Microsoft Azure
- Multi-Cloud Strategies

### 🔒 **Security & DevSecOps**
- Security Fundamentals
- Container & Kubernetes Security
- Secrets Management (Vault)
- Compliance & Audit
- Vulnerability Scanning

### 💾 **Databases & Automation**
- Relational & NoSQL Databases
- Database DevOps & Migrations
- Python, PowerShell & Go for DevOps

### 💡 **Best Practices & Career**
- DevOps Best Practices
- Case Studies & Success Stories
- Common Pitfalls & Anti-patterns
- DevOps Roadmap
- Learning Resources

---

## 🔗 Links

- **GitHub Repository**: [luutu868/devops-docs](https://github.com/luutu868/devops-docs)
- **MkDocs**: [Project documentation with Markdown](https://github.com/mkdocs/mkdocs/)
- **ReadTheDocs Theme**: [MkDocs ReadTheDocs theme](https://www.mkdocs.org/user-guide/choosing-your-theme/#readthedocs)

---

## 📝 License

- [MIT License](https://github.com/luutu868/devops-docs/blob/main/LICENSE)
- [The graduate cap icon](https://www.flaticon.com/free-icon/graduate-cap_62627) made by [Freepik](https://www.freepik.com/) from [www.flaticon.com](https://www.flaticon.com/) is licensed by [CC 3.0 BY](http://creativecommons.org/licenses/by/3.0/)

---

## 👨‍💻 About VNPT VCI

**VNPT VCI DevOps Team**

- Email: security@vnpt.vn
- Website: [https://sec.vnpt.vn/](https://sec.vnpt.vn/)

© 2026 VNPT VCI - DevOps Team
