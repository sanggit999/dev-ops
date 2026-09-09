# Cẩm Nang Tóm Tắt & Kiến Trúc Kubernetes

---

## 1. Tại Sao Docker Là Chưa Đủ?

> **Tình huống thực tế:** Giả sử bạn chạy 5 microservices của ứng dụng **Quiz App** bằng lệnh `docker run` trên một máy chủ VPS (máy chủ ảo) duy nhất. Các vấn đề nghiêm trọng sẽ phát sinh:

* **Sự cố giữa đêm (Crash & Downtime):**
  * Container bị sập do tràn bộ nhớ (Out Of Memory) hoặc lỗi mã nguồn.
  * Docker đơn thuần không có cơ chế tự điều phối, tự phát hiện và phục hồi trên nhiều máy chủ nếu máy chủ vật lý gặp sự cố.
* **Bùng nổ lưu lượng (Traffic Spike):**
  * Có 5.000 thí sinh đồng loạt vào thi. Làm sao tự động nhân bản (scale) từ 2 lên 10 containers và phân phối đều tải (Load Balancing) giữa các máy chủ?
* **Nâng cấp phiên bản (Zero-Downtime Deployment):**
  * Khi cập nhật mã nguồn mới, làm sao để quá trình làm bài của thí sinh không bị ngắt quãng dù chỉ 1 giây (Rolling Update)?

---

## 2. Bản Chất Của Kubernetes

> **Nguồn gốc tên gọi:** **Kubernetes** bắt nguồn từ tiếng Hy Lạp mang nghĩa là *Người lái tàu* hoặc *Thuyền trưởng*. Tên viết tắt phổ biến **K8s** được tạo thành từ chữ cái đầu **K**, 8 chữ cái ở giữa (`ubernete`), và chữ cái cuối **s**.

Kubernetes đóng vai trò là hệ thống điều phối container (Container Orchestration) hoàn toàn tự động:

* **Giám sát liên tục (Monitoring):** Theo dõi trạng thái sức khỏe của hàng ngàn container 24/7.
* **Tự co giãn (Auto-scaling):** Tự động tăng hoặc giảm số lượng bản sao theo tải thực tế (mức sử dụng CPU, RAM, hoặc lưu lượng request).
* **Tự phục hồi (Self-healing):** Tự động khởi động lại container bị lỗi, thay thế và loại bỏ container không phản hồi kiểm tra sức khỏe (health check).
* **Cân bằng tải & Khám phá dịch vụ (Load Balancing & Service Discovery):** Phân phối lưu lượng mạng thông minh và tự động định tuyến địa chỉ IP nội bộ.

---

## 3. Kiến Trúc Tổng Quát Của Cụm Kubernetes

Một cụm Kubernetes tiêu chuẩn bao gồm 2 phần chính: **Control Plane** (Bộ não điều khiển / Master Node) và **Worker Nodes** (Máy chủ thực thi).

