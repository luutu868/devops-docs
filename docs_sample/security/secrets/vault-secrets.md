# Secrets Management with HashiCorp Vault

## 1. Giới thiệu Secrets Management

**Secret** là bất kỳ thông tin nhạy cảm nào mà bạn muốn kiểm soát quyền truy cập một cách chặt chẽ, ví dụ:
-   Mật khẩu cơ sở dữ liệu
-   API keys
-   Chứng chỉ TLS (TLS certificates)
-   Tokens xác thực
-   Khóa SSH

**Secrets Management** là tập hợp các công cụ và phương pháp để quản lý vòng đời của các secret một cách an toàn, bao gồm: lưu trữ, tạo mới, cấp phát, xoay vòng (rotation), và thu hồi.

### Tại sao không nên hard-code secrets?

-   **Rò rỉ thông tin:** Bất kỳ ai có quyền truy cập vào mã nguồn đều có thể thấy secret.
-   **Khó khăn trong việc xoay vòng (rotation):** Khi một secret cần được thay đổi, bạn phải sửa mã nguồn, build lại, và triển khai lại ứng dụng.
-   **Phân tán secrets:** Các secret nằm rải rác ở nhiều nơi (code, file config, biến môi trường), khó kiểm soát và kiểm toán.
-   **Không có kiểm toán (auditing):** Khó để biết ai đã truy cập secret nào và khi nào.

## 2. HashiCorp Vault là gì?

**Vault** là một công cụ mã nguồn mở của HashiCorp, được thiết kế để quản lý secrets và bảo vệ dữ liệu nhạy cảm. Nó cung cấp một giao diện thống nhất để quản lý vòng đời của mọi loại secret.

### Các tính năng chính của Vault

-   **Secure Secret Storage (Lưu trữ Secret an toàn):** Vault mã hóa tất cả dữ liệu được lưu trữ. Nó không bao giờ lưu trữ secret ở dạng cleartext.
-   **Dynamic Secrets (Secrets động):** Vault có thể tạo ra các secret "theo yêu cầu" (on-demand) cho các hệ thống như AWS, cơ sở dữ liệu SQL, ... Các secret này có vòng đời ngắn (time-to-live - TTL) và sẽ tự động bị thu hồi sau khi hết hạn.
-   **Data Encryption (Mã hóa dữ liệu):** Vault có thể được sử dụng như một "dịch vụ mã hóa" (encryption as a service) để mã hóa và giải mã dữ liệu mà không cần lưu trữ nó.
-   **Leasing and Renewal (Cho thuê và Gia hạn):** Mọi secret trong Vault đều có một "lease" (hợp đồng thuê) đi kèm. Client phải gia hạn lease này định kỳ. Nếu không, lease sẽ hết hạn và secret sẽ bị thu hồi.
-   **Revocation (Thu hồi):** Vault có khả năng thu hồi một secret cụ thể hoặc toàn bộ cây secret.
-   **Detailed Audit Log (Log kiểm toán chi tiết):** Vault ghi lại mọi yêu cầu và phản hồi. Log này cho phép theo dõi chi tiết ai đã làm gì.

### Kiến trúc của Vault

-   **Storage Backend:** Nơi Vault lưu trữ dữ liệu đã được mã hóa. Vault hỗ trợ nhiều backend như Consul, etcd, S3, hoặc file system.
-   **Secret Engines:** Các thành phần chịu trách nhiệm quản lý các loại secret khác nhau (ví dụ: `kv` cho key-value, `aws` cho AWS credentials, `database` cho database credentials).
-   **Auth Methods:** Các phương pháp để client xác thực với Vault và nhận token (ví dụ: username/password, tokens, AppRole, AWS IAM, Kubernetes).
-   **Audit Devices:** Nơi Vault gửi các log kiểm toán.

## 3. Cài đặt và khởi tạo Vault (Dev Mode)

Chế độ `dev` rất hữu ích cho việc học và thử nghiệm. Nó chạy Vault trong bộ nhớ và tự động unseal. **Không bao giờ sử dụng chế độ `dev` cho môi trường production.**

### Cài đặt Vault

