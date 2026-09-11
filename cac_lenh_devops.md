# Tổng Hợp Toàn Bộ Các Câu Lệnh DevOps & Kubernetes Đã Sử Dụng

Tài liệu này lưu trữ toàn bộ các câu lệnh PowerShell / CLI đã thực thi trong quá trình kiểm tra hệ thống, cài đặt, nâng cấp phiên bản, dọn dẹp môi trường, triển khai ứng dụng dạng Declarative (YAML) và đưa dịch vụ ra Internet trên máy tính của bạn.

---

## MỤC LỤC

1. [Kiểm Tra Hệ Thống & Phiên Bản Phần Mềm](#1-kiểm-tra-hệ-thống--phiên-bản-phần-mềm)
2. [Cài Đặt & Nâng Cấp Minikube, kubectl, Docker Desktop](#2-cài-đặt--nâng-cấp-minikube-kubectl-docker-desktop)
3. [Khởi Chạy & Vận Hành Cụm Kubernetes (Minikube)](#3-khởi-chạy--vận-hành-cụm-kubernetes-minikube)
4. [Quản Lý Máy Chủ Nodes Chuyên Sâu](#4-quản-lý-máy-chủ-nodes-chuyên-sâu)
5. [Quản Lý Phân Vùng Độc Lập (Namespace)](#5-quản-lý-phân-vùng-độc-lập-namespace)
6. [Triển Khai Ứng Dụng Dạng Khai Báo (YAML Manifests)](#6-triển-khai-ứng-dụng-dạng-khai-báo-yaml-manifests)
7. [Khám Bệnh, Gỡ Lỗi & Truy Cập Container (Exec & Logs)](#7-khám-bệnh-gỡ-lỗi--truy-cập-container-exec--logs)
8. [Mở Cổng & Đưa Ứng Dụng Ra Mạng Internet (Tunneling)](#8-mở-cổng--đưa-ứng-dụng-ra-mạng-internet-tunneling)
9. [Dọn Dẹp Triệt Để Docker (Containers, Images, Volumes, Cache)](#9-dọn-dẹp-triệt-để-docker-containers-images-volumes-cache)
10. [Xóa Sạch Lịch Sử Builds & File Logs Hệ Thống](#10-xóa-sạch-lịch-sử-builds--file-logs-hệ-thống)

---

## 1. Kiểm Tra Hệ Thống & Phiên Bản Phần Mềm

### 1.1. Kiểm tra vị trí các công cụ đang cài đặt
```powershell
# Kiểm tra xem winget, docker, kubectl, minikube có sẵn trong PATH và nằm ở đâu
Get-Command winget, docker, kubectl, minikube -ErrorAction SilentlyContinue | Select-Object Name, Source
```

### 1.2. Kiểm tra phiên bản các công cụ
```powershell
# Kiểm tra phiên bản Docker Engine
docker --version

# Kiểm tra phiên bản kubectl client
kubectl version --client

# Kiểm tra phiên bản minikube
minikube version

# Kiểm tra thông tin chi tiết và phiên bản Docker Desktop trên winget
winget list -q Docker
```

### 1.3. Kiểm tra thông số phần cứng & môi trường
```powershell
# Kiểm tra dung lượng đã dùng và dung lượng trống của tất cả các ổ đĩa (C, D, E)
Get-PSDrive -PSProvider FileSystem | Select-Object Name, Used, Free, Root

# Kiểm tra dung lượng RAM tổng và RAM còn trống trên Windows
$os = Get-CimInstance Win32_OperatingSystem; [PSCustomObject]@{TotalRAM_GB = [math]::Round($os.TotalVisibleMemorySize/1MB, 2); FreeRAM_GB = [math]::Round($os.FreePhysicalMemory/1MB, 2)}

# Kiểm tra số nhân CPU thực tế
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors

# Kiểm tra quyền hạn của PowerShell hiện tại (có phải Administrator không)
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)

# Xem danh sách tất cả các đường dẫn trong biến môi trường PATH
$env:PATH -split ';'
```

---

## 2. Cài Đặt & Nâng Cấp Minikube, kubectl, Docker Desktop

### 2.1. Cài đặt Minikube
```powershell
# Cách 1: Cài tự động bằng winget
winget install -e --id Kubernetes.minikube --accept-package-agreements --accept-source-agreements

# Cách 2: Tải trực tiếp file binary minikube mới nhất vào thư mục .local\bin (đã có trong PATH, không cần quyền Admin)
curl.exe -L -o "$env:USERPROFILE\.local\bin\minikube.exe" "https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe"
```

### 2.2. Tải & Cập nhật kubectl phiên bản mới nhất (v1.37.0)
```powershell
# Kiểm tra phiên bản stable mới nhất của Kubernetes
curl.exe -L "https://dl.k8s.io/release/stable.txt"

# Tải file binary kubectl v1.37.0 mới nhất vào .local\bin
curl.exe -L -o "$env:USERPROFILE\.local\bin\kubectl.exe" "https://dl.k8s.io/release/v1.37.0/bin/windows/amd64/kubectl.exe"

# Kiểm tra lại phiên bản của kubectl vừa tải
& "$env:USERPROFILE\.local\bin\kubectl.exe" version --client
```

### 2.3. Nâng cấp Docker Desktop lên 4.90.0
```powershell
# Lệnh tải bản nâng cấp qua winget
winget upgrade --id Docker.DockerDesktop --accept-package-agreements --accept-source-agreements

# Khởi chạy bộ cài đặt có giao diện trên màn hình để xác nhận UAC và bấm Update
Start-Process -FilePath "$env:LOCALAPPDATA\Temp\WinGet\Docker.DockerDesktop.4.90.0\Docker Desktop Installer.exe"
```

---

## 3. Khởi Chạy & Vận Hành Cụm Kubernetes (Minikube)

### 3.1. Khởi động cụm Kubernetes (Tối ưu phần cứng cá nhân)
```powershell
# Khởi động cụm Minikube tối ưu cho laptop/PC (cấp 2 CPU và 3GB RAM, tránh nghẽn RAM Windows):
minikube start --driver=docker --cpus 2 --memory 3072

# Khởi động cụm Minikube mô phỏng đa máy chủ (Multi-node: 1 Master + 1 Worker):
minikube start --nodes 2 --driver=docker --cpus 4 --memory 8192
```

### 3.2. Quản lý vòng đời cụm (Tiết kiệm điện và RAM)
```powershell
# Kiểm tra trạng thái hoạt động của cụm
minikube status

# Xem IP của máy chủ Minikube
minikube ip

# TẠM DỪNG cụm khi học xong (giải phóng 100% RAM/CPU cho máy tính, dữ liệu vẫn giữ nguyên):
minikube stop

# BẬT LẠI cụm khi muốn học tiếp (khởi động tức thì, không cần tải lại image):
minikube start
```

### 3.3. Tiện ích mở rộng & Giao diện đồ họa (Dashboard)
```powershell
# Kích hoạt Ingress Controller (Nginx Ingress) định tuyến tên miền
minikube addons enable ingress

# Kích hoạt Metrics Server đo lường tài nguyên CPU/RAM
minikube addons enable metrics-server

# MỞ GIAO DIỆN ĐỒ HỌA TRÊN TRÌNH DUYỆT (Web GUI Dashboard):
minikube dashboard
```

---

## 4. Quản Lý Máy Chủ Nodes Chuyên Sâu

```powershell
# 1. Xem danh sách nhanh tất cả các Node (Kiểm tra trạng thái Ready hay NotReady)
kubectl get nodes

# 2. Xem chi tiết mở rộng: Địa chỉ IP nội bộ, Hệ điều hành và Container Runtime (containerd)
kubectl get nodes -o wide

# 3. "Khám sức khỏe toàn diện" của Node (Dung lượng CPU/RAM, Pods đang cắm rễ, % tài nguyên đã cấp):
kubectl describe node minikube

# Mẹo lọc nhanh các vùng thông tin quan trọng của Node trên PowerShell:
# - Xem chỉ số sinh tồn (Conditions: MemoryPressure, DiskPressure, Ready):
kubectl describe node minikube | Select-String -Pattern "Conditions:" -Context 0,6

# - Xem bản đồ tài nguyên đã phân bổ (Allocated resources: CPU Requests, Memory Requests):
kubectl describe node minikube | Select-String -Pattern "Allocated resources:" -Context 0,8

# - Xem các sự kiện cảnh báo hoặc lỗi gần nhất của máy chủ (Events):
kubectl describe node minikube | Select-String -Pattern "Events:" -Context 0,5

# 4. Xem mức độ ngốn CPU và RAM thực tế theo thời gian thực (Real-time metrics):
kubectl top nodes
```

---

## 5. Quản Lý Phân Vùng Độc Lập (Namespace)

```powershell
# 1. Xem danh sách tất cả các Namespace hiện có trong cụm
kubectl get namespaces
# (viết tắt: kubectl get ns)

# 2. Tạo nhanh một Namespace mới bằng lệnh
kubectl create namespace quiz-app

# 3. Xem TẤT CẢ tài nguyên (Pods, Service, Deployment, ReplicaSet) nằm trong một Namespace cụ thể:
kubectl get all -n quiz-app

# 4. Xóa sạch toàn bộ một Namespace (Dọn dẹp cả dự án trong 1 nốt nhạc, không để lại rác):
kubectl delete namespace quiz-app
```

---

## 6. Triển Khai Ứng Dụng Dạng Khai Báo (YAML Manifests)

### 6.1. Triển khai theo bộ thư mục chuẩn DevOps
```powershell
# Triển khai toàn bộ dự án từ thư mục manifests theo thứ tự (Namespace -> Config -> Secret -> Deployment -> Service):
kubectl apply -f ./k8s-manifests/

# Xóa toàn bộ tài nguyên được khai báo trong thư mục manifests:
kubectl delete -f ./k8s-manifests/
```

### 6.2. Triển khai theo từng file YAML đơn lẻ
```powershell
# Triển khai từ 1 file YAML duy nhất (ví dụ: demo-app.yaml):
kubectl apply -f demo-app.yaml

# Xóa tài nguyên khai báo trong file demo-app.yaml:
kubectl delete -f demo-app.yaml
```

---

## 7. Khám Bệnh, Gỡ Lỗi & Truy Cập Container (Exec & Logs)

```powershell
# 1. Xem nhật ký hoạt động (Logs) của Pod theo thời gian thực (Real-time logs)
kubectl logs -f <tên-pod> -n quiz-app

# 2. Xem chi tiết thông số kỹ thuật và Lịch sử sự kiện (Events) của Pod xem bị lỗi gì
kubectl describe pod <tên-pod> -n quiz-app

# 3. CHUI THẲNG VÀO RUỘT CONTAINER để gõ lệnh Linux (Terminal / Shell):
kubectl exec -it <tên-pod> -n quiz-app -- sh
# (Gõ "exit" để thoát ra lại PowerShell)

# 4. Kiểm tra nhanh toàn bộ biến môi trường (ENV) đã nạp từ ConfigMap/Secret vào Pod chưa:
kubectl exec -it <tên-pod> -n quiz-app -- env

# 5. Xem mức tiêu thụ CPU/RAM thực tế của từng Pod trong Namespace:
kubectl top pods -n quiz-app
```

### 7.2. "Đánh Chết Pod" Để Kiểm Chứng Tự Phục Hồi (Self-Healing / Chaos Test)
```powershell
# 1. Bắn hạ 1 Pod thông thường (Graceful termination - K8s cấp 30 giây để dọn dẹp trước khi tắt):
kubectl delete pod <tên-pod> -n quiz-app

# 2. Bắn hạ khẩn cấp ngay lập tức (Force Kill - ép chết tức thì trong 0 giây):
kubectl delete pod <tên-pod> -n quiz-app --force --grace-period=0

# 3. Giả lập ứng dụng bị Crash mã nguồn (Bắn hạ tiến trình PID 1 trong ruột container):
kubectl exec -it <tên-pod> -n quiz-app -- kill 1
# (Quan sát lệnh: Số RESTARTS của Pod sẽ tự động tăng lên +1 và container tự khởi động lại!)

# 4. Bắn hạ hàng loạt TOÀN BỘ Pods trong Namespace cùng một lúc:
kubectl delete pods --all -n quiz-app

# 5. Mở màn hình radar theo dõi trực tiếp quá trình Pod cũ chết đi và Pod mới hồi sinh (Live Watch):
kubectl get pods -n quiz-app -w
```

---

## 8. Mở Cổng & Đưa Ứng Dụng Ra Mạng Internet (Tunneling)

### 8.1. Mở xem trực tiếp trên máy tính cá nhân (Localhost)
```powershell
# Lệnh Minikube tự động mở trình duyệt web Windows tới trang dịch vụ:
minikube service quiz-backend-service -n quiz-app
# (Chỉ máy tính của bạn xem được tại địa chỉ 127.0.0.1)
```

### 8.2. Bắn dịch vụ ra toàn thế giới (Cho người dùng ngoài Internet truy cập qua 4G)
```powershell
# BƯỚC 1: Chuyển tiếp cổng từ Kubernetes Service ra cổng 8080 của máy Windows:
# (Mở tab PowerShell thứ nhất và giữ nguyên lệnh này chạy)
kubectl port-forward svc/quiz-backend-service 8080:80 -n quiz-app

# BƯỚC 2: Mở đường hầm (Tunnel) phát sóng cổng 8080 ra Internet miễn phí:
# (Mở tab PowerShell thứ hai và chạy lệnh này)
npx localtunnel --port 8080
```

### 8.3. Vận hành chịu tải & Cập nhật khi lên Internet (Auto-scale & Rolling Update)
```powershell
# 1. Bật tự động co giãn Pods theo CPU (HPA):
kubectl autoscale deployment quiz-backend --cpu-percent=70 --min=2 --max=15 -n quiz-app

# 2. Xem trạng thái tự động co giãn thời gian thực:
kubectl get hpa -n quiz-app

# 3. Cập nhật mã nguồn phiên bản mới không gián đoạn (Rolling Update):
kubectl set image deployment/quiz-backend quiz-backend-container=nginx:alpine -n quiz-app

# 4. Theo dõi quá trình cập nhật cuốn chiếu:
kubectl rollout status deployment/quiz-backend -n quiz-app

# 5. Khởi động lại toàn bộ Pods trong Deployment (không gây downtime):
kubectl rollout restart deployment/quiz-backend -n quiz-app
```

### 8.4. Quản lý Ingress & Tên miền ảo (`quiz-app.com`)
```powershell
# 1. Bật Addon Ingress trên Minikube:
minikube addons enable ingress

# 2. Xem danh sách Ingress và địa chỉ IP phân giải:
kubectl get ingress -n quiz-app

# 3. Xem chi tiết cấu hình và luật định tuyến của Ingress:
kubectl describe ingress quiz-ingress -n quiz-app

# 4. Cấu hình Local DNS trên Windows để nhận diện tên miền quiz-app.com:
# Mở file C:\Windows\System32\drivers\etc\hosts bằng quyền Admin và thêm dòng:
# 127.0.0.1 quiz-app.com

# 5. Mở đường hầm mạng Ingress trên Windows (Cần thiết với Docker driver):
# (Chạy trên cửa sổ PowerShell quyền Administrator)
minikube tunnel

# 6. Kiểm tra phản hồi từ tên miền quiz-app.com:
curl.exe http://quiz-app.com
```

### 8.5. Quản lý Tự động Co giãn Pods (HPA - Horizontal Pod Autoscaler)
```powershell
# 1. Kiểm tra trạng thái và mức tải CPU hiện tại so với ngưỡng đặt ra:
kubectl get hpa -n quiz-app

# 2. Xem chi tiết các sự kiện co giãn (Scaling Events) của HPA:
kubectl describe hpa quiz-backend-hpa -n quiz-app

# 3. Theo dõi liên tục biến động số lượng Pods khi có tải (Watch mode):
kubectl get pods -n quiz-app -w
```

---

## 9. Dọn Dẹp Triệt Để Docker (Containers, Images, Volumes, Cache)

### 9.1. Kiểm tra tài nguyên trước khi dọn
```powershell
# Xem tất cả containers (cả đang chạy và đã tắt)
docker ps -a

# Xem tất cả Docker images hiện có
docker images

# Xem tổng quan mức chiếm dụng ổ đĩa của Docker
docker system df
```

### 9.2. Xóa toàn bộ Containers
```powershell
# Dừng và ép xóa tất cả các container trong máy
docker rm -f $(docker ps -aq)
```

### 9.3. Xóa toàn bộ Images
```powershell
# Xóa tất cả các image hiện có trên máy (Lưu ý: Không xóa nếu Minikube đang cần kicbase)
docker rmi -f $(docker images -q)
```

### 9.4. Dọn sạch Build Cache, Networks và Volumes
```powershell
# Dọn dẹp toàn bộ build cache, container thừa, networks không dùng và anonymous volumes
docker system prune -a --volumes -f

# Xóa triệt để tất cả các Named Volumes còn sót lại
docker volume prune -a -f
```

---

## 10. Xóa Sạch Lịch Sử Builds & File Logs Hệ Thống

### 10.1. Dọn dẹp Buildx & Build Records (Lịch sử build trong Docker Desktop)
```powershell
# Xem mức chiếm dụng của buildx
docker buildx du

# Xem danh sách lịch sử các lần build
docker buildx history ls

# Xóa toàn bộ danh sách lịch sử build trong Docker Buildx
docker buildx history rm --all

# Dọn dẹp toàn bộ builder cache
docker buildx prune -a -f
```

### 10.2. Xóa triệt để file tham chiếu cache của Docker Desktop (Xóa mục Builds trên giao diện)
```powershell
# Xóa sạch các file metadata tham chiếu build cũ
Remove-Item -Path "$env:USERPROFILE\.docker\buildx\refs\default\default\*" -Force -Recurse -ErrorAction SilentlyContinue
Remove-Item -Path "$env:USERPROFILE\.docker\buildx\refs\__group__\*" -Force -Recurse -ErrorAction SilentlyContinue
Remove-Item -Path "$env:USERPROFILE\.docker\buildx\activity\default\*" -Force -Recurse -ErrorAction SilentlyContinue
```

### 10.3. Xóa các file log hệ thống cũ của Docker Desktop
```powershell
# Xóa toàn bộ file log cũ trong AppData\Local\Docker\log
Get-ChildItem -Path "$env:LOCALAPPDATA\Docker\log" -Recurse -File | Remove-Item -Force -ErrorAction SilentlyContinue

# Xóa các file log của bộ cài đặt Docker Desktop
Remove-Item -Path "$env:LOCALAPPDATA\Docker\install-log*.txt", "$env:LOCALAPPDATA\Docker\installer.error.json" -Force -ErrorAction SilentlyContinue
```
