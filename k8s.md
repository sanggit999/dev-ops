# Cẩm Nang Tóm Tắt, Kiến Trúc & Thực Chiến Kubernetes

---

## 1. Tại Sao Docker Là Chưa Đủ?

> **Tình huống thực tế:** Giả sử bạn chạy 5 microservices của ứng dụng **Quiz App** bằng lệnh `docker run` trên một máy chủ VPS (máy chủ ảo) duy nhất. Các vấn đề nghiêm trọng sẽ phát sinh:

* **Sự cố giữa đêm (Crash & Downtime):**
  * Container bị sập do tràn bộ nhớ (Out Of Memory) hoặc lỗi mã nguồn.
  * Docker đơn thuần không có cơ chế tự điều phối, tự phát hiện và phục hồi trên nhiều máy chủ nếu máy chủ vật lý gặp sự cố.
* **Bùng nổ lưu lượng (Traffic Spike):**
  * Có 5.000 thí sinh đồng loạt vào thi. Làm sao tự động nhân bản (scale) từ 2 lên 10 containers và phân phối đều tải (Load Balancing) giữa các máy chủ?
* **Nâng cấp phiên bản (Zero-Downtime Deployment):**
  * Khi cập nhật mã nguồn mới, làm sao để quá trình làm bài của thí sinh không bị ngắt quãng dù chỉ 1 giây (Rolling Update - cập nhật cuốn chiếu)?

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
|          | (Lưu trữ KV) |   | (Lập lịch Pod)    |   | (Quản lý các trạng thái)|         |
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
  * **Lưu ý quan trọng:** Pod có tính chất **tạm thời (ephemeral)**. Khi Pod gặp sự cố, bị xóa hoặc được tạo mới ở lần kế tiếp, địa chỉ IP của Pod sẽ bị thay đổi hoàn toàn.

---

### 3.3. Khám Sức Khỏe Toàn Diện Máy Chủ Node (`kubectl describe node minikube`)

Khi cụm Kubernetes gặp sự cố Pod bị treo (`Pending`) hoặc máy chủ quá tải, lệnh đầu tiên kỹ sư DevOps dùng để "chẩn đoán bệnh" của máy chủ Node là:

```powershell
kubectl describe node minikube
```

Lệnh này cung cấp 5 khu vực thông tin sống còn:

1. **`Conditions` (Chỉ số sinh tồn của Node):**
   * `MemoryPressure: False` $\rightarrow$ Máy chủ không bị cạn kiệt RAM.
   * `DiskPressure: False` $\rightarrow$ Ổ cứng máy chủ còn rộng, không lo tràn đĩa.
   * `PIDPressure: False` $\rightarrow$ Không bị bùng nổ tiến trình Linux (hết Process ID).
   * `Ready: True` $\rightarrow$ Node hoàn toàn khỏe mạnh, sẵn sàng nhận Pods mới.
2. **`Capacity` vs `Allocatable` (Sức chứa phần cứng):**
   * Cho biết tổng số nhân CPU (ví dụ: 8 Cores), tổng RAM (ví dụ: ~8GB) và số lượng Pod tối đa máy chủ cho phép chạy (mặc định 110 Pods).
3. **`Non-terminated Pods` (Danh sách Pods đang cắm rễ):**
   * Liệt kê chi tiết từng Pod đang chạy trên Node (bao gồm cả các Pod hệ thống `kube-system`, `ingress-nginx` và ứng dụng `quiz-app`).
4. **`Allocated resources` (Bản đồ phân bổ tài nguyên):**
   * Thống kê tổng số % CPU/RAM đã được xí chỗ trước (`Requests`) và trần tối đa (`Limits`).
   * *Ý nghĩa:* Nếu tổng `Requests` đạt 100%, K8s sẽ từ chối nhận thêm Pod mới và đẩy các Pod sau vào trạng thái `Pending`.
5. **`Events` (Lịch sử sự kiện gần nhất):**
   * Ghi lại toàn bộ nhật ký quan trọng: Kéo image, khởi động lại kubelet, hay các cảnh báo phần cứng.

---

