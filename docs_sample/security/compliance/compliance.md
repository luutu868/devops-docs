# Compliance and Auditing in DevOps

## 1. Giới thiệu về Compliance (Tuân thủ)

**Compliance** (Tuân thủ) là quá trình đảm bảo rằng một tổ chức tuân theo các quy định, tiêu chuẩn, và luật lệ đã được thiết lập. Trong lĩnh vực công nghệ, điều này thường liên quan đến các tiêu chuẩn bảo mật và quyền riêng tư dữ liệu.

**Audit** (Kiểm toán) là quá trình kiểm tra và đánh giá độc lập để xác minh sự tuân thủ đó. Kết quả của một cuộc kiểm toán là một báo cáo chi tiết về các điểm mạnh, điểm yếu, và các khu vực cần cải thiện.

### Tại sao Compliance quan trọng trong DevOps?

-   **Yêu cầu pháp lý và hợp đồng:** Nhiều ngành (tài chính, y tế, chính phủ) có các quy định bắt buộc như PCI DSS, HIPAA, GDPR. Không tuân thủ có thể dẫn đến các khoản phạt nặng.
-   **Bảo vệ dữ liệu khách hàng:** Tuân thủ các tiêu chuẩn bảo mật giúp bảo vệ thông tin nhạy cảm của khách hàng, xây dựng lòng tin.
-   **Giảm thiểu rủi ro:** Các quy trình tuân thủ giúp xác định và giảm thiểu các rủi ro bảo mật và vận hành.
-   **Lợi thế cạnh tranh:** Chứng nhận tuân thủ (ví dụ: ISO 27001, SOC 2) có thể là một yếu tố khác biệt quan trọng trên thị trường.

### Thách thức của Compliance trong môi trường DevOps

-   **Tốc độ:** DevOps ưu tiên tốc độ và sự linh hoạt, trong khi tuân thủ truyền thống thường chậm và thủ công.
-   **Sự thay đổi liên tục:** Cơ sở hạ tầng và ứng dụng thay đổi liên tục, gây khó khăn cho việc theo dõi và kiểm toán.
-   **Cơ sở hạ tầng dưới dạng mã (IaC):** Việc quản lý tuân thủ cần được mở rộng sang cả mã nguồn định nghĩa hạ tầng.
-   **Phân tán:** Trong môi trường microservices, việc kiểm soát và kiểm toán trở nên phức tạp hơn.

Giải pháp là **Compliance as Code** - tự động hóa việc kiểm tra và thực thi tuân thủ như một phần không thể thiếu của pipeline CI/CD.

## 2. Các tiêu chuẩn tuân thủ phổ biến

-   **PCI DSS (Payment Card Industry Data Security Standard):** Bắt buộc đối với các tổ chức xử lý, lưu trữ, hoặc truyền tải dữ liệu thẻ thanh toán.
-   **HIPAA (Health Insurance Portability and Accountability Act):** Quy định về bảo vệ thông tin y tế cá nhân (Protected Health Information - PHI) tại Hoa Kỳ.
-   **GDPR (General Data Protection Regulation):** Quy định của Liên minh Châu Âu về bảo vệ dữ liệu và quyền riêng tư cho tất cả cá nhân trong EU.
-   **ISO/IEC 27001:** Tiêu chuẩn quốc tế về hệ thống quản lý an toàn thông tin (Information Security Management System - ISMS).
-   **SOC 2 (Service Organization Control 2):** Một framework kiểm toán tập trung vào 5 "nguyên tắc dịch vụ tin cậy" (Trust Services Criteria): an toàn, sẵn sàng, toàn vẹn xử lý, bảo mật và riêng tư.
-   **CIS Benchmarks (Center for Internet Security):** Các hướng dẫn cấu hình cứng (hardening guidelines) được công nhận rộng rãi cho nhiều loại hệ thống, bao gồm hệ điều hành, cloud, Kubernetes, ...

## 3. Compliance as Code

Compliance as Code là thực hành sử dụng mã nguồn để định nghĩa, quản lý, và tự động hóa các kiểm soát tuân thủ. Thay vì các checklist thủ công, các quy tắc tuân thủ được viết dưới dạng code và được tích hợp vào pipeline.

### Lợi ích

-   **Tốc độ và Hiệu quả:** Kiểm tra tuân thủ được thực hiện tự động trong vài phút, thay vì vài tuần hoặc vài tháng.
-   **Tính nhất quán:** Các quy tắc được áp dụng nhất quán trên tất cả các môi trường.
-   **Khả năng lặp lại:** Các kiểm tra có thể được chạy lại bất cứ lúc nào.
-   **Phát hiện sớm:** Các vấn đề tuân thủ được phát hiện sớm trong chu trình phát triển ("shift left").
-   **Khả năng kiểm toán:** Mọi thay đổi đối với quy tắc tuân thủ đều được theo dõi qua Git, cung cấp một dấu vết kiểm toán rõ ràng.

