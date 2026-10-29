# Triển khai Hiệp Sĩ Web trên Ubuntu Server bằng Docker

Tài liệu này ghi lại quy trình cài Docker trên Ubuntu Server và deploy ứng dụng Hiệp Sĩ Web bằng Docker Compose. Mục tiêu là chạy ứng dụng trong lab VMware và truy cập từ máy Windows cùng mạng có thể kết nối tới Ubuntu VM.

> **Lưu ý:** Ứng dụng được publish qua cổng `8081` trên Ubuntu host, chuyển tiếp vào cổng `8080` trong container. Không mở cổng ứng dụng ra Internet nếu chỉ dùng trong lab.

## 1. Mô hình triển khai

```text
Windows (trình duyệt)
        |
        | HTTP :8081
        v
Ubuntu Server VM
        |
        | Docker port mapping 8081:8080
        v
Container hiepsi-web (:8080)
        |
        v
Volume dữ liệu: ./data -> /data
```

- Ubuntu Server chạy trên VMware Workstation.
- Docker Engine và Docker Compose plugin quản lý container.
- Ứng dụng Hiệp Sĩ Web lắng nghe cổng `8080` bên trong container.
- Máy Ubuntu publish ứng dụng qua cổng `8081`.
- Thư mục `./data` lưu dữ liệu bền vững trên host.

## 2. Cập nhật Ubuntu

Đăng nhập Ubuntu bằng tài khoản có quyền `sudo`:

```bash
sudo apt update
sudo apt upgrade -y
```

## 3. Cài Docker Engine và Docker Compose

### 3.1. Gỡ các gói Docker cũ (nếu có)

Lệnh này chỉ gỡ các gói cũ, không xóa dữ liệu trong thư mục dự án:

```bash
sudo apt remove -y docker.io docker-doc docker-compose podman-docker containerd runc
```

### 3.2. Thêm repository chính thức của Docker

```bash
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Tạo file repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 3.3. Cài Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Khởi động Docker và bật tự chạy khi máy khởi động:

```bash
sudo systemctl enable --now docker
```

### 3.4. Kiểm tra

```bash
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

Nếu lệnh `hello-world` hiển thị thông báo thành công, Docker Engine đã hoạt động.

### 3.5. (Tùy chọn) Dùng Docker không cần `sudo`

```bash
sudo usermod -aG docker "$USER"
```

Đăng xuất rồi đăng nhập lại để nhóm có hiệu lực. Sau đó kiểm tra:

```bash
docker ps
```

> Thành viên nhóm `docker` có quyền rất cao trên máy chủ, gần tương đương quyền root. Chỉ thêm tài khoản đáng tin cậy vào nhóm này.

## 4. Đưa source code lên Ubuntu

Ví dụ dự án được đặt tại:

```text
/home/ubuntu/standee
```

Nếu repository đã có trên GitHub:

```bash
sudo apt install -y git
git clone <GITHUB_REPOSITORY_URL> standee
cd standee
```

Nếu source code đã có trên Ubuntu, chuyển đến thư mục dự án:

```bash
cd /home/ubuntu/standee
```

Kiểm tra cấu trúc dự án:

```bash
ls -la
```

Dự án cần có Dockerfile và file Compose (ví dụ `compose.yaml`) để build và chạy ứng dụng.

## 5. Cấu hình Docker Compose

Tạo hoặc chỉnh sửa file `compose.yaml` trong thư mục dự án:

```yaml
services:
  hiepsi-web:
    build: .
    container_name: hiepsi-web
    restart: unless-stopped
    ports:
      - "8081:8080"
    environment:
      PORT: "8080"
      DATA_DIR: /data
      ADMIN_PASSWORD: "${ADMIN_PASSWORD:?Set ADMIN_PASSWORD in .env}"
    volumes:
      - ./data:/data
```

### Giải thích cấu hình

| Cấu hình | Ý nghĩa |
|---|---|
| `build: .` | Build image từ Dockerfile trong thư mục hiện tại |
| `container_name` | Đặt tên container là `hiepsi-web` |
| `restart: unless-stopped` | Tự khởi động lại container, trừ khi đã dừng thủ công |
| `8081:8080` | Cổng `8081` trên Ubuntu host chuyển vào cổng `8080` trong container |
| `PORT: "8080"` | Ứng dụng lắng nghe cổng 8080 bên trong container |
| `DATA_DIR: /data` | Đường dẫn dữ liệu bên trong container |
| `./data:/data` | Lưu dữ liệu container vào thư mục `data` trên Ubuntu |

**Quan trọng:** Không đổi `PORT` trong container thành `8081` chỉ vì muốn truy cập bằng cổng 8081. Với cấu hình này, ứng dụng vẫn chạy ở `8080`; `8081` chỉ là cổng publish trên host.

## 6. Tạo file biến môi trường `.env`

Trong thư mục dự án:

```bash
nano .env
```

Nội dung mẫu:

```dotenv
ADMIN_PASSWORD=ThayBangMatKhauManhVaRieng
```

Lưu file, sau đó tạo thư mục dữ liệu:

```bash
mkdir -p data
```

Bảo vệ file `.env`:

```bash
chmod 600 .env
```

Không commit file `.env` lên GitHub vì file có thể chứa mật khẩu. Thêm các mục sau vào `.gitignore`:

```gitignore
.env
data/
```

Nếu mật khẩu thật đã từng được commit hoặc chia sẻ, hãy thay mật khẩu đó.

