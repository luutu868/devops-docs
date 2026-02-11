# Container Security: Image Scanning

## 1. Giới thiệu Container Security

Bảo mật container là một tập hợp các phương pháp và công cụ để bảo vệ cơ sở hạ tầng container, từ lúc xây dựng image cho đến khi chạy ứng dụng trong môi trường production. Bảo mật container cần được xem xét ở nhiều lớp khác nhau:

1.  **Infrastructure Security (Bảo mật hạ tầng):** Bảo vệ máy chủ host, mạng, và storage.
2.  **Host OS Security (Bảo mật hệ điều hành host):** Cấu hình cứng hệ điều hành, giới hạn quyền truy cập, sử dụng các kernel security module (SELinux, AppArmor).
3.  **Container Runtime Security (Bảo mật trình chạy container):** Bảo vệ Docker daemon, containerd. Giới hạn quyền của container (capabilities, seccomp).
4.  **Image Security (Bảo mật Image):** Quét lỗ hổng, sử dụng base image tin cậy, không chứa thông tin nhạy cảm.
5.  **Application Security (Bảo mật ứng dụng):** Bảo mật mã nguồn ứng dụng chạy bên trong container.

**Image Scanning** là một trong những bước quan trọng nhất trong "Image Security".

## 2. Image Scanning là gì?

Image Scanning là quá trình phân tích một container image để phát hiện các lỗ hổng bảo mật đã biết (CVEs - Common Vulnerabilities and Exposures) trong các gói phần mềm của hệ điều hành (OS packages) và các thư viện của ngôn ngữ lập trình (language-specific dependencies).

Quá trình này nên được tích hợp vào chu trình CI/CD để đảm bảo rằng chỉ những image đã được kiểm tra và an toàn mới được đẩy lên registry và triển khai. Đây là một phần cốt lõi của triết lý "Shift Left" trong DevSecOps - phát hiện và sửa lỗi bảo mật càng sớm càng tốt.

### Tại sao Image Scanning quan trọng?

-   **Phát hiện sớm lỗ hổng:** Tìm ra các vấn đề bảo mật trước khi chúng được triển khai ra môi trường production, giảm thiểu rủi ro bị tấn công.
-   **Tuân thủ chính sách:** Đảm bảo các image tuân thủ các tiêu chuẩn bảo mật và quy định của tổ chức (ví dụ: không có lỗ hổng nghiêm trọng "Critical").
-   **Giảm bề mặt tấn công (Attack Surface):** Bằng cách loại bỏ các gói và thư viện không cần thiết, bạn làm cho image nhỏ gọn và an toàn hơn.
-   **Tự động hóa bảo mật:** Tích hợp vào CI/CD giúp quá trình kiểm tra bảo mật diễn ra tự động, nhất quán và không làm chậm quá trình phát triển.

## 3. Các công cụ Image Scanning phổ biến

Có nhiều công cụ mã nguồn mở và thương mại cho việc quét image. Dưới đây là một số công cụ phổ biến:

| Công cụ | Nhà phát triển | Đặc điểm |
| :--- | :--- | :--- |
| **Trivy** | Aqua Security | Mã nguồn mở, rất phổ biến, nhanh, dễ sử dụng, hỗ trợ nhiều loại lỗ hổng. |
| **Grype** | Anchore | Mã nguồn mở, mạnh mẽ, tập trung vào độ chính xác cao. |
| **Clair** | Quay (Red Hat) | Mã nguồn mở, thường được tích hợp với các container registry. |
| **Docker Scout** | Docker | Tích hợp sẵn trong Docker Desktop và Docker Hub, cung cấp phân tích chi tiết. |
| **Snyk** | Snyk | Thương mại, mạnh mẽ, hỗ trợ quét cả mã nguồn, image và IaC. |

Trong tài liệu này, chúng ta sẽ tập trung vào **Trivy** vì tính phổ biến, đơn giản và hiệu quả của nó.

## 4. Hướng dẫn sử dụng Trivy

Trivy là một công cụ quét lỗ hổng đơn giản và toàn diện. Nó có thể phát hiện lỗ hổng trong:
-   Các gói của hệ điều hành (Alpine, RHEL, CentOS, Debian, Ubuntu,...)
-   Các thư viện của ngôn ngữ lập trình (npm, yarn, pip, maven, gradle, go,...)
-   File cấu hình IaC (Terraform, CloudFormation, Kubernetes,...)
-   Thông tin nhạy cảm (secrets) bị hard-code.

### Cài đặt Trivy

```bash
# Cài đặt trên Linux (Ubuntu/Debian)
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Cài đặt trên macOS
brew install trivy
```

### Quét một Image

Cú pháp cơ bản để quét một image từ Docker Hub hoặc registry khác:

```bash
# Quét image nginx phiên bản mới nhất
trivy image nginx:latest

# Quét image python với phiên bản cụ thể
trivy image python:3.9-slim
```

**Ví dụ output:**
```
python:3.9-slim (debian 11.2)
============================
Total: 2 (UNKNOWN: 0, LOW: 1, MEDIUM: 1, HIGH: 0, CRITICAL: 0)

+---------+------------------+----------+-------------------+---------------+--------------------------------+
| LIBRARY | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |             TITLE              |
+---------+------------------+----------+-------------------+---------------+--------------------------------+
| bsdutils| CVE-2021-37750   | MEDIUM   | 2.36.1-8+deb11u1  |               | util-linux: integer overflow   |
|         |                  |          |                   |               | in get_sem_elements()          |
+---------+------------------+----------+-------------------+---------------+--------------------------------+
| libcomerr2| CVE-2022-1304  | LOW      | 1.46.2-2          |               | e2fsprogs: out-of-bounds       |
|         |                  |          |                   |               | read/write in                  |
|         |                  |          |                   |               | e2fsck/rehash.c                |
+---------+------------------+----------+-------------------+---------------+--------------------------------+
```