### 3.4. Bản Chất Của Pod & Bên Trong Container Chứa Những Gì?

Để tránh hiểu nhầm phổ biến của người mới bắt đầu:

> ⚠️ **Quy tắc sống còn:** Trong kiến trúc chuẩn, **chúng ta KHÔNG nhồi nhét tất cả Nginx, Backend, Database, Redis vào chung 1 container**. Mỗi dịch vụ là 1 container/Pod độc lập chuyên biệt.

```
+-----------------------------------------------------------------------+
|                             POD (Vỏ bọc)                              |
|                                                                       |
|   +---------------------------------------------------------------+   |
|   |                       CONTAINER (Cái ruột)                    |   |
|   |                                                               |   |
|   |  1. MÃ NGUỒN ỨNG DỤNG (Source Code / Binary)                  |   |
|   |     - File .py, .js, .java, file HTML, CSS, nginx.conf...     |   |
|   |                                                               |   |
|   |  2. MÔI TRƯỜNG THỰC THI (Runtimes & Libraries)                |   |
|   |     - Node/npm, Python/pip, JRE, nginx web server...          |   |
|   |                                                               |   |
|   |  3. HỆ ĐIỀU HÀNH LINUX SIÊU NHỎ (Mini Root Filesystem)        |   |
|   |     - Bản Linux tối giản (như Alpine chỉ 5MB, Debian rút gọn) |   |
|   |     - Chứa lệnh cơ bản: ls, cd, cat, sh, cấu trúc /etc, /var  |   |
|   |                                                               |   |
|   |  4. CẤU HÌNH & BIẾN MÔI TRƯỜNG (Configs & ENV)                |   |
|   |     - PORT, DB_HOST, DB_PASSWORD, Token kết nối mạng          |   |
|   +---------------------------------------------------------------+   |
+-----------------------------------------------------------------------+
```

#### Ý nghĩa thực tế của lệnh `kubectl exec -it <tên-pod> -- sh`:
* **Bản chất:** Đưa bạn từ môi trường Windows bên ngoài chui thẳng vào bên trong hệ điều hành Linux thu nhỏ của chính Container đó (dấu nhắc lệnh đổi thành `/ #`).
* **Ứng dụng thực chiến:**
  1. *Kiểm tra kết nối mạng nội bộ:* Đứng từ Backend thử `ping database-service` hoặc `curl http://other-service:8080`.
  2. *Kiểm tra biến môi trường:* Gõ `env` để xem các thông số cấu hình K8s cấp cho ứng dụng.
  3. *Thao tác Database / Cache trực tiếp:* Gõ `psql` hoặc `redis-cli` ngay trong ruột DB mà không cần mở cổng ra máy tính ngoài.
  4. *Đọc file log và cấu hình:* Gõ `cat /etc/nginx/nginx.conf` hoặc xem file mã nguồn.
* **Nguyên tắc vàng:** Chỉ dùng lệnh này để **khám bệnh (troubleshoot)**, tuyệt đối không sửa file trực tiếp bên trong container vì khi Pod restart, toàn bộ sửa đổi thủ công sẽ mất hết.

---

## 4. Các Thành Phần Mạng & Lưu Trữ Cốt Lõi

### 4.1. Giải pháp cho vấn đề "IP của Pod thay đổi": Kubernetes Service
* **Service** đóng vai trò là một đầu mối mạng cố định (cung cấp Địa chỉ IP ảo - Virtual IP và Tên miền nội bộ - DNS Name ổn định) đứng trước một tập hợp các Pod (được chọn thông qua nhãn dán `selector` và `labels`):
  * **ClusterIP (Mặc định):** Cung cấp địa chỉ IP nội bộ, chỉ cho phép giao tiếp giữa các dịch vụ bên trong nội bộ cụm Kubernetes.
  * **NodePort:** Mở một cổng cố định trên tất cả các Worker Node để các ứng dụng bên ngoài có thể truy cập trực tiếp qua địa chỉ `<Địa_Chỉ_IP_Node>:<Cổng_NodePort>` (dải cổng chuẩn: 30000–32767).
  * **LoadBalancer:** Tự động tích hợp với các nhà cung cấp điện toán đám mây (Cloud Providers như AWS, Google Cloud, Microsoft Azure) để cấp một địa chỉ IP công khai ra ngoài Internet.

