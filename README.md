# DevOps Hackathon - Đề 005 : Quản lý nhân sự (HRM)

## 1. Thông tin sinh viên
- **Họ tên:** Nguyễn Thế Kiên
- **Lớp:** KS24 - CNTT03
- **Đề:** 005 - Hệ thống Quản lý nhân sự (HRM)

## 2. Môi trường triển khai
- **VPS:** Ubuntu 22.04.5 LTS
- **Địa chỉ IP:** 160.187.229.73
- **Hostname:** kien-d24
- **Web server:** Nginx 1.18.0 (reverse proxy)
- **Backend:** Spring Boot (Java), chạy nội bộ tại `127.0.0.1:8082`
- **Truy cập:** SSH cổng 22, HTTP cổng 80

## 3. Cấu trúc dự án
```
HN_KS24_CNTT03_NguyenTheKienDe005/
├── nginx/
│   └── spring-proxy.conf      
├── src/
│   └── index.html            
├── screenshot/
│   ├── 01-User.png            
│   └── 3.3 Tường lửa.png      
└── README.md
```

## 4. Cấu hình Nginx
File `nginx/spring-proxy.conf` — `/` phục vụ trang tĩnh, `/api/` reverse proxy sang Spring Boot cổng 8082, có ghi log riêng.

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/html;
    index index.html;

    access_log /var/log/nginx/kien-d24.access.log;
    error_log  /var/log/nginx/kien-d24.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8082/;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
- Kích hoạt bằng symlink sang `sites-enabled/`, gỡ site `default` để tránh trùng `server_name _` cổng 80.
- `proxy_set_header` chuyển tiếp Host và IP thật của client cho backend.

## 5. Tường lửa UFW
Chặn mặc định lưu lượng vào, cho ra ngoài; mở SSH + HTTP:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 80/tcp        # HTTP
sudo ufw --force enable
sudo ufw status verbose
```
Kết quả: UFW **Active**, 22 và 80 ở trạng thái ALLOW. (Minh chứng: `screenshot/3.3 Tường lửa.png`)

## 6. Các bước triển khai
```bash
ssh root@160.187.229.73

sudo hostnamectl set-hostname kien-d24
echo "127.0.1.1 kien-d24" | sudo tee -a /etc/hosts

sudo apt update
sudo apt install -y nginx openjdk-17-jre-headless

sudo cp src/index.html /var/www/html/index.html

nohup java -jar ten-hrm.jar --server.port=8082 > ~/hrm.log 2>&1 &
sudo ss -tunlp | grep 8082        # xác nhận đang nghe cổng 8082

sudo cp nginx/spring-proxy.conf /etc/nginx/sites-available/spring-proxy.conf
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx

```

## 7. Kiểm tra & minh chứng
```bash
sudo nginx -t                          
sudo ss -tunlp | grep 8082             
curl -I http://localhost/              
curl -i http://localhost/api/<endpoint>
sudo ufw status verbose                 
```
Truy cập ngoài: `http://160.187.229.73/`. Minh chứng: `screenshot/01-User.png`, `screenshot/3.3 Tường lửa.png`.

## 8. Quy trình cập nhật website
```bash

git pull origin main


sudo cp src/index.html /var/www/html/index.html


kill -15 <PID_cũ>                       
nohup java -jar ten-hrm.jar --server.port=8082 > ~/hrm.log 2>&1 &


sudo nginx -t && sudo systemctl reload nginx
```

## 9. Sự cố gặp phải và cách khắc phục