```
                             +-------------------------------+
                             |    DevOps / SRE (kubectl)     |
                             +---------------+---------------+
                                             |
                                             v
+-----------------------------------------------------------------------------------------+
|                               CONTROL PLANE (MASTER NODE)                               |
|                                                                                         |
|       +-------------------------------------------------------------------------+       |
|       |                           kube-apiserver                                |       |
|       |                  (Cổng giao tiếp trung tâm của cụm)                     |       |
|       +---------+--------------------+---------------------+--------------------+       |
|                 |                    |                     |                            |
|                 v                    v                     v                            |
|          +--------------+   +-------------------+   +-------------------------+         |
|          |     etcd     |   |   kube-scheduler  |   | kube-controller-manager |         |
|          | (Lưu trữ KV) |   |   (Lập lịch Pod)    |   | (Quản lý các trạng thái)|         |
|          +--------------+   +-------------------+   +-------------------------+         |
+-----------------------------------------------------------------------------------------+
                                             |
                     +-----------------------+-----------------------+
                     |                                               |
                     v                                               v
+-----------------------------------------+     +-----------------------------------------+
|              WORKER NODE 1              |     |              WORKER NODE 2              |
|                                         |     |                                         |
|  +-----------------------------------+  |     |  +-----------------------------------+  |
|  |              kubelet              |  |     |  |              kubelet              |  |
|  |  (Nhận lệnh từ apiserver, chạy Pod)  |  |  |  (Nhận lệnh từ apiserver, chạy Pod)  |  |
|  +-----------------------------------+  |     |  +-----------------------------------+  |
|  |         Container Runtime         |  |     |  |         Container Runtime         |  |
|  |     (containerd / CRI-O / v.v.)   |  |     |  |     (containerd / CRI-O / v.v.)   |  |
|  +-----------------------------------+  |     |  +-----------------------------------+  |
|  |            kube-proxy             |  |     |  |            kube-proxy             |  |
|  |     (Quy tắc mạng, cân bằng tải)  |  |     |  |     (Quy tắc mạng, cân bằng tải)  |  |
|  +-----------------------------------+  |     |  +-----------------------------------+  |
|                                         |     |                                         |
|  +-----------------------------------+  |     |  +-----------------------------------+  |
|  |   Pods: [Container 1][Container 2]|  |     |  |   Pods: [Container A][Container B]|  |
|  +-----------------------------------+  |     |  +-----------------------------------+  |
+-----------------------------------------+     +-----------------------------------------+
```

### 3.1. Control Plane (Bộ não điều khiển)

* **`kube-apiserver`**: Đầu não tiếp nhận mọi yêu cầu thông qua giao thức REST và gRPC (từ công cụ dòng lệnh `kubectl`, giao diện Dashboard, hoặc từ các Worker Node). Mọi giao tiếp trong toàn bộ cụm đều phải đi qua thành phần này.
* **`etcd`**: Cơ sở dữ liệu phân tán dạng Key-Value (Khóa - Giá trị) có độ tin cậy và tính nhất quán cao, lưu trữ toàn bộ dữ liệu cấu hình và trạng thái của toàn bộ cụm Kubernetes.
* **`kube-scheduler`**: Lắng nghe `kube-apiserver` để phát hiện các **Pod** mới được tạo ra mà chưa được gán vào máy chủ nào. Dựa trên tài nguyên phần cứng còn trống (CPU, RAM), chính sách ràng buộc (Affinity, Anti-Affinity, Taints, Tolerations) để chỉ định Worker Node phù hợp nhất cho Pod đó.
* **`kube-controller-manager`**: Tiến trình tập hợp nhiều bộ điều khiển trạng thái (Controllers) chạy ngầm nhằm liên tục đối soát giữa trạng thái mong muốn (Desired State) do người dùng khai báo và trạng thái thực tế (Current State) đang chạy:
  * *Node Controller:* Giám sát tình trạng sống/chết và hiệu năng của các Worker Node.
  * *Deployment Controller:* Quản lý vòng đời cập nhật ứng dụng (cập nhật cuốn chiếu - rolling update, quay về bản cũ - rollback).
  * *ReplicaSet Controller:* Đảm bảo luôn duy trì chính xác số lượng bản sao Pod mong muốn.
  * *StatefulSet Controller & DaemonSet Controller:* Quản lý các dạng ứng dụng đặc biệt (ứng dụng lưu trữ trạng thái như cơ sở dữ liệu, hoặc ứng dụng chạy đúng một bản sao trên mỗi máy chủ như bộ thu thập log, dịch vụ giám sát).
  * *Namespace Controller:* Quản lý việc phân chia không gian tài nguyên logic giữa các dự án hoặc môi trường (Development, Staging, Production).

### 3.2. Worker Nodes (Máy chủ thực thi)

