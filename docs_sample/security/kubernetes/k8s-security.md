# Kubernetes Security

## 1. Giới thiệu Kubernetes Security

Bảo mật Kubernetes là một chủ đề phức tạp, đòi hỏi sự chú ý ở nhiều lớp khác nhau của một cluster. Một cuộc tấn công có thể xảy ra ở bất kỳ đâu, từ mã nguồn ứng dụng cho đến hạ tầng cloud bên dưới. Mô hình "4C" của Cloud Native Security là một cách tiếp cận tốt để hiểu các lớp bảo mật này:

1.  **Cloud/Corporate Datacenter (Hạ tầng):** Lớp vật lý hoặc ảo (AWS, GCP, Azure, on-premise). Bảo mật ở lớp này là nền tảng cho mọi thứ khác.
2.  **Cluster (Cụm):** Bảo mật các thành phần của Kubernetes cluster, bao gồm control plane và worker nodes.
3.  **Container (Vùng chứa):** Bảo mật container runtime và các image được sử dụng.
4.  **Code (Mã nguồn):** Bảo mật ứng dụng chạy bên trong container.

Tài liệu này tập trung vào lớp **Cluster** và **Container** trong bối cảnh Kubernetes.

## 2. Kubernetes Attack Surface (Bề mặt tấn công)

Một cluster Kubernetes có nhiều thành phần có thể bị tấn công:
-   **Control Plane:**
    -   `kube-apiserver`: Cửa ngõ vào toàn bộ cluster. Nếu bị chiếm quyền, kẻ tấn công có toàn quyền kiểm soát.
    -   `etcd`: Nơi lưu trữ toàn bộ trạng thái của cluster, bao gồm cả secrets.
    -   `kube-scheduler`, `kube-controller-manager`: Có thể bị lợi dụng để lên lịch các pod độc hại.
-   **Worker Nodes:**
    -   `kubelet`: Agent chạy trên mỗi node, nhận lệnh từ API server. Nếu bị chiếm quyền, kẻ tấn công kiểm soát node đó.
    -   `Container Runtime`: Docker, containerd. Lỗ hổng ở đây có thể cho phép tấn công thoát khỏi container (container escape).
-   **Workloads (Ứng dụng):**
    -   Các Pod, Deployment, Service có thể chứa lỗ hổng hoặc được cấu hình sai.
    -   Secrets không được quản lý đúng cách.

## 3. Best Practices cho Kubernetes Security

### 3.1. Cluster Hardening (Cấu hình cứng Cluster)

**Control Plane:**
-   **Bảo vệ API Server:**
    -   Bật xác thực (authentication) và ủy quyền (authorization) mạnh mẽ.
    -   Sử dụng TLS cho mọi giao tiếp (`--tls-cert-file` và `--tls-private-key-file`).
    -   Hạn chế quyền truy cập vào API server từ các IP không tin cậy.
    -   Tắt các port không cần thiết, đặc biệt là port không bảo mật.
-   **Bảo vệ etcd:**
    -   Mã hóa dữ liệu lưu trữ (encryption at rest). Kubernetes 1.13+ hỗ trợ tính năng này.
    -   Sử dụng TLS để giao tiếp giữa API server và etcd.
    -   Phân tách etcd ra một mạng riêng.
-   **Sử dụng phiên bản Kubernetes mới nhất:** Luôn cập nhật để nhận các bản vá bảo mật mới nhất.

**Worker Nodes:**
-   **Cấu hình cứng hệ điều hành:** Sử dụng các hệ điều hành tối giản dành cho container (ví dụ: Bottlerocket, Container-Optimized OS).
-   **Hạn chế quyền truy cập SSH:** Chỉ cho phép truy cập từ các bastion host hoặc địa chỉ IP tin cậy.
-   **Tách biệt mạng:** Sử dụng các mạng riêng cho control plane và worker nodes.
-   **Quét lỗ hổng node:** Sử dụng các công cụ để quét lỗ hổng trên hệ điều hành của các node.

### 3.2. Authentication & Authorization (Xác thực & Ủy quyền)

**RBAC (Role-Based Access Control):**
RBAC là cơ chế ủy quyền tiêu chuẩn trong Kubernetes. Nó cho phép bạn định nghĩa các quyền (permissions) thông qua `Roles` và `ClusterRoles`, sau đó gán các quyền đó cho người dùng hoặc service account thông qua `RoleBindings` và `ClusterRoleBindings`.

**Nguyên tắc đặc quyền tối thiểu (Principle of Least Privilege):**
-   **Không sử dụng `default` service account:** Tạo các `ServiceAccount` riêng cho từng ứng dụng với các quyền tối thiểu cần thiết.
-   **Sử dụng `Roles` thay vì `ClusterRoles`:** `Roles` có phạm vi trong một namespace, giúp giới hạn quyền tốt hơn. Chỉ sử dụng `ClusterRoles` khi thực sự cần quyền trên toàn cluster.
-   **Tránh gán quyền `cluster-admin`:** Đây là quyền cao nhất, chỉ nên được sử dụng cho quản trị viên cluster.
-   **Không cấp quyền `create` pods cho người dùng không tin cậy:** Quyền này có thể bị lạm dụng để chiếm quyền của node.

**Ví dụ về RBAC:**
Tạo một `ServiceAccount` và `Role` cho một ứng dụng chỉ cần đọc `Pods` trong namespace `myapp`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: myapp
rules:
- apiGroups: [""] # "" chỉ định core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: myapp
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: myapp
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3.3. Pod Security Policies & Admission Controllers