### 4.2. Ingress & Ingress Controller
* Đóng vai trò là cổng định tuyến lớp ứng dụng (Reverse Proxy / API Gateway tại Layer 7 HTTP/HTTPS), điều hướng tên miền (Domain) và đường dẫn URL vào các Service tương ứng (ví dụ: `api.quiz.com` chuyển vào Service API, `quiz.com` chuyển vào Service Frontend), đồng thời hỗ trợ chứng chỉ bảo mật SSL/TLS.

### 4.3. Quản lý cấu hình & Lưu trữ dữ liệu
* **ConfigMap & Secret:** Tách biệt hoàn toàn các file cấu hình ứng dụng (ConfigMap) và thông tin bảo mật nhạy cảm (Secret: mật khẩu, API key, access token) ra khỏi container image để tăng tính bảo mật và linh hoạt.
* **PersistentVolume (PV) & PersistentVolumeClaim (PVC):** Cung cấp cơ chế gắn ổ đĩa lưu trữ bền vững (Persistent Storage) vào Pod, đảm bảo các dịch vụ lưu trữ dữ liệu (MySQL, PostgreSQL, MongoDB) không bị mất dữ liệu khi Pod bị tắt hoặc khởi động lại.

---

## 5. Thực Chiến Với Minikube Trên Máy Cá Nhân

### 5.1. Khởi tạo cụm phù hợp với phần cứng thực tế

> 💡 **Kinh nghiệm thực chiến:** Không nên dập khuôn cấp 8GB RAM nếu máy tính cá nhân chỉ có 16GB RAM tổng và đang trống 4–5GB (sẽ gây đơ máy). Công thức tối ưu cho laptop/PC thông thường là cấp **2 CPU và 3GB RAM (3072 MB)**:

```powershell
# Khởi động cụm Minikube tối ưu cho máy tính cá nhân
minikube start --driver=docker --cpus 2 --memory 3072
```

### 5.2. Quản lý tài nguyên & Vòng đời cụm
```powershell
# Xem trạng thái hoạt động của cụm
minikube status

# Xem IP của máy chủ Minikube
minikube ip

# TẠM DỪNG cụm khi học xong (giải phóng RAM/CPU cho máy tính, dữ liệu vẫn giữ nguyên):
minikube stop

# BẬT LẠI cụm khi muốn tiếp tục học (khởi động tức thì, không cần tải lại):
minikube start
```

### 5.3. Tiện ích mở rộng & Giao diện đồ họa (Dashboard)
```powershell
# Bật Ingress Controller định tuyến tên miền
minikube addons enable ingress

# Bật Metrics Server đo lường tài nguyên CPU/RAM
minikube addons enable metrics-server

# MỞ GIAO DIỆN ĐỒ HỌA TRÊN TRÌNH DUYỆT (Web GUI Dashboard):
minikube dashboard
```

---

## 6. Phân Biệt: Kubernetes Của Docker Desktop vs Minikube

Nhiều người dùng băn khoăn khi thấy tab "Kubernetes" xuất hiện ngay trong Docker Desktop:

| Tiêu chí | Kubernetes tích hợp trong Docker Desktop | Minikube (Đang sử dụng) |
| :--- | :--- | :--- |
| **Nền tảng** | Nhúng thẳng trong WSL2 VM của Docker | Chạy container độc lập (CNCF Official) |
| **Số lượng Node** | **Chỉ hỗ trợ 1 Node duy nhất** | **Hỗ trợ Multi-node (1 Master + nhiều Worker)** |
| **Mục đích phù hợp** | Chạy thử container nhanh | Học tập & Thực hành bài bản kiến trúc DevOps thực tế |
| **Tiện ích đi kèm** | Cơ bản | Rất phong phú (`ingress`, `metrics-server`, `dashboard`) |