### Các công cụ cho Compliance as Code

| Công cụ | Lĩnh vực | Mô tả |
| :--- | :--- | :--- |
| **Open Policy Agent (OPA)** | Chính sách chung | Một công cụ mã nguồn mở đa năng để thực thi chính sách trên toàn bộ stack. |
| **Chef InSpec** | Hạ tầng | Một framework kiểm tra và kiểm toán để xác minh trạng thái hiện tại của hạ tầng. |
| **Terratest / tfsec** | IaC (Terraform) | Các công cụ để kiểm tra và phân tích tĩnh mã nguồn Terraform. |
| **kube-bench** | Kubernetes | Kiểm tra xem Kubernetes có được triển khai theo các hướng dẫn bảo mật của CIS hay không. |
- **Checkov** | IaC & Images | Phân tích tĩnh mã nguồn IaC (Terraform, CloudFormation, K8s) và Dockerfile để tìm các cấu hình sai. |

## 4. Open Policy Agent (OPA)

OPA là một "công cụ chính sách" (policy engine) mã nguồn mở, được tốt nghiệp từ CNCF. Nó cung cấp một ngôn ngữ bậc cao gọi là **Rego** để định nghĩa các chính sách và thực thi chúng.

OPA tách biệt việc định nghĩa chính sách khỏi việc thực thi chính sách. Ứng dụng của bạn sẽ gửi một câu hỏi (query) dưới dạng JSON đến OPA, và OPA sẽ trả lời dựa trên các chính sách và dữ liệu mà nó có.

### Tích hợp OPA Gatekeeper với Kubernetes

**OPA Gatekeeper** là một dự án con của OPA, hoạt động như một admission controller trong Kubernetes. Nó cho phép bạn thực thi các chính sách tùy chỉnh trên các đối tượng Kubernetes (Pods, Deployments, Services, ...) trước khi chúng được tạo hoặc cập nhật.

**Luồng hoạt động:**
1.  Admin định nghĩa các `ConstraintTemplate` (mẫu chính sách) bằng ngôn ngữ Rego.
2.  Admin tạo các `Constraint` (chính sách cụ thể) dựa trên các template đó, áp dụng cho các loại tài nguyên nhất định.
3.  Khi một người dùng `kubectl apply` một file YAML, `kube-apiserver` sẽ gửi một yêu cầu "admission review" đến Gatekeeper.
4.  Gatekeeper đánh giá yêu cầu dựa trên các `Constraint` đã định nghĩa.
5.  Nếu yêu cầu vi phạm chính sách, Gatekeeper sẽ từ chối nó và trả về một thông báo lỗi.

**Ví dụ:**
**Chính sách 1: Tất cả các image phải đến từ một registry tin cậy (`my-registry.com`).**

1.  **Tạo `ConstraintTemplate`:**
    ```yaml
    apiVersion: templates.gatekeeper.sh/v1
    kind: ConstraintTemplate
    metadata:
      name: k8srequiredlabels
    spec:
      crd:
        spec:
          names:
            kind: K8sRequiredLabels
          validation:
            openAPIV3Schema:
              properties:
                labels:
                  type: array
                  items:
                    type: string
      targets:
        - target: admission.k8s.gatekeeper.sh
          rego: |
            package k8srequiredlabels

            violation[{"msg": msg, "details": {"missing_labels": missing}}] {
              provided := {label | input.review.object.metadata.labels[label]}
              required := {label | label := input.parameters.labels[_]}
              missing := required - provided
              count(missing) > 0
              msg := sprintf("you must provide labels: %v", [missing])
            }
    ```

2.  **Tạo `Constraint`:**
    ```yaml
    apiVersion: constraints.gatekeeper.sh/v1beta1
    kind: K8sRequiredLabels
    metadata:
      name: ns-must-have-owner
    spec:
      match:
        kinds:
          - apiGroups: [""]
            kinds: ["Namespace"]
      parameters:
        labels: ["owner"]
    ```
Kết quả: Bất kỳ ai cố gắng tạo một `Namespace` mà không có label `owner` sẽ bị từ chối.

## 5. Auditing (Kiểm toán)

Auditing là quá trình thu thập và phân tích bằng chứng để đánh giá sự tuân thủ.

### Kubernetes Audit Logs

API Server của Kubernetes có thể tạo ra các log kiểm toán (audit logs) ghi lại mọi yêu cầu. Các log này là nguồn thông tin vô giá để:
-   Điều tra các sự cố bảo mật.
-   Theo dõi các thay đổi trong cluster.
-   Phát hiện các hành vi bất thường.

