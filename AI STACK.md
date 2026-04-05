# 🖥️ TÀI LIỆU HƯỚNG DẪN CÀI ĐẶT & VẬN HÀNH AI STACK (GEMMA 4 + RAG)

## 1. Thành phần hệ thống
- LLM Engine: Ollama (Chạy trực tiếp trên Host WSL để tận dụng GPU RTX 4060).

- Interface: Open WebUI (Chạy trong K8s).

- Vector DB: Qdrant (Chạy trong K8s - Lưu trữ tri thức riêng).

- Orchestration: Kubernetes (Kind) trên WSL2.
## 2. Cấu hình Ollama (Host WSL)

Để các Pod trong K8s có thể kết nối với GPU, Ollama phải được cấu hình để lắng nghe trên tất cả các card mạng.

**Bước 1**: Thiết lập biến môi trường

Mở terminal WSL và chạy:
```bash
export OLLAMA_HOST=0.0.0.0:11434
export OLLAMA_ORIGINS="*"
ollama serve &
```

- Lưu ý: Nếu báo lỗi address already in use, hãy chạy sudo pkill ollama trước.

Bước 2: Tải Model Gemma 4

```bash
ollama pull gemma4:e4b
```

## 3. Cấu hình Kubernetes (AI-Stack)

Khi Deploy file ai-stack.yaml, cần lưu ý đặc biệt về kết nối Networking giữa Pod và Host.
Fix lỗi "Failed to resolve host.docker.internal"

Sử dụng hostAliases trong file YAML để ánh xạ IP của máy WSL vào Pod:

```yaml
spec:
  template:
    spec:
      hostAliases:
      - ip: "192.168.192.72" # IP WSL lấy từ lệnh 'hostname -I'
        hostnames:
        - "host.docker.internal"
      containers:
      - name: open-webui
        env:
        - name: OLLAMA_BASE_URL
          value: "http://host.docker.internal:11434"
```

## 4. Quy trình khởi động lại hệ thống (Sau khi Reboot máy)

Nếu máy tính khởi động lại, bạn thực hiện theo thứ tự sau:

### 1. Bật Ollama (Host):

```bash
ollama pull gemma4:e4b
```

### 2. Kiểm tra cụm Kind:

```bash
kind get clusters
# Nếu cụm chưa chạy, docker sẽ tự start nó hoặc chạy: docker start kind-control-plane
```

### 3. Bật Port-Forward (Duy trì WebUI):
Để không bị ngắt khi tắt Terminal, dùng nohup:

```bash
nohup kubectl port-forward svc/open-webui-service 3000:80 --address 0.0.0.0 > pf.log 2>&1 &
```

5. Hướng dẫn kiểm tra Log & Troubleshooting

Là một SysAdmin, bạn cần nắm lòng các lệnh kiểm tra trạng thái "sức khỏe" của ứng dụng:

**Kiểm tra Log của Open WebUI**

Khi gặp lỗi "Server Connection Error" hoặc lỗi khi chat:

```bash
# Tìm tên Pod
kubectl get pods
# Xem log (thêm -f để xem real-time)
kubectl logs -f <tên-pod-open-webui>
```
**Kiểm tra Log của Qdrant (Database)**

Nếu việc upload tài liệu (RAG) bị lỗi:

```bash
kubectl logs -f deployment/qdrant
```

**Kiểm tra kết nối từ Pod tới Ollama**

Để test xem "mạch máu" networking có thông không:


```bash
kubectl exec -it deployment/open-webui -- curl -I http://host.docker.internal:11434/api/tags
```

- **Kết quả 200 OK**: Kết nối tốt.

- **Kết quả Connection Refused**: Kiểm tra lại OLLAMA_HOST trên máy host.

## 6. Các lưu ý quan trọng khi vận hành

- **VRAM Management**: Card RTX 4060 có 8GB VRAM. Khi dùng Gemma 4 (E4B), hệ thống sẽ chiếm khoảng 3-4GB. Tránh chạy song song các ứng dụng nặng hoặc game khi đang dùng AI để tránh lỗi CUDA OOM (Out of Memory).

- **IP Changes**: Nếu máy WSL đổi IP, bạn phải cập nhật lại hostAliases trong file ai-stack.yaml và kubectl apply -f ai-stack.yaml.

- **Dữ liệu RAG**: Các tài liệu bạn upload lên Open WebUI sẽ được lưu vào Persistent Volume của Qdrant. Đừng xóa PVC nếu bạn không muốn mất dữ liệu tri thức đã nạp.
