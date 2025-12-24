# **PHASE 2: CÀI ĐẶT CÁC THÀNH PHẦN BASE**

## **LÀM TRÊN CẢ 3 VM (docker1, docker2, docker3)**

### **Bước 1: Update hệ thống**
```bash
sudo apt update && sudo apt upgrade -y
```

### **Bước 2: Cài các package cơ bản**
```bash
sudo apt install -y \
    curl \
    wget \
    git \
    vim \
    htop \
    net-tools \
    tree \
    jq \
    openssh-client \
    ca-certificates \
    gnupg \
    lsb-release
```

### **Bước 3: Cài Docker Engine (Official cách)**
```bash
# 1. Thêm Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 2. Thiết lập repository
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Cài đặt Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. Kiểm tra
docker --version
docker compose version
```

### **Bước 4: Cấu hình Docker daemon**
```bash
# Tạo file cấu hình Docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "live-restore": true
}
EOF

# Khởi động lại Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```

### **Bước 5: Thêm user vào docker group**
```bash
# Thêm user hiện tại
sudo usermod -aG docker $USER

# Áp dụng ngay không cần logout
newgrp docker << END
echo "User $USER added to docker group"
docker run --rm hello-world
END
```

### **Bước 6: Pull các Docker images cần thiết (MySQL)**
```bash
# Database - DÙNG MYSQL
docker pull mysql:8
docker pull phpmyadmin/phpmyadmin:latest  # GUI quản lý MySQL

# Backend base images
docker pull python:3.11-alpine
docker pull node:18-alpine
docker pull openjdk:17-jdk-slim
docker pull php:8.2-fpm-alpine
docker pull mcr.microsoft.com/dotnet/aspnet:7.0
docker pull mcr.microsoft.com/dotnet/sdk:7.0

# Frontend & Proxy
docker pull nginx:alpine
docker pull traefik:v2.10

# Monitoring (optional)
docker pull portainer/portainer-ce:latest
```

### **Bước 7: Cài đặt Docker Compose standalone (backup)**
```bash
# Download binary
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Phân quyền
sudo chmod +x /usr/local/bin/docker-compose

# Kiểm tra
docker-compose --version
```

### **Bước 8: Tạo cấu trúc thư mục cho lab**
```bash
mkdir -p ~/docker-lab/{frontend,backends,shared,scripts,stacks}

# Tạo thư mục cho từng backend
mkdir -p ~/docker-lab/backends/{python,nodejs,java,php,dotnet,nestjs}

# Thư mục shared - CHỈ MYSQL
mkdir -p ~/docker-lab/shared/{nginx,mysql,secrets}

# Tạo thư mục init scripts cho MySQL
mkdir -p ~/docker-lab/shared/mysql/init
```

### **Bước 9: Tạo MySQL init script**
```bash
# Tạo file init database cho app notes
cat > ~/docker-lab/shared/mysql/init/01-init.sql <<'EOF'
-- Tạo database notesdb
CREATE DATABASE IF NOT EXISTS notesdb;
USE notesdb;

-- Tạo bảng notes
CREATE TABLE IF NOT EXISTS notes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    status ENUM('pending', 'in_progress', 'done') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Tạo index
CREATE INDEX idx_status ON notes(status);

-- Insert sample data
INSERT INTO notes (title, content, status) VALUES
('Mua sữa', 'Mua sữa ở siêu thị', 'pending'),
('Làm bài tập Docker', 'Hoàn thành lab Docker Swarm', 'in_progress'),
('Họp team', 'Họp lúc 14:00', 'done');

-- Tạo user cho ứng dụng (không dùng root)
CREATE USER IF NOT EXISTS 'notesuser'@'%' IDENTIFIED BY 'notespass';
GRANT ALL PRIVILEGES ON notesdb.* TO 'notesuser'@'%';
FLUSH PRIVILEGES;
EOF

# Tạo file cấu hình MySQL
cat > ~/docker-lab/shared/mysql/my.cnf <<'EOF'
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
default-authentication-plugin=mysql_native_password
max_connections=100
wait_timeout=28800
interactive_timeout=28800

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
EOF
```

### **Bước 10: Kiểm tra images đã pull**
```bash
# Kiểm tra MySQL image đã pull
docker images | grep mysql

# Kết quả mong đợi:
# mysql      8      xxxxx   1 phút trước
```

### **Bước 11: Tạo alias và shell helpers**
```bash
# Tạo file .bash_aliases
cat >> ~/.bash_aliases <<'EOF'
# Docker aliases
alias d='docker'
alias dc='docker compose'
alias dcd='docker compose down'
alias dcu='docker compose up -d'
alias dcl='docker compose logs -f'
alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
alias dimg='docker images'
alias dvol='docker volume ls'
alias dnet='docker network ls'
alias dlog='docker logs'
alias dstop='docker stop'
alias drm='docker rm'

# Docker Swarm aliases
alias ds='docker service'
alias dsl='docker service ls'
alias dsp='docker service ps'
alias dss='docker service scale'
alias dnode='docker node ls'

# System
alias ll='ls -la'
alias lt='ls -lahtr'
alias c='clear'
EOF

# Áp dụng ngay
source ~/.bash_aliases
```

### **Bước 12: Tạo script kiểm tra**
```bash
# Tạo file check-env.sh
cat > ~/check-env.sh <<'EOF'
#!/bin/bash
echo "=== Environment Check ==="
echo "1. Hostname: $(hostname)"
echo "2. Docker: $(docker --version)"
echo "3. Docker Compose: $(docker compose version)"
echo "4. Docker Info:"
docker info 2>/dev/null | grep -E "Server Version:|Swarm:|Containers:|Images:"
echo ""
echo "5. Images pulled:"
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | head -10
echo ""
echo "6. Network connectivity:"
ping -c 1 docker1 >/dev/null 2>&1 && echo "✓ docker1 reachable" || echo "✗ docker1 unreachable"
ping -c 1 docker2 >/dev/null 2>&1 && echo "✓ docker2 reachable" || echo "✗ docker2 unreachable"
ping -c 1 docker3 >/dev/null 2>&1 && echo "✓ docker3 reachable" || echo "✗ docker3 unreachable"
EOF

chmod +x ~/check-env.sh

# Thêm kiểm tra MySQL image
cat >> ~/check-env.sh <<'EOF'

echo "7. MySQL Image check:"
docker images | grep mysql || echo "MySQL image not found"
EOF
```

### **Bước 13: Chạy test MySQL container**
```bash
# Test MySQL container chạy được không
docker run --rm -d \
  --name test-mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=testdb \
  mysql:8

# Kiểm tra
sleep 10
docker logs test-mysql | grep "ready for connections"

# Dọn dẹp
docker stop test-mysql
```

## **KIỂM TRA CUỐI CÙNG:**

### **Chạy trên cả 3 VM:**
```bash
cd ~
./check-env.sh

# Chạy test container
docker run --rm hello-world
```

### **Kết quả mong đợi:**
```
✓ Docker cài thành công
✓ Docker Compose có sẵn  
✓ Các images đã pulled
✓ Network connectivity giữa các nodes
✓ Hello-world container chạy được
```

## **Ghi chú:**
1. **Làm trên cả 3 VM** - các bước giống nhau
2. **Mất ~5-10 phút** mỗi VM
3. **Kiểm tra kết nối** giữa các VM sau khi cài xong

**Xong Phase 2!** Tiếp theo Phase 3: Thiết lập Docker Swarm trên docker1 và join các worker.