* **`kubelet`**: Tiến trình đại diện chạy trực tiếp trên từng Worker Node. Thành phần này tiếp nhận chỉ thị (thông qua cấu hình chi tiết `PodSpec`) từ `kube-apiserver` để ra lệnh cho Container Runtime tải image, bật/tắt container và liên tục báo cáo tình trạng của Pod cũng như Node về lại Control Plane.
* **Container Runtime**: Phần mềm thực thi container cấp thấp tương thích chuẩn CRI (Container Runtime Interface) như **`containerd`** hoặc **`CRI-O`** (lưu ý: Kubernetes từ phiên bản 1.24 trở đi đã loại bỏ hoàn toàn việc tích hợp trực tiếp Dockershim).
* **`kube-proxy`**: Xử lý mạng, duy trì các quy tắc định tuyến mạng (thông qua iptables hoặc IPVS) để cân bằng tải lưu lượng truy cập tới các Pod thông qua Kubernetes Service.
* **`Pod`**:
  * Đơn vị nhỏ nhất và cơ bản nhất mà Kubernetes quản lý và triển khai (Kubernetes không quản lý từng container đơn lẻ mà đóng gói chúng vào bên trong Pod).
  * Một Pod có thể chứa một hoặc nhiều container (các container trong cùng Pod dùng chung Network Namespace, chung địa chỉ IP và có thể chia sẻ cùng ổ đĩa Volume).
  * **Lưu ý quan trọng:** Pod có tính chất **tạm thời (ephemeral)**. Khi Pod gặp sự cố, bị xóa hoặc được tạo mới ở lần kế tiếp, địa chỉ IP của Pod sẽ bị thay đổi hoàn toàn.

---

## 4. Các Thành Phần Quan Trọng Khác (Bổ Sung Cốt Lõi)

### 4.1. Giải pháp cho vấn đề "IP của Pod thay đổi": Kubernetes Service
* **Service** đóng vai trò là một đầu mối mạng cố định (cung cấp Địa chỉ IP ảo - Virtual IP và Tên miền nội bộ - DNS Name ổn định) đứng trước một tập hợp các Pod (được chọn thông qua nhãn dán `selector` và `labels`):
  * **ClusterIP (Mặc định):** Cung cấp địa chỉ IP nội bộ, chỉ cho phép giao tiếp giữa các dịch vụ bên trong nội bộ cụm Kubernetes.
  * **NodePort:** Mở một cổng cố định trên tất cả các Worker Node để các ứng dụng bên ngoài có thể truy cập trực tiếp qua địa chỉ `<Địa_Chỉ_IP_Node>:<Cổng_NodePort>`.
  * **LoadBalancer:** Tự động tích hợp với các nhà cung cấp điện toán đám mây (Cloud Providers như AWS, Google Cloud, Microsoft Azure) để cấp một địa chỉ IP công khai ra ngoài Internet.

### 4.2. Ingress & Ingress Controller
* Đóng vai trò là cổng định tuyến lớp ứng dụng (Reverse Proxy / API Gateway tại Layer 7 HTTP/HTTPS), điều hướng tên miền (Domain) và đường dẫn URL vào các Service tương ứng (ví dụ: `api.quiz.com` chuyển vào Service API, `quiz.com` chuyển vào Service Frontend), đồng thời hỗ trợ chứng chỉ bảo mật SSL/TLS.

### 4.3. Quản lý cấu hình & Lưu trữ dữ liệu
* **ConfigMap & Secret:** Tách biệt hoàn toàn các file cấu hình ứng dụng (ConfigMap) và thông tin bảo mật nhạy cảm (Secret: mật khẩu, API key, access token) ra khỏi container image để tăng tính bảo mật và linh hoạt.
* **PersistentVolume (PV) & PersistentVolumeClaim (PVC):** Cung cấp cơ chế gắn ổ đĩa lưu trữ bền vững (Persistent Storage) vào Pod, đảm bảo các dịch vụ lưu trữ dữ liệu (MySQL, PostgreSQL, MongoDB) không bị mất dữ liệu khi Pod bị tắt hoặc khởi động lại.

---