> 💡 **Kiểm tra ngữ cảnh (Context) hiện tại:**
> ```powershell
> kubectl config get-contexts
> ```
> Cụm có dấu `*` ở cột `CURRENT` là nơi mà `kubectl` đang gửi lệnh điều khiển.

---

## 7. Quy Trình Thực Hành Từng Bước Dành Cho Người Mới

### Bước 1: Triển khai một ứng dụng Web
```powershell
# 1. Tạo Deployment chạy Nginx
kubectl create deployment my-web --image=nginx:alpine

# 2. Xem trạng thái triển khai
kubectl get deployments

# 3. Xem Pod đang chạy
kubectl get pods -o wide
```

### Bước 2: Mở cổng và truy cập
```powershell
# 1. Mở cổng NodePort
kubectl expose deployment my-web --type=NodePort --port=80

# 2. Xem Service được cấp cổng bao nhiêu
kubectl get svc

# 3. Lệnh mở thẳng trang Web lên trình duyệt Windows (Chrome/Edge):
minikube service my-web
```

### Bước 3: Trải nghiệm 2 "Phép Thuật" của Kubernetes
* **Mở rộng quy mô tức thì (Scale Out):**
  ```powershell
  # Nhân bản lên 4 bản sao cùng chia tải:
  kubectl scale deployment my-web --replicas=4
  kubectl get pods
  ```
* **Khả năng tự phục hồi (Self-Healing):**
  ```powershell
  # Thử xóa (giết) 1 Pod bất kỳ trong danh sách:
  kubectl delete pod <tên-pod>
  # Kiểm tra lại ngay lập tức: K8s tự động sinh ngay 1 Pod mới thay thế trong 1 giây!
  kubectl get pods
  ```

### Bước 4: Điều tra và gỡ lỗi (Debugging)
```powershell
# Xem log ứng dụng thời gian thực
kubectl logs -f <tên-pod>

# Xem lịch sử sự kiện và chi tiết cấu hình Pod
kubectl describe pod <tên-pod>

# Chui vào bên trong Container để khám phá
kubectl exec -it <tên-pod> -- sh
# (Gõ 'exit' để quay lại PowerShell)
```

### Bước 5: Dọn dẹp tài nguyên học tập
```powershell
kubectl delete svc my-web
kubectl delete deployment my-web
```

---

---

## 8. Chuẩn Mực Khai Báo YAML Manifest: Trái Tim Của DevOps Thực Chiến

Trong môi trường thực tế tại doanh nghiệp, kỹ sư DevOps **không gõ lệnh tạo Pod/Deployment bằng tay**, và **TUYỆT ĐỐI KHÔNG gộp chung tất cả vào một file YAML duy nhất**.

### 8.1. Tại sao chuẩn DevOps bắt buộc phải tách riêng từng file YAML?
1. **Phân quyền & Rà soát Git (GitOps & Code Review):**
   * Đội ngũ phụ trách mạng chỉ sửa `service.yaml`.
   * Đội ngũ bảo mật chỉ quản lý `secret.yaml`.
   * Lập trình viên chỉ cập nhật phiên bản image trong `deployment.yaml`.
   * Khi mở Pull Request (PR) trên GitHub/GitLab, lịch sử thay đổi (Git Diff) rõ ràng, minh bạch, không ai vô tình sửa đè lên phần việc của người khác.
2. **Quản lý vòng đời độc lập (Decoupling):**
   * Bạn có thể cập nhật cổng Service mà không làm trigger việc khởi động lại toàn bộ Pods của Deployment.
3. **Tương thích tuyệt đối với các công cụ CI/CD doanh nghiệp:**
   * Các công cụ tự động hóa triển khai như **Kustomize**, **Helm**, **ArgoCD** đều bắt buộc cấu trúc từng file tách rời để quản lý theo từng môi trường (Dev, Staging, Production).

---

### 8.2. Cấu trúc bộ Manifests chuẩn doanh nghiệp (`k8s-manifests/`)

Bộ file được tổ chức và đánh số thứ tự theo **chuỗi phụ thuộc (Dependency Chain)**:

