# Gemma 4 + Open WebUI + Qdrant + Ubuntu WSL

## 1. Kiến trúc hệ thống

Hệ thống được thiết kế để chạy tối ưu trên RTX 4060 8GB:

- **Ollama**: Chạy trực tiếp trên WSL để truy cập GPU nhanh nhất.

- **Qdrant**: Vector Database chạy trong K8s để lưu trữ các đoạn văn bản đã được vector hóa (Embeddings).

- **Open WebUI**: Giao diện người dùng, kết nối Ollama và Qdrant để thực hiện tra cứu tài liệu.

### Bước 1: Cài đặt và cấu hình WSL với GPU Nvidia

Mở PowerShell trên Windows với quyền Admin và thực hiện:

#### 1. Cập nhật WSL và cài đặt Ubuntu:

```powershell
wsl --update
wsl --install Ubuntu
```
#### 2. Kiểm tra nhận diện Card đồ họa:
Vào môi trường Ubuntu (gõ wsl), sau đó chạy lệnh:

```powershell
nvidia-smi
```
### Bước 2: Cài đặt Ollama (Backend chạy Gemma 4)

Ollama sẽ đóng vai trò là "động cơ" để chạy mô hình. Trong cửa sổ Ubuntu WSL, chạy lệnh sau:

Cài đặt tự động:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```
Tải và chạy Gemma 4 (Bản E4B tối ưu cho 8GB VRAM):

```bash
ollama run gemma4:e4b
```
Đợi máy tải khoảng 3GB dữ liệu. Sau khi tải xong, bạn có thể chat thử ngay trong terminal.

### Bước 3. Chuẩn bị Backend (Ollama)

trên môi trường Ubuntu WSL, thiết lập Ollama lắng nghe các kết nối từ bên trong Cluster:

```bash
# Thiết lập biến môi trường để Ollama chấp nhận kết nối từ Docker/K8s
export OLLAMA_HOST=0.0.0.0
export OLLAMA_ORIGINS="*"

# Khởi chạy Ollama service
ollama serve &

# Tải mô hình Gemma 4 bản 4 tỷ tham số
ollama pull gemma4:e4b
```

### Bước 4. Triển khai Vector Database (Qdrant) & Open WebUI

Lưu nội dung dưới đây thành file ai-stack.yaml. Tài liệu này bao gồm **Storage**, **Deployment** và **Service**.

```yaml
---
# 1. STORAGE CHO VECTOR DB VÀ WEBUI
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: qdrant-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: open-webui-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 2Gi
---
# 2. TRIỂN KHAI QDRANT (VECTOR DATABASE)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qdrant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
      - name: qdrant
        image: qdrant/qdrant:latest
        ports:
        - containerPort: 6333
        volumeMounts:
        - name: qdrant-data
          mountPath: /qdrant/storage
      volumes:
      - name: qdrant-data
        persistentVolumeClaim:
          claimName: qdrant-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: qdrant-service
spec:
  selector:
    app: qdrant
  ports:
    - protocol: TCP
      port: 6333
      targetPort: 6333
---
# 3. TRIỂN KHAI OPEN WEBUI
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-webui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: open-webui
  template:
    metadata:
      labels:
        app: open-webui
    spec:
      containers:
      - name: open-webui
        image: ghcr.io/open-webui/open-webui:main
        ports:
        - containerPort: 8080
        env:
        # Kết nối tới Ollama trên Host WSL
        - name: OLLAMA_BASE_URL
          value: "http://host.docker.internal:11434"
        # Kết nối tới Qdrant Service trong cụm
        - name: VECTOR_DB
          value: "qdrant"
        - name: QDRANT_HOST
          value: "qdrant-service"
        volumeMounts:
        - name: webui-data
          mountPath: /app/backend/data
      volumes:
      - name: webui-data
        persistentVolumeClaim:
          claimName: open-webui-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: open-webui-service
spec:
  type: NodePort
  selector:
    app: open-webui
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080
```

**Lệnh thực thi & Truy cập**

Triển khai lên Kind:

```bash
kubectl apply -f ai-stack.yaml

Kiểm tra trạng thái Pod:
kubectl get pods -w

Kiểm tra danh sách Cluster hiện có
Bạn cần xác định tên cụm Kind mà bạn đã tạo:
kind get clusters

Lệnh này sẽ lấy cấu hình kết nối đúng từ cụm Kind và ghi vào file ~/.kube/config:
kind export kubeconfig --name kind

Kiểm tra lại kết nối
kubectl cluster-info

Port-forward để truy cập từ Windows:
nohup kubectl port-forward svc/open-webui-service 3000:80 --address 0.0.0.0 > pf.log 2>&1 &
```
