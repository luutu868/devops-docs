<h1 align="center">
<img src="https://raw.githubusercontent.com/luutu868/devops-docs/main/docs_sample/images/career/graduate-cap.png" alt="DevOps Docs icon" width="170">
<br>Tài Liệu DevOps - VNPT VCI<br>
</h1>

[![GitHub](https://img.shields.io/badge/GitHub-luutu868%2Fdevops--docs-blue?logo=github)](https://github.com/luutu868/devops-docs)
[![MkDocs](https://img.shields.io/badge/MkDocs-Material-blue)](https://squidfunk.github.io/mkdocs-material/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](https://github.com/luutu868/devops-docs/blob/main/LICENSE)

---

## 📚 Giới Thiệu

Tài liệu DevOps toàn diện từ cơ bản đến nâng cao - Linux, Docker, Kubernetes, CI/CD, Monitoring, Security và nhiều hơn nữa.

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

## 📖 Nội Dung

- **Linux Basics**: Commands, Shell Scripting, Process Management
- **Docker & Kubernetes**: Containerization, Orchestration
- **CI/CD**: Jenkins, GitLab CI, GitHub Actions, ArgoCD
- **Infrastructure as Code**: Terraform, Ansible, CloudFormation
- **Monitoring & Logging**: Prometheus, Grafana, ELK Stack
- **Cloud Platforms**: AWS, GCP, Azure
- **Security & DevSecOps**: Container Security, K8s Security, Secrets Management
- **Best Practices & Roadmap**

---

## 🔗 Links

- **GitHub Repository**: [luutu868/devops-docs](https://github.com/luutu868/devops-docs)
- **MkDocs**: [Project documentation with Markdown](https://github.com/mkdocs/mkdocs/)
- **MkDocs Material Theme**: [A Material Design theme for MkDocs](https://github.com/squidfunk/mkdocs-material)
- **ReadTheDocs Theme**: [MkDocs ReadTheDocs theme](https://www.mkdocs.org/user-guide/choosing-your-theme/#readthedocs)

---

## 📝 License

- [MIT License](https://github.com/luutu868/devops-docs/blob/main/LICENSE)
- [The graduate cap icon](https://www.flaticon.com/free-icon/graduate-cap_62627) made by [Freepik](https://www.freepik.com/) from [www.flaticon.com](https://www.flaticon.com/) is licensed by [CC 3.0 BY](http://creativecommons.org/licenses/by/3.0/)

---

## 👨‍💻 About

**VNPT VCI DevOps Team**

- Email: security@vnpt.vn
- Website: [https://sec.vnpt.vn/](https://sec.vnpt.vn/)

© 2026 VNPT VCI - DevOps Team
