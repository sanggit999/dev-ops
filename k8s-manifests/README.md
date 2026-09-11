# Hướng Dẫn Bộ Manifests Kubernetes Chuẩn Doanh Nghiệp (Dành Cho Người Mới Bắt Đầu)

Chào mừng bạn! Thư mục này được tổ chức theo đúng **quy chuẩn thực tế của các kỹ sư DevOps chuyên nghiệp** tại các công ty công nghệ lớn.

---

## 1. TẠI SAO PHẢI ĐẶT TÊN CÓ TIỀN TỐ ĐÁNH SỐ: `00-`, `01-`, `02-`...?

Khi một hệ thống Kubernetes khởi động, các tài nguyên có **quan hệ phụ thuộc lẫn nhau (Dependency Chain)**:

```
[00-namespace.yaml]     --> Phải có CĂN PHÒNG (Namespace) trước tiên
        │
        ▼
[01-configmap.yaml]     --> Phải chuẩn bị sẵn ĐỒ ĐẠC CẤU HÌNH (Port, Domain)
[02-secret.yaml]        --> Phải chuẩn bị sẵn CHÌA KHÓA MẬT KHẨU (Password, Token)
        │
        ▼
[03-deployment.yaml]    --> ỨNG DỤNG KHỞI ĐỘNG (cần nạp ConfigMap & Secret ở trên)
        │
        ▼
[04-service.yaml]       --> MỞ CỬA NỘI BỘ (Cung cấp IP & Port ổn định cho Pods)
        │
        ▼
[05-ingress.yaml]       --> ĐÓN TIẾP KHÁCH & ĐỊNH TUYẾN DOMAIN (quiz-app.com)
        │
        ▼
[06-hpa.yaml]           --> TỰ ĐỘNG CO GIÃN PODS (Tự tăng/giảm Pods theo tải CPU)
```

> 💡 **Kinh nghiệm DevOps:** Kubernetes đọc file theo thứ tự bảng chữ cái `A-Z`. Khi bạn đặt tên `00-`, `01-`, `02-...`, bạn đảm bảo mọi thứ được khởi tạo theo đúng trình tự tự nhiên mà không bao giờ bị lỗi *"Thiếu ConfigMap"*, *"Không tìm thấy Namespace"* hay *"Ingress trỏ vào Service chưa tồn tại"*.

---

## 2. SO SÁNH: GÓC NHÌN NGƯỜI MỚI vs GÓC NHÌN DEVOPS CHUYÊN NGHIỆP

| Tiêu chí | Cách làm của người mới (Junior / Tự phát) | Chuẩn mực DevOps chuyên nghiệp (Senior / Enterprise) |
| :--- | :--- | :--- |
| **Cách triển khai** | Gõ từng lệnh `kubectl create...` trên PowerShell | Viết tất cả vào file `.yaml` và lưu vào Git (GitOps) |
| **Phân vùng** | Ném bừa vào namespace `default` | Mỗi dự án, mỗi môi trường (Dev/Staging/Prod) một **Namespace riêng biệt** |
| **Cấu hình & Mật khẩu** | Viết cứng (Hardcode) URL và Password vào code | Tách riêng ra **`ConfigMap`** và **`Secret`** (Nguyên tắc 12-Factor App) |
| **Quản lý tài nguyên** | Không cài đặt giới hạn | Luôn luôn đặt **`resources: requests & limits`** để tránh 1 Pod ăn hết RAM làm sập máy chủ |
| **Truy cập ứng dụng** | Nhớ từng địa chỉ IP & NodePort khô khan | Dùng **`Ingress`** để ánh xạ tên miền thương hiệu đẹp (`quiz-app.com`) |
| **Chịu tải cao** | Canh me máy chủ quá tải để gõ scale bằng tay | Dùng **`HPA`** tự động đẻ thêm Pods khi CPU vượt ngưỡng 70% |
| **Dọn dẹp dự án** | Đi tìm từng Pod, từng Service xóa bằng tay rất lâu | Chỉ cần 1 lệnh: `kubectl delete namespace quiz-app` là xóa sạch cả dự án trong 1 giây |

---

## 3. QUY TRÌNH VẬN HÀNH THỰC TẾ (CHỈ VỚI 3 BƯỚC)

Mở **PowerShell** tại thư mục `d:\DevOps`:

### Bước 1: Khởi động toàn bộ dự án chỉ bằng 1 câu lệnh duy nhất
```powershell
kubectl apply -f ./k8s-manifests/
```
*(Kubernetes sẽ tự động đọc cả 7 file và tạo lần lượt: Namespace $\rightarrow$ ConfigMap $\rightarrow$ Secret $\rightarrow$ 3 Pods Deployment $\rightarrow$ Service nội bộ $\rightarrow$ Ingress định tuyến tên miền $\rightarrow$ HPA tự động co giãn)*.

### Bước 2: Kiểm tra tình trạng hoạt động trong phân vùng `quiz-app`
```powershell
# 1. Xem tất cả tài nguyên cơ bản (Pods, Services, Deployments, ReplicaSets):
kubectl get all -n quiz-app

# 2. Xem luật định tuyến tên miền Ingress:
kubectl get ingress -n quiz-app
# (Kết quả sẽ thấy HOSTS: quiz-app.com và CLASS: nginx)

# 3. Xem bộ tự động co giãn Pods (HPA):
kubectl get hpa -n quiz-app
# (Kết quả sẽ thấy TARGETS: cpu: 1%/70%, MINPODS: 2, MAXPODS: 10)

# 4. Kiểm tra xem ConfigMap và Secret đã nạp vào Pod chưa:
kubectl exec -it <tên-pod-trong-danh-sách> -n quiz-app -- env

# 5. Mở ứng dụng bằng tên miền quiz-app.com:
# - Thêm vào file C:\Windows\System32\drivers\etc\hosts: 127.0.0.1 quiz-app.com
# - Mở 1 terminal quyền Admin chạy: minikube tunnel
# - Mở trình duyệt gõ: http://quiz-app.com
```

### Bước 3: Dọn dẹp sạch bóng toàn bộ dự án khi không còn dùng
```powershell
# Cách 1: Xóa theo toàn bộ thư mục cấu hình
kubectl delete -f ./k8s-manifests/

# Hoặc Cách 2 (Nhanh nhất): Xóa luôn cái Namespace
kubectl delete namespace quiz-app
```
*(Toàn bộ Pod, Service, Ingress, HPA, ConfigMap, Secret bên trong sẽ biến mất 100%, trả lại cụm Kubernetes tinh sạch như ban đầu!)*