## 7. Kiểm tra cấu hình và deploy

Kiểm tra Compose có hợp lệ:

```bash
sudo docker compose config
```

Build image và chạy container:

```bash
sudo docker compose up -d --build
```

Kiểm tra trạng thái:

```bash
sudo docker compose ps
```

Xem log ứng dụng:

```bash
sudo docker compose logs -f hiepsi-web
```

Thoát xem log bằng `Ctrl+C`; thao tác này không dừng container.

Kiểm tra ứng dụng từ Ubuntu host:

```bash
curl -I http://localhost:8081
```

Nếu ứng dụng không trả lời, xem log và trạng thái container trước khi thay đổi cấu hình.

## 8. Truy cập từ Windows

### 8.1. Lấy địa chỉ IP Ubuntu

```bash
ip -br addr
ip route
```

Tìm địa chỉ IPv4 của card mạng đang sử dụng, ví dụ `192.168.1.50`.

### 8.2. Mở ứng dụng

Trên Windows, mở trình duyệt và truy cập:

```text
http://<UBUNTU_IP>:8081
```

Ví dụ:

```text
http://192.168.1.50:8081
```

Khả năng truy cập phụ thuộc vào chế độ mạng VMware:

- **Bridged:** VM thường nhận IP cùng mạng với Wi-Fi/LAN; Windows có thể truy cập trực tiếp nếu mạng cho phép các thiết bị liên lạc.
- **NAT:** VM đi Internet qua host. Windows host thường có thể truy cập VM qua IP NAT của VMware; máy Windows khác trên mạng thường không thể truy cập trực tiếp nếu chưa cấu hình chuyển tiếp.
- **Host-only:** chỉ các máy trong mạng host-only liên lạc với nhau; VM không tự có Internet qua mạng này.

Nếu Windows và Ubuntu không liên lạc được, kiểm tra VMware Network Adapter và firewall trước. Không cần mở cổng trên router nếu chỉ test từ Windows host hoặc trong mạng lab.

## 9. Xử lý lỗi thường gặp

### 9.1. Cổng 8080 đã được sử dụng

Kiểm tra cổng:

```bash
sudo ss -lntup | grep -E ':8080|:8081'
sudo docker ps
```

Nếu một container khác đã chiếm cổng host `8080`, dùng port mapping riêng cho Hiệp Sĩ, ví dụ:

```yaml
ports:
  - "8081:8080"
```

Sau đó áp dụng lại:

```bash
sudo docker compose up -d
```

### 9.2. Container không chạy

```bash
sudo docker compose ps -a
sudo docker compose logs --tail=100 hiepsi-web
```

Kiểm tra lại file `.env`, Dockerfile, port mapping và các thông báo lỗi trong log.

### 9.3. Không truy cập được từ Windows

1. Kiểm tra container có đang chạy:

   ```bash
   sudo docker compose ps
   ```

2. Kiểm tra ứng dụng từ Ubuntu:

   ```bash
   curl -I http://localhost:8081
   ```

3. Kiểm tra IP Ubuntu:

   ```bash
   ip -br addr
   ```

4. Kiểm tra firewall:

   ```bash
   sudo ufw status
   ```

5. Kiểm tra chế độ mạng VMware (NAT/Bridged/Host-only) và khả năng kết nối giữa Windows với Ubuntu.

### 9.4. Xem log hoặc khởi động lại

```bash
sudo docker compose logs -f hiepsi-web
sudo docker compose restart hiepsi-web
```

## 10. Các lệnh vận hành nhanh

Chạy trong thư mục có `compose.yaml`:

```bash
# Khởi động
sudo docker compose up -d

# Build lại và khởi động
sudo docker compose up -d --build

# Dừng và xóa container/network do Compose tạo
sudo docker compose down

# Xem trạng thái
sudo docker compose ps

# Xem log
sudo docker compose logs -f hiepsi-web

# Khởi động lại ứng dụng
sudo docker compose restart hiepsi-web
```

`docker compose down` không xóa thư mục `./data`. Không dùng `docker compose down -v` nếu chưa hiểu rõ ảnh hưởng đến dữ liệu volume.

## 11. Ghi chú bảo mật

- Không đưa `.env`, mật khẩu quản trị hoặc dữ liệu thực lên GitHub.
- Không public port `8081` ra Internet khi ứng dụng chỉ dành cho lab.
- Chỉ cho phép truy cập từ các máy/mạng cần thiết.
- Nếu sau này dùng NetBird, có thể giới hạn đường truy cập Hiệp Sĩ qua VPN; NetBird Management và ứng dụng Hiệp Sĩ là hai thành phần khác nhau.
- Cập nhật Ubuntu, Docker image và ứng dụng định kỳ; sao lưu thư mục `data` trước khi nâng cấp.

---

## Checklist triển khai

- [ ] Ubuntu đã cập nhật
- [ ] Docker Engine và Docker Compose plugin đã cài
- [ ] `docker run --rm hello-world` chạy thành công
- [ ] Source code và Dockerfile có trên Ubuntu
- [ ] `compose.yaml` cấu hình đúng port `8081:8080`
- [ ] File `.env` đã tạo và không được commit
- [ ] `docker compose up -d --build` chạy thành công
- [ ] `curl -I http://localhost:8081` có phản hồi
- [ ] Windows truy cập được `http://<UBUNTU_IP>:8081`