```bash
# Cài đặt trên Linux (Ubuntu/Debian)
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vault

# Cài đặt trên macOS
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

### Khởi chạy Vault ở chế độ Dev

```bash
# Khởi chạy server dev
vault server -dev
```

Output sẽ cung cấp cho bạn **Unseal Key** và **Root Token**. Hãy lưu lại **Root Token**.

```
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", ...
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem (not recommended for production)
                 Version: Vault v1.10.0

==> Vault server started! Log data will stream in below:

...
Unseal Key: ...
Root Token: s.xxxxxxxxxxxxxxxxxxxxxx  <-- LƯU LẠI TOKEN NÀY
...
```

### Cấu hình Vault CLI

Mở một terminal khác và thiết lập biến môi trường để CLI có thể giao tiếp với server.

```bash
# Địa chỉ của Vault server
export VAULT_ADDR='http://127.0.0.1:8200'

# Sử dụng Root Token để xác thực
export VAULT_TOKEN='s.xxxxxxxxxxxxxxxxxxxxxx' # Thay bằng Root Token của bạn

# Kiểm tra trạng thái
vault status
```

## 4. Sử dụng Vault cơ bản

### 4.1. Key/Value (KV) Secret Engine

Đây là secret engine đơn giản nhất, hoạt động như một kho lưu trữ key-value.

**Bật KV engine (phiên bản 2):**
Phiên bản 2 hỗ trợ versioning, cho phép bạn xem lại các phiên bản cũ của một secret.

```bash
# Bật kv-v2 engine tại path 'secret'
vault secrets enable -path=secret kv-v2
```

**Ghi một secret:**

```bash
# Ghi một secret tại path 'secret/myapp/database'
vault kv put secret/myapp/database username="db_user" password="SuperSecretPassword123!"
```

**Đọc một secret:**

```bash
# Đọc secret
vault kv get secret/myapp/database