**Pod Security Context:**
Đây là các thiết lập bảo mật được định nghĩa trong file YAML của Pod hoặc Deployment. Chúng cho phép bạn kiểm soát các khía cạnh bảo mật ở cấp độ Pod và container.

**Các thiết lập quan trọng:**
-   `runAsUser` / `runAsGroup`: Chạy container với một user ID cụ thể, không phải root.
-   `runAsNonRoot: true`: Bắt buộc container phải chạy với user không phải root.
-   `readOnlyRootFilesystem: true`: Gắn hệ thống file của container ở chế độ chỉ đọc.
-   `allowPrivilegeEscalation: false`: Ngăn chặn container có được các quyền mới sau khi khởi động.
-   `capabilities`: Giới hạn các Linux capabilities mà container có thể sử dụng. Bỏ các `capability` không cần thiết.

**Ví dụ `securityContext`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - "ALL" # Bỏ tất cả capabilities
        add:
        - "NET_BIND_SERVICE" # Chỉ thêm lại capability cần thiết
```

**Pod Security Admission (PSA):**
Từ Kubernetes 1.23, PSA là cơ chế tích hợp sẵn để thực thi các tiêu chuẩn bảo mật cho Pod. Nó định nghĩa 3 cấp độ:
-   `privileged`: Không có giới hạn, dành cho các workload hệ thống.
-   `baseline`: Ngăn chặn các leo thang đặc quyền đã biết.
-   `restricted`: Tuân thủ các best practice về cấu hình cứng Pod.

Bạn có thể áp dụng các chính sách này cho từng namespace.
```bash
# Áp dụng chính sách restricted cho namespace 'myapp'
kubectl label --overwrite ns myapp pod-security.kubernetes.io/enforce=restricted
```

### 3.4. Network Policies (Chính sách mạng)

Mặc định, tất cả các Pod trong một cluster có thể giao tiếp với nhau. `NetworkPolicy` cho phép bạn kiểm soát luồng traffic ở lớp 3 và 4 (IP và port) giữa các Pod.

**Nguyên tắc:**
-   **Từ chối mặc định (Default Deny):** Bắt đầu bằng cách chặn tất cả traffic vào (ingress) và ra (egress) của một namespace.
-   **Cho phép có chọn lọc:** Chỉ cho phép các kết nối cần thiết.

**Ví dụ:**
Chặn tất cả traffic vào namespace `myapp`, sau đó chỉ cho phép traffic từ các Pod có label `app=frontend` đến các Pod có label `app=backend` trên port 6379.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: myapp
spec:
  podSelector: {} # Chọn tất cả các pod trong namespace
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 6379
```

### 3.5. Secrets Management (Quản lý Secrets)

-   **Không hard-code secrets:** Tránh lưu secrets trong `Dockerfile`, mã nguồn, hoặc file YAML của Pod.
-   **Sử dụng Kubernetes Secrets:** Đây là cách cơ bản để lưu trữ secrets. Tuy nhiên, mặc định chúng chỉ được mã hóa base64, không an toàn.
-   **Bật mã hóa at rest cho Secrets:** Cấu hình API server để mã hóa secrets khi lưu trữ trong `etcd`.
-   **Sử dụng các công cụ quản lý secrets bên ngoài:**
    -   **HashiCorp Vault:** Giải pháp mạnh mẽ, cung cấp mã hóa, quản lý vòng đời, và cấp phát secrets động.
    -   **AWS Secrets Manager / GCP Secret Manager:** Các dịch vụ quản lý secrets của các nhà cung cấp cloud.
    -   Tích hợp các công cụ này vào Kubernetes thông qua các sidecar injector hoặc CSI driver.

### 3.6. Logging and Auditing (Ghi log và Kiểm toán)

-   **Bật Audit Logging:** API server có thể ghi lại mọi yêu cầu được thực hiện. Các log này rất quan trọng để điều tra các sự cố bảo mật.
-   **Tập trung hóa log:** Thu thập log từ tất cả các thành phần (control plane, nodes, applications) và đưa về một hệ thống quản lý log tập trung (ví dụ: ELK Stack, Loki).
-   **Sử dụng các công cụ giám sát bảo mật:**
    -   **Falco:** Công cụ phát hiện hành vi bất thường trong thời gian chạy (runtime security). Nó có thể phát hiện các hoạt động đáng ngờ như một shell được chạy trong container, ghi vào các thư mục nhạy cảm, hoặc các kết nối mạng bất thường.
    -   **Aqua Security, Sysdig, Tigera:** Các giải pháp thương mại cung cấp khả năng giám sát và bảo vệ toàn diện.

## 4. Các công cụ bảo mật Kubernetes

-   **kube-bench:** Chạy kiểm tra để xem cluster của bạn có tuân thủ các benchmark bảo mật từ CIS (Center for Internet Security) hay không.
-   **Kube-hunter:** Quét cluster để tìm các lỗ hổng bảo mật (pentesting).
-   **Trivy / Grype:** Quét container images để tìm lỗ hổng.
-   **Kyverno / OPA Gatekeeper:** Các admission controller mạnh mẽ cho phép bạn định nghĩa và thực thi các chính sách phức tạp.
-   **Falco:** Giám sát và phát hiện hành vi bất thường trong thời gian chạy.

Bảo mật Kubernetes là một quá trình liên tục, không phải là một dự án làm một lần. Bằng cách áp dụng các best practice và sử dụng các công cụ phù hợp, bạn có thể xây dựng một môi trường Kubernetes an toàn và đáng tin cậy.