### Các tùy chọn hữu ích

-   **Lọc theo mức độ nghiêm trọng (Severity):**
    ```bash
    # Chỉ hiển thị các lỗ hổng từ HIGH trở lên
    trivy image --severity HIGH,CRITICAL nginx:latest
    ```

-   **Bỏ qua các lỗ hổng chưa có bản vá (unfixed):**
    ```bash
    # Chỉ hiển thị các lỗ hổng đã có bản vá
    trivy image --ignore-unfixed python:3.9-slim
    ```

-   **Định dạng output:**
    ```bash
    # Output dưới dạng JSON
    trivy image --format json -o results.json nginx:latest

    # Output dưới dạng bảng (mặc định)
    trivy image --format table nginx:latest

    # Tạo báo cáo HTML (yêu cầu template)
    trivy image --format template --template "@contrib/html.tpl" -o report.html nginx:latest
    ```

-   **Thoát với mã lỗi nếu có lỗ hổng:**
    Tùy chọn này rất quan trọng khi tích hợp vào CI/CD.
    ```bash
    # Thoát với mã lỗi 1 nếu có lỗ hổng CRITICAL
    trivy image --exit-code 1 --severity CRITICAL nginx:latest
    ```

## 5. Tích hợp Image Scanning vào CI/CD

Mục tiêu là tự động thất bại (fail) pipeline nếu một image chứa các lỗ hổng nghiêm trọng.

### Ví dụ với GitHub Actions

Tạo một file workflow `.github/workflows/image-scan.yml`:

```yaml
name: "Container Image Scan"

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Build an image from Dockerfile
        run: |
          docker build -t myapp:${{ github.sha }} .

      - name: Install Trivy
        run: |
          sudo apt-get update
          sudo apt-get install -y wget apt-transport-https gnupg lsb-release
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
          echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
          sudo apt-get update
          sudo apt-get install -y trivy

      - name: Scan image with Trivy
        run: |
          trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:${{ github.sha }}
```

**Giải thích workflow:**
1.  **Checkout code:** Lấy mã nguồn từ repository.
2.  **Build image:** Xây dựng image từ `Dockerfile` và tag nó với mã SHA của commit.
3.  **Install Trivy:** Cài đặt Trivy vào môi trường runner.
4.  **Scan image:**
    -   `trivy image ... myapp:${{ github.sha }}`: Quét image vừa được build.
    -   `--exit-code 1`: Nếu Trivy tìm thấy lỗ hổng, nó sẽ thoát với mã lỗi 1, làm cho step này thất bại.
    -   `--severity CRITICAL,HIGH`: Chỉ định rằng pipeline chỉ thất bại nếu có lỗ hổng ở mức `CRITICAL` hoặc `HIGH`.

## 6. Best Practices cho Image Security

1.  **Sử dụng Base Image tối giản và tin cậy (Minimal and Trusted Base Images):**
    -   Bắt đầu với các image nhỏ nhất có thể, ví dụ: `alpine`, `distroless`, hoặc `scratch`. Image càng nhỏ, bề mặt tấn công càng nhỏ.
    -   Luôn sử dụng các image chính thức từ các nhà cung cấp uy tín (ví dụ: `python`, `node`, `nginx` từ Docker Hub).

2.  **Sử dụng Multi-Stage Builds:**
    -   Tách biệt môi trường build và môi trường runtime.
    -   Chỉ copy những file thực thi và thư viện cần thiết vào image cuối cùng. Điều này giúp loại bỏ các công cụ build, mã nguồn và các thư viện không cần thiết khỏi image production.

    **Ví dụ Dockerfile với multi-stage build:**
    ```dockerfile
    # --- Build Stage ---
    FROM golang:1.19 as builder
    WORKDIR /app
    COPY . .
    RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

    # --- Production Stage ---
    FROM alpine:latest
    WORKDIR /app
    COPY --from=builder /app/myapp .
    
    # Chạy ứng dụng
    CMD ["./myapp"]
    ```

3.  **Chạy container với người dùng không phải root (Non-Root User):**
    -   Thêm một người dùng không có quyền root vào image và sử dụng nó để chạy ứng dụng. Điều này giảm thiểu thiệt hại nếu container bị xâm nhập.

    **Ví dụ trong Dockerfile:**
    ```dockerfile
    FROM alpine:latest
    
    # Tạo user và group
    RUN addgroup -S appgroup && adduser -S appuser -G appgroup
    
    # Chuyển sang user không phải root
    USER appuser
    
    WORKDIR /home/appuser/app
    COPY . .
    
    CMD ["./myapp"]
    ```

4.  **Quét Image thường xuyên và tự động:**
    -   Tích hợp quét image vào mọi giai đoạn: CI (khi build), CD (trước khi deploy), và trong registry (quét định kỳ).
    -   Cập nhật cơ sở dữ liệu lỗ hổng của công cụ quét thường xuyên (`trivy image --download-db-only`).

5.  **Không nhúng thông tin nhạy cảm (Secrets) vào Image:**
    -   Không bao giờ hard-code API keys, mật khẩu, hoặc tokens vào `Dockerfile` hoặc mã nguồn.
    -   Sử dụng các cơ chế quản lý secrets như Docker secrets, Kubernetes Secrets, hoặc HashiCorp Vault.

Bằng cách áp dụng các phương pháp này, bạn có thể cải thiện đáng kể mức độ bảo mật của các container và giảm thiểu rủi ro trong môi trường sản xuất.