# Chỉ lấy giá trị của một key
vault kv get -field=password secret/myapp/database
```

**Liệt kê secrets:**

```bash
vault kv list secret/myapp/
```

**Xóa một secret:**

```bash
vault kv delete secret/myapp/database
```

### 4.2. Dynamic Secrets: Database

Vault có thể kết nối đến một cơ sở dữ liệu và tự động tạo ra các cặp username/password động với các quyền và TTL được định nghĩa trước.

**Cấu hình:**
1.  **Bật Database secret engine:**
    ```bash
    vault secrets enable database
    ```

2.  **Cấu hình kết nối đến database:**
    (Giả sử bạn có một PostgreSQL server đang chạy)
    ```bash
    vault write database/config/postgresql \
        plugin_name=postgresql-database-plugin \
        allowed_roles="readonly-role" \
        connection_url="postgresql://root:rootpassword@localhost:5432/postgres?sslmode=disable"
    ```

3.  **Tạo một role:**
    Role này định nghĩa các câu lệnh SQL để tạo và xóa user, cùng với TTL.
    ```bash
    vault write database/roles/readonly-role \
        db_name=postgresql \
        creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
        default_ttl="1h" \
        max_ttl="24h"
    ```

4.  **Lấy credentials động:**
    Mỗi lần bạn chạy lệnh này, Vault sẽ tạo một user/password mới trong database.
    ```bash
    vault read database/creds/readonly-role
    ```
    Output sẽ chứa `username`, `password`, và `lease_duration`. Khi lease hết hạn, Vault sẽ tự động chạy câu lệnh xóa user khỏi database.

## 5. Tích hợp Vault với Kubernetes

Mục tiêu là cho phép các Pod trong Kubernetes có thể xác thực với Vault và lấy secrets một cách an toàn mà không cần hard-code token.

### Phương pháp xác thực Kubernetes (Kubernetes Auth Method)

Cơ chế này cho phép Vault tin tưởng vào Kubernetes. Một Pod có thể sử dụng Service Account Token của nó để xác thực với Vault.

**Luồng hoạt động:**
1.  Pod đọc Service Account Token (JWT) được Kubernetes tự động mount vào.
2.  Pod gửi JWT này đến Vault.
3.  Vault xác thực JWT bằng cách gọi đến một API của Kubernetes.
4.  Nếu JWT hợp lệ, Vault sẽ trả về một Vault Token cho Pod.
5.  Pod sử dụng Vault Token này để yêu cầu các secret cần thiết.

### Vault Agent Injector

Đây là cách dễ nhất để thực hiện luồng trên. Vault Agent Injector là một admission controller của Kubernetes. Khi một Pod được tạo với các annotation cụ thể, injector sẽ tự động:
-   Thêm một **init container** để lấy Vault token lúc khởi động.
-   Thêm một **sidecar container** (vault-agent) để:
    -   Quản lý vòng đời của Vault token (tự động gia hạn).
    -   Lấy secrets từ Vault và ghi chúng vào một volume được chia sẻ với container ứng dụng.

**Các bước cài đặt (sử dụng Helm):**
1.  **Thêm Helm repo của HashiCorp:**
    ```bash
    helm repo add hashicorp https://helm.releases.hashicorp.com
    helm repo update
    ```

2.  **Cài đặt Vault server vào Kubernetes:**
    (Khuyến khích cho production, nhưng bạn cũng có thể kết nối đến một Vault server bên ngoài)
    ```bash
    helm install vault hashicorp/vault
    ```

3.  **Cấu hình Kubernetes Auth Method trong Vault:**
    ```bash
    # Bật auth method
    vault auth enable kubernetes

    # Cấu hình để Vault có thể giao tiếp với K8s API
    vault write auth/kubernetes/config \
        kubernetes_host="https://<KUBERNETES_HOST>" \
        kubernetes_ca_cert=@/path/to/ca.crt \
        token_reviewer_jwt=@/path/to/reviewer-token.jwt
    ```
    (Khi cài bằng Helm, nhiều bước trong số này được tự động hóa)

4.  **Tạo một policy và role trong Vault:**
    -   **Policy (`myapp-policy.hcl`):** Định nghĩa quyền đọc secret cho ứng dụng.
        ```hcl
        path "secret/data/myapp/*" {
          capabilities = ["read"]
        }
        ```
        ```bash
        vault policy write myapp-policy myapp-policy.hcl
        ```
    -   **Role:** Liên kết policy này với một Service Account trong Kubernetes.
        ```bash
        vault write auth/kubernetes/role/myapp-role \
            bound_service_account_names=myapp-sa \
            bound_service_account_namespaces=default \
            policies=myapp-policy \
            ttl=24h
        ```

5.  **Triển khai ứng dụng với các annotation:**
    Trong file YAML của Deployment, thêm các annotation sau vào `template.metadata.annotations`:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      template:
        metadata:
          annotations:
            # Annotation để kích hoạt injector
            vault.hashicorp.com/agent-inject: "true"
            # Role trong Vault mà Pod sẽ sử dụng
            vault.hashicorp.com/role: "myapp-role"
            # Định nghĩa secret cần lấy và nơi ghi ra
            vault.hashicorp.com/agent-inject-secret-database.ini: "secret/data/myapp/database"
        spec:
          serviceAccountName: myapp-sa # Service Account của ứng dụng
          containers:
          - name: myapp
            image: myapp:1.0
            volumeMounts:
            - name: vault-secrets
              mountPath: "/etc/secrets"
              readOnly: true
          volumes:
          - name: vault-secrets
            emptyDir:
              medium: Memory
    ```

**Kết quả:**
-   Khi Pod `myapp` khởi động, Vault Agent Injector sẽ thêm các container cần thiết.
-   Vault agent sẽ xác thực với Vault bằng Service Account `myapp-sa`, nhận được một token với policy `myapp-policy`.
-   Agent sẽ đọc secret từ `secret/data/myapp/database` trong Vault.
-   Agent sẽ ghi secret này vào file `/etc/secrets/database.ini` bên trong Pod.
-   Ứng dụng của bạn giờ đây có thể đọc secret từ file đó mà không cần biết gì về Vault.

Quản lý secrets là một nền tảng quan trọng của DevSecOps. HashiCorp Vault cung cấp một giải pháp mạnh mẽ và linh hoạt để tự động hóa và bảo mật vòng đời của secrets trong các môi trường hiện đại như Kubernetes.