```text
📁 k8s-manifests/
├── 00-namespace.yaml      # Bước 1: Tạo căn phòng độc lập cho dự án
├── 01-configmap.yaml      # Bước 2: Chuẩn bị biến môi trường tĩnh
├── 02-secret.yaml         # Bước 3: Chuẩn bị mật khẩu, token bảo mật (Base64)
├── 03-deployment.yaml     # Bước 4: Bản thiết kế Pod (nhúng ConfigMap & Secret)
└── 04-service.yaml        # Bước 5: Cung cấp đầu mối mạng và cân bằng tải
```

#### File 1: [00-namespace.yaml](file:///d:/DevOps/k8s-manifests/00-namespace.yaml) (Phân vùng dự án)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: quiz-app
  labels:
    project: quiz-system
```

#### File 2: [01-configmap.yaml](file:///d:/DevOps/k8s-manifests/01-configmap.yaml) (Cấu hình không nhạy cảm)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: quiz-config
  namespace: quiz-app          # Bắt buộc: Nằm trong phân vùng quiz-app
data:
  APP_ENV: "development"
  PORT: "80"
  MAX_CONCURRENT_USERS: "5000"
```

#### File 3: [02-secret.yaml](file:///d:/DevOps/k8s-manifests/02-secret.yaml) (Bảo mật & Mật khẩu dạng Base64)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: quiz-secret
  namespace: quiz-app
type: Opaque
data:
  # Chuỗi 'SuperSecretPass123' sau khi mã hóa Base64:
  DB_PASSWORD: U3VwZXJTZWNyZXRQYXNzMTIz
  JWT_SECRET: cXVpei1hcHAtand0LXNlY3JldC1rZXktMjAyNg==
```

#### File 4: [03-deployment.yaml](file:///d:/DevOps/k8s-manifests/03-deployment.yaml) (Quản lý Pods & Giới hạn tài nguyên)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quiz-backend
  namespace: quiz-app
  labels:
    app: quiz-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: quiz-backend
  template:
    metadata:
      labels:
        app: quiz-backend
    spec:
      containers:
      - name: quiz-backend-container
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:               # Bắt buộc: Giới hạn CPU/RAM tránh sập node
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        envFrom:                 # Nạp tự động ConfigMap & Secret vào trong ruột Pod
        - configMapRef:
            name: quiz-config
        - secretRef:
            name: quiz-secret
```

#### File 5: [04-service.yaml](file:///d:/DevOps/k8s-manifests/04-service.yaml) (Mở cổng mạng & Cân bằng tải)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: quiz-backend-service
  namespace: quiz-app
spec:
  type: NodePort
  selector:
    app: quiz-backend          # Tự động tìm tất cả Pod có nhãn này để chuyển khách vào
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 32100
```

#### File 6: [05-ingress.yaml](file:///d:/DevOps/k8s-manifests/05-ingress.yaml) (Cổng Lễ Tân: Định Tuyến Tên Miền & Reverse Proxy)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: quiz-ingress
  namespace: quiz-app
  annotations:
    # Báo cho K8s biết: Hãy dùng bộ điều khiển Nginx để xử lý luật này
    kubernetes.io/ingress.class: "nginx"
    # Tắt SSL redirect trên môi trường dev nội bộ (khi chưa gắn chứng chỉ SSL)
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx        # Khai báo chuẩn hiện đại của Kubernetes v1.18+
  rules:
  - host: quiz-app.com           # Tên miền khách gõ trên trình duyệt
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: quiz-backend-service   # Chuyển tiếp vào Service này
            port:
              number: 80                 # Cổng của Service
```
* **Bản chất DevOps:** Ingress đứng ở tầng 7 (HTTP/HTTPS Application). Thay vì người dùng phải gõ IP và Port kỳ dị như `192.168.49.2:32100`, Ingress soi trường `Host` trong HTTP Header để biết khách muốn vào `quiz-app.com` và chuyển tiếp chuẩn xác tới `quiz-backend-service:80`.