Mỗi bản ghi log chứa thông tin chi tiết:
-   **Ai:** User hoặc service account thực hiện yêu cầu.
-   **Làm gì:** Hành động (ví dụ: `create`, `delete`, `update`).
-   **Trên tài nguyên nào:** Loại tài nguyên (ví dụ: `pod`, `secret`) và tên của nó.
-   **Khi nào:** Dấu thời gian của yêu cầu.
-   **Từ đâu:** Địa chỉ IP nguồn.
-   **Kết quả:** Yêu cầu thành công hay thất bại.

**Cấu hình Audit Policy:**
Bạn có thể định nghĩa một `Audit Policy` để kiểm soát mức độ chi tiết của log.
-   `None`: Không ghi log.
-   `Metadata`: Chỉ ghi log metadata của yêu cầu (user, timestamp, resource, verb, ...).
-   `Request`: Ghi log metadata và request body.
-   `RequestResponse`: Ghi log metadata, request body, và response body.

**Ví dụ Audit Policy:**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Ghi log chi tiết cho các thay đổi liên quan đến secrets
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["secrets"]

  # Ghi log metadata cho các yêu cầu đọc pod
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

  # Bỏ qua các log không quan trọng
  - level: None
    resources:
    - group: ""
      resources: ["endpoints"]
```

### Các công cụ Auditing

-   **Falco:** Một công cụ phát hiện hành vi bất thường trong thời gian chạy. Falco sử dụng các quy tắc để theo dõi các system call và các sự kiện của Kubernetes. Nó có thể phát hiện các hoạt động như:
    -   Một shell được chạy bên trong một container.
    -   Ghi vào một file hoặc thư mục nhạy cảm.
    -   Một kết nối mạng bất ngờ được thiết lập.
    -   Một `secret` bị truy cập.
-   **ELK Stack / Loki & Grafana:** Các hệ thống quản lý log tập trung để thu thập, lưu trữ, và phân tích audit logs từ Kubernetes và các nguồn khác.
-   **Cloud Provider Tools:**
    -   **AWS:** CloudTrail (ghi lại các cuộc gọi API), GuardDuty (phát hiện mối đe dọa), AWS Config (theo dõi thay đổi tài nguyên).
    -   **GCP:** Cloud Audit Logs.
    -   **Azure:** Azure Monitor, Azure Policy.

## 6. Xây dựng một quy trình Tuân thủ và Kiểm toán

1.  **Xác định yêu cầu:** Xác định các tiêu chuẩn tuân thủ mà tổ chức của bạn cần tuân theo (PCI DSS, HIPAA, ...).
2.  **Mã hóa chính sách (Policy as Code):**
    -   Sử dụng OPA/Gatekeeper để định nghĩa các chính sách bảo mật và cấu hình cho Kubernetes.
    -   Sử dụng `tfsec` hoặc `checkov` để quét mã nguồn IaC.
    -   Tích hợp các kiểm tra này vào pipeline CI. Nếu vi phạm, pipeline sẽ thất bại.
3.  **Cấu hình cứng (Hardening):**
    -   Sử dụng `kube-bench` để kiểm tra cấu hình của cluster theo CIS Benchmarks.
    -   Áp dụng các `PodSecurityPolicy` hoặc `Pod Security Admission` để giới hạn quyền của các Pod.
4.  **Quét liên tục:**
    -   Quét container images để tìm lỗ hổng (Trivy, Grype).
    -   Quét các ứng dụng đang chạy để tìm các cấu hình sai.
5.  **Thu thập và Phân tích Log:**
    -   Bật Kubernetes Audit Logs và cấu hình một audit policy hợp lý.
    -   Tập trung hóa tất cả các log (audit logs, application logs, system logs) vào một nơi.
6.  **Giám sát và Cảnh báo:**
    -   Sử dụng Falco để phát hiện các hành vi bất thường trong thời gian chạy.
    -   Thiết lập cảnh báo cho các sự kiện quan trọng (ví dụ: một `secret` bị truy cập, một `cluster-admin` role binding được tạo).
7.  **Kiểm toán định kỳ:**
    -   Thực hiện các cuộc kiểm toán nội bộ và bên ngoài định kỳ.
    -   Sử dụng các log và báo cáo đã thu thập để cung cấp bằng chứng về sự tuân thủ.

Bằng cách tự động hóa tuân thủ và kiểm toán, bạn có thể đảm bảo rằng môi trường DevOps của mình vừa nhanh chóng, linh hoạt, vừa an toàn và tuân thủ các quy định cần thiết.