## 5. Thiết Lập Môi Trường Kubernetes Cục Bộ (Local) Với Minikube

Minikube là công cụ chính thức giúp xây dựng nhanh cụm Kubernetes thử nghiệm (hỗ trợ cả mô hình đa máy chủ - Multi-node) ngay trên máy tính cá nhân bằng cách sử dụng Docker hoặc công nghệ ảo hóa (Hypervisor).

### 5.1. Câu lệnh khởi tạo cụm

```bash
# Khởi động cụm Minikube với 2 máy chủ (1 Master Node + 1 Worker Node), cấp 4 nhân CPU và 8GB RAM
minikube start --nodes 2 --driver=docker --cpus 4 --memory 8192
```

*Giải thích chi tiết các tham số:*
* `--nodes 2`: Tạo cụm gồm 2 máy chủ ảo để kiểm thử tính năng phân phối trên nhiều node.
* `--driver=docker`: Sử dụng nền tảng Docker Desktop sẵn có trên máy để chạy các máy chủ giả lập.
* `--cpus 4`: Cấp phát 4 nhân vi xử lý (vCPU) cho cụm.
* `--memory 8192`: Cấp phát 8GB RAM (8192 Megabytes) cho cụm.

### 5.2. Kích hoạt các tiện ích mở rộng (Addons)

```bash
# Kích hoạt Ingress Controller (Nginx Ingress) để điều hướng tên miền vào cụm
minikube addons enable ingress

# Kích hoạt Metrics Server để đo lường tài nguyên CPU và RAM (dùng cho tính năng tự động co giãn Pod)
minikube addons enable metrics-server
```

---

## 6. Làm Quen Với Bộ Lệnh Điều Khiển `kubectl` Cơ Bản

`kubectl` (Kubernetes Control Command-Line Interface) là công cụ dòng lệnh tiêu chuẩn dùng để gửi chỉ thị trực tiếp đến `kube-apiserver` nhằm điều khiển toàn bộ cụm.

### 6.1. Giám sát trạng thái cụm & Tài nguyên

```bash
# Xem danh sách tất cả các máy chủ (Nodes) đang hoạt động trong cụm
kubectl get nodes

# Xem chi tiết thông số kỹ thuật và tình trạng sức khỏe của một Node
kubectl describe node <tên-node>

# Xem danh sách tất cả các Pod đang chạy trong một Namespace cụ thể (ví dụ: quiz-app)
kubectl get pods -n quiz-app

# Xem danh sách Pod kèm thông tin chi tiết về Địa chỉ IP và Node đang chứa Pod (-o wide)
kubectl get pods -n quiz-app -o wide

# Xem chi tiết thông số kỹ thuật và toàn bộ lịch sử sự kiện (Events) của một Pod cụ thể
kubectl describe pod <tên-pod> -n quiz-app
```

### 6.2. Triển khai & Áp dụng cấu hình dạng khai báo (Declarative)

```bash
# Triển khai mới hoặc cập nhật tài nguyên từ file cấu hình YAML
kubectl apply -f deployment.yaml

# Xóa các tài nguyên đã được khai báo bên trong file cấu hình YAML
kubectl delete -f deployment.yaml
```

### 6.3. Gỡ lỗi & Kiểm tra nhật ký hoạt động (Debugging & Logs)

```bash
# Xem và theo dõi nhật ký hoạt động (log) trực tiếp theo thời gian thực của một Pod
kubectl logs -f <tên-pod> -n quiz-app

# Truy cập trực tiếp vào giao diện dòng lệnh (Terminal / Shell) bên trong Container đang chạy
kubectl exec -it <tên-pod> -n quiz-app -- /bin/sh

# Xem mức tiêu thụ tài nguyên CPU và RAM thực tế của các Node (yêu cầu metrics-server đã bật)
kubectl top nodes

# Xem mức tiêu thụ tài nguyên CPU và RAM thực tế của các Pod trong Namespace quiz-app
kubectl top pods -n quiz-app
```