#### File 7: [06-hpa.yaml](file:///d:/DevOps/k8s-manifests/06-hpa.yaml) (Bộ Não Tự Co Giãn Pods Theo Tải - Autoscaling)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: quiz-backend-hpa
  namespace: quiz-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: quiz-backend          # Áp dụng cho Deployment quiz-backend
  minReplicas: 2                # Lúc vắng khách: Giữ tối thiểu 2 Pods (tiết kiệm RAM/CPU)
  maxReplicas: 10               # Lúc bùng nổ: Tự tăng tối đa 10 Pods (chống sập)
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Khi Metrics-Server báo CPU bình quân vượt quá 70% -> TỰ ĐẺ THÊM POD!
```
* **Bản chất DevOps:** HPA liên tục truy vấn dữ liệu từ `metrics-server`. Khi thí sinh ùa vào làm bài thi và CPU trung bình các Pods vượt ngưỡng 70%, HPA tự ra lệnh cho Deployment sinh thêm Pods (từ 2 lên đến 10 Pods). Khi hết thi, CPU hạ xuống, HPA tự thu hẹp Pods về lại 2 để giải phóng tài nguyên.

---

### 8.3. Bốn trường bắt buộc của MỌI file YAML Kubernetes
1. **`apiVersion`**: Phiên bản API của Kubernetes (`apps/v1` cho Deployment, `v1` cho Pod/Service/Namespace/ConfigMap/Secret, `networking.k8s.io/v1` cho Ingress, `autoscaling/v2` cho HPA).
2. **`kind`**: Loại tài nguyên muốn tạo (`Namespace`, `ConfigMap`, `Secret`, `Deployment`, `Service`, `Ingress`, `HorizontalPodAutoscaler`).
3. **`metadata`**: Định danh tài nguyên (Tên `name`, `namespace`, nhãn `labels`, chú giải `annotations`).
4. **`spec`** *(Specification)*: Trạng thái mong muốn (Desired State) mà bạn yêu cầu Kubernetes tự động duy trì.

---

### 8.4. Lệnh triển khai & Dọn dẹp cả thư mục chỉ bằng 1 dòng
```powershell
# Triển khai toàn bộ hệ thống theo đúng thứ tự 00 -> 06:
kubectl apply -f ./k8s-manifests/

# Xem toàn bộ tài nguyên bao gồm Ingress và HPA:
kubectl get all,ingress,hpa -n quiz-app

# Xóa sạch toàn bộ hệ thống đã khai báo:
kubectl delete -f ./k8s-manifests/

# Hoặc xóa tận gốc cả căn phòng dự án (nhanh nhất):
kubectl delete namespace quiz-app
```


### 8.5. Đưa Ứng Dụng Ra Ngoài Internet (Localhost vs Public Internet)
Nhiều người mới lầm tưởng lệnh `minikube service` là đã ra Internet:
* **`minikube service <tên-service> -n <namespace>`**: Chỉ mở trên **máy tính cá nhân (Localhost `127.0.0.1`)**, máy người khác hoặc điện thoại 4G **không thể truy cập được**.
* **Cách đưa ra Internet thật sự từ máy tính cá nhân (Miễn phí qua Tunnel):**
  1. *Bước 1 (Chuyển tiếp cổng):* `kubectl port-forward svc/<tên-service> 8080:80 -n <namespace>`
  2. *Bước 2 (Mở đường hầm toàn cầu):* `npx localtunnel --port 8080` (sẽ cấp một đường link HTTPS công khai để bất kỳ ai trên thế giới đều mở xem được).
* **Cách chuẩn doanh nghiệp (Production Cloud):** Triển khai lên Google Cloud GKE / AWS EKS $\rightarrow$ dùng `Service type: LoadBalancer` (cấp Public IP tĩnh) $\rightarrow$ Nginx Ingress Controller $\rightarrow$ Trỏ tên miền Domain và cấp SSL/TLS tự động qua `cert-manager`.

### 8.6. Chạy Tên Miền Ảo `quiz-app.com` Trên Máy Tính Cá Nhân (Local Dev)
Trong môi trường phát triển (Dev) hoặc chạy thử nghiệm, bạn không cần phải tốn tiền mua tên miền thật mà vẫn có thể gõ `http://quiz-app.com` ngay trên trình duyệt:

1. **Bước 1 (Khai báo Ingress):** Trong file [05-ingress.yaml](file:///d:/DevOps/k8s-manifests/05-ingress.yaml), đặt trường `host: quiz-app.com`.
2. **Bước 2 (Cấu hình Local DNS trên Windows):**
   * Mở Notepad bằng quyền **Administrator** (Chạy dưới quyền quản trị viên).
   * Mở tệp: `C:\Windows\System32\drivers\etc\hosts`.
   * Thêm dòng sau vào cuối tệp rồi lưu lại:
     ```text
     127.0.0.1 quiz-app.com
     ```
3. **Bước 3 (Mở cầu nối Ingress trên Windows):**
   * Do Minikube chạy qua Docker Desktop (trong máy ảo WSL2), cần mở đường hầm mạng:
     ```powershell
     minikube tunnel
     ```
   * *(Giữ cửa sổ terminal này chạy ngầm)*.
4. **Bước 4 (Truy cập):**
   * Mở trình duyệt gõ: `http://quiz-app.com` hoặc dùng lệnh PowerShell `curl.exe http://quiz-app.com`.

---

## 9. Khi Ứng Dụng Lên Internet: 5 Tiện Ích Sống Còn Của DevOps

Khi bước ra thế giới Internet, mục tiêu không chỉ là "chạy được" mà là **an toàn, chịu tải cao, tự động hóa và không bao giờ sập**:

1. **Tên miền đẹp & HTTPS (`Ingress` + `Cert-Manager`):**
   * Người dùng không bao giờ truy cập bằng IP và cổng lạ (`http://103.x.x.x:32100`). Họ cần tên miền `https://quizapp.com`.
   * `Ingress Controller` đứng làm cổng lễ tân phân luồng và `Cert-Manager` tự động xin/gia hạn chứng chỉ SSL miễn phí từ Let's Encrypt.
2. **Tự động co giãn theo lượng người dùng (`HPA - Horizontal Pod Autoscaler`):**
   * Ban ngày 5.000 người vào thi $\rightarrow$ K8s tự tăng từ 2 lên 15 Pods để chia tải. Ban đêm vắng khách $\rightarrow$ tự thu hẹp về 2 Pods để tiết kiệm tiền thuê máy chủ.
   * Lệnh kích hoạt nhanh:
     ```powershell
     kubectl autoscale deployment quiz-backend --cpu-percent=70 --min=2 --max=15 -n quiz-app
     ```
3. **Tự kiểm tra sức khỏe thông minh (`Liveness & Readiness Probes`):**
   * *Readiness Probe:* Hỏi container *"Bạn đã sẵn sàng nhận khách chưa?"* $\rightarrow$ Nếu chưa nạp xong cache thì không cho khách vào.
   * *Liveness Probe:* Hỏi container *"Bạn có bị đơ/treo mã nguồn không?"* $\rightarrow$ Nếu chết đứng thì K8s tự động bắn hạ và sinh Pod mới thay thế ngay lập tức.
4. **Giám sát biểu đồ & Cảnh báo (`Prometheus + Grafana + Alertmanager`):**
   * Đo lường thời gian phản hồi (latency), số lượng request/giây, mức ăn CPU/RAM.
   * Tự động gửi tin nhắn cảnh báo khẩn cấp vào **Telegram / Slack** của kỹ sư khi CPU > 85% hoặc tỷ lệ lỗi 500 tăng vọt.
5. **Thu thập nhật ký tập trung (`Centralized Logging` - Loki / EFK):**
   * Khi có 20 Pods chạy song song, không thể gõ `kubectl logs` từng Pod.
   * Toàn bộ log được hút về một chỗ duy nhất, giúp kỹ sư gõ tìm kiếm `status=500` ra ngay dòng code bị lỗi trong 0.1 giây.

---

## 10. Sự Thật Về Addons & Kubernetes Dashboard: DevOps Có Luôn Dùng Không?

Một thắc mắc phổ biến của người mới: *"Có phải lúc nào DevOps cũng cài Addons và bật Dashboard đồ họa không?"*

### 10.1. Minikube Addons vs Môi trường Cloud thật
* Lệnh `minikube addons enable...` **chỉ tồn tại trên Minikube** để giúp người học kích hoạt nhanh tính năng thử nghiệm.
* Trên các cụm Kubernetes thật của doanh nghiệp (AWS EKS, Google Cloud GKE), **không có lệnh này**. Thay vào đó, DevOps dùng công cụ quản lý gói **`Helm`** (`helm install ...`) để cài đặt các dịch vụ.

### 10.2. Tại sao trên Production, DevOps thường "NÉ" dùng Kubernetes Dashboard?
Nhiều người nghĩ lên Production thì Dashboard càng quan trọng, nhưng các công ty lớn thường **không cài Kubernetes Dashboard** vì 3 lý do:
1. **Lỗ hổng bảo mật cực lớn:** Đã từng có vụ tấn công chấn động vào tập đoàn **Tesla (năm 2018)** khi hacker xâm nhập chiếm quyền đào tiền ảo chỉ vì kỹ sư để lộ giao diện Kubernetes Dashboard ra Internet mà không bảo vệ mật khẩu.
2. **Vi phạm nguyên tắc "GitOps":** Mọi thay đổi trên Production phải được lưu vết trong Git (ai sửa, ngày nào, ai duyệt). Dùng Dashboard dễ sinh thói quen click chuột sửa tay trực tiếp trên web, gây ra lỗi lệch cấu hình (*Configuration Drift*).
3. **Dashboard mặc định quá sơ sài:** Không vẽ được biểu đồ đo lường chuyên sâu theo tuần/tháng.

### 10.3. Bộ 3 công cụ thay thế của Senior DevOps trong thực tế
* **Để xem biểu đồ & cảnh báo:** Dùng **`Grafana + Prometheus`** (Tiêu chuẩn số 1 thế giới).
* **Để thao tác giao diện an toàn trên máy cá nhân:** Dùng **`Lens Desktop`** (Phần mềm cài trên laptop, kết nối an toàn vào K8s qua file `kubeconfig`).
* **Để thao tác phím tắt siêu tốc trên Terminal:** Dùng **`k9s`** (Giao diện dòng lệnh TUI cực kỳ nhanh, bấm phím tắt xem log, restart pod trong 0.1 giây).
* **Trên nền tảng Cloud (GKE, AKS, EKS):** Dùng luôn giao diện Web Console bảo mật có sẵn của Google Cloud hoặc AWS.

---

## 11. Những Bài Học & Lưu Ý Kỹ Thuật Quan Trọng Đã Đúc Kết

1. **Về Image của Minikube (`kicbase`):**
   * Trong Docker Desktop, bạn có thể thấy 2 dòng chứa `kicbase` (1 dòng có tag và 1 dòng có hash digest `146cd636030e`). 
   * **Tuyệt đối không xóa image này**, vì đây là hệ điều hành nền tảng để Minikube chạy. Hai dòng này dùng chung 100% dung lượng (`SHARED SIZE`), không gây tốn ổ đĩa.
2. **Chính sách kéo Image (`imagePullPolicy`):**
   * Nếu nạp image thủ công vào Minikube, cần đảm bảo `imagePullPolicy: IfNotPresent` để Kubernetes ưu tiên dùng image có sẵn trên máy, tránh việc cố gắng tải lại qua Internet gây lỗi treo `ContainerCreating`.
3. **Phân biệt Build Cache và Lịch sử Builds trên Docker Desktop:**
   * Lệnh `docker system prune` chỉ xóa cache nhị phân trong engine.
   * Để xóa sạch mục **Builds** trên giao diện Docker Desktop, cần xóa lịch sử bản ghi:
     ```powershell
     docker buildx history rm --all
     ```
4. **Bảo tồn tài nguyên máy tính:**
   * Luôn nhớ chạy lệnh `minikube stop` khi kết thúc buổi làm việc để máy tính của bạn lấy lại 100% RAM và CPU.


