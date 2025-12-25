# **PHASE 1: CÀI ĐẶT CÁC THÀNH PHẦN BASE**

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
docker compose --version
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

**Xong Phase 1!** Tiếp theo Phase 2: Thiết lập Docker Swarm trên docker1 và join các worker.

---

# **PHASE 2: THIẾT LẬP DOCKER SWARM**

## **BƯỚC 1: TRÊN DOCKER1 (MANAGER)**

### **1.1 Khởi tạo Docker Swarm**
```bash
# SSH vào docker1
ssh dockeruser@192.168.230.190

# Khởi tạo Swarm với advertise IP
docker swarm init --advertise-addr 192.168.230.190

# Kết quả sẽ hiện:
# Swarm initialized: current node (xxx) is now a manager.
# 
# To add a worker to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-xxx 192.168.230.190:2377
# 
# To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

# LƯU LẠI TOKEN và IP:PORT
```

### **1.2 Kiểm tra trạng thái Swarm**
```bash
# Kiểm tra node
docker node ls

# Kết quả mong đợi:
# ID                        HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
# xxx *      docker1     Ready     Active         Leader           xx.xx.x
```

### **1.3 Lấy join token cho worker (nếu cần)**
```bash
# Lấy token join cho worker
docker swarm join-token worker

# Lưu token để dùng cho docker2 và docker3
```

## **BƯỚC 2: TRÊN DOCKER2 (WORKER 1)**

### **2.1 Join Swarm cluster**
```bash
# SSH vào docker2
ssh dockeruser@192.168.230.191

# Join Swarm (dùng token từ bước trên)
docker swarm join --token SWMTKN-1-xxx 192.168.230.190:2377

# Kết quả:
# This node joined a swarm as a worker.
```

## **BƯỚC 3: TRÊN DOCKER3 (WORKER 2)**

### **3.1 Join Swarm cluster**
```bash
# SSH vào docker3
ssh dockeruser@192.168.230.199

# Join Swarm
docker swarm join --token SWMTKN-1-xxx 192.168.230.190:2377

# Kết quả:
# This node joined a swarm as a worker.
```

## **BƯỚC 4: QUAY LẠI DOCKER1 KIỂM TRA**

### **4.1 Kiểm tra tất cả nodes**
```bash
# Trên docker1
docker node ls

# Kết quả mong đợi:
# ID                        HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
# xxx *      docker1     Ready     Active         Leader           xx.xx.x
# yyy        docker2     Ready     Active                           xx.xx.x
# zzz        docker3     Ready     Active                           xx.xx.x
```

### **4.2 Kiểm tra chi tiết hơn**
```bash
# Xem thông tin Swarm
docker info | grep -A 10 "Swarm"

# Kiểm tra node chi tiết
docker node inspect docker1 --pretty
docker node inspect docker2 --pretty
```

## **BƯỚC 5: TẠO OVERLAY NETWORK (cho các service giao tiếp)**

### **5.1 Tạo network trên manager**
```bash
# Tạo overlay network cho ứng dụng
docker network create -d overlay --attachable app-network

# Kiểm tra
docker network ls

# Kết quả nên có:
# NETWORK ID     NAME              DRIVER    SCOPE
# xxx          app-network        overlay   swarm
```

## **BƯỚC 6: KIỂM TRA SWARM HOẠT ĐỘNG**

### **6.1 Deploy test service**
```bash
# Tạo service test
docker service create \
  --name test-service \
  --replicas 3 \
  --network app-network \
  alpine:latest sleep 3600

# Kiểm tra service
docker service ls

# Kiểm tra tasks phân bổ
docker service ps test-service

# Kết quả: 3 tasks trên 3 nodes
```

### **6.2 Kiểm tra network connectivity**
```bash
# Scale service
docker service scale test-service=5

# Kiểm tra lại phân bổ
docker service ps test-service

# Xóa test service
docker service rm test-service
```

## **BƯỚC 7: TẠO DOCKER VOLUME CHO MYSQL**

### **7.1 Tạo volume trên manager**
```bash
# Tạo volume cho MySQL data
docker volume create mysql_data

# Kiểm tra volume (sẽ sync trên các node khi cần)
docker volume ls
```

## **BƯỚC 8: KIỂM TRA TỔNG QUAN**

### **8.1 Tạo script kiểm tra Swarm**
```bash
cat > ~/check-swarm.sh <<'EOF'
#!/bin/bash
echo "=== DOCKER SWARM STATUS ==="
echo ""
echo "1. Node Status:"
docker node ls
echo ""
echo "2. Networks:"
docker network ls | grep -E "(overlay|bridge)"
echo ""
echo "3. Services (if any):"
docker service ls
echo ""
echo "4. Ping test between nodes:"
echo "From $(hostname):"
for node in docker1 docker2 docker3; do
  if [ "$node" != "$(hostname)" ]; then
    ping -c 1 -W 1 $node >/dev/null 2>&1
    if [ $? -eq 0 ]; then
      echo "  ✓ $node reachable"
    else
      echo "  ✗ $node unreachable"
    fi
  fi
done
EOF

chmod +x ~/check-swarm.sh

# Chạy trên docker1
./check-swarm.sh
```

## **BƯỚC 9: FIX LỖI NẾU CÓ**

### **Nếu node không join được:**
```bash
# Trên node có vấn đề:
docker swarm leave --force

# Trên manager, tạo token mới:
docker swarm join-token --rotate worker

# Join lại với token mới
```

### **Nếu network không tạo được:**
```bash
# Kiểm tra firewall
sudo ufw status
# Nếu active, mở port:
sudo ufw allow 2377/tcp
sudo ufw allow 7946/tcp
sudo ufw allow 7946/udp
sudo ufw allow 4789/udp
```

## **KIỂM TRA CUỐI CÙNG:**

### **Trên docker1 chạy:**
```bash
echo "=== FINAL SWARM CHECK ==="
echo "Manager IP: 192.168.230.190"
echo "Worker1 IP: 192.168.230.191"
echo "Worker2 IP: 192.168.230.199"
echo ""
docker node ls
```

**Kết quả thành công khi:**
- ✅ `docker node ls` hiện 3 nodes
- ✅ docker1 là Leader
- ✅ docker2, docker3 là workers
- ✅ `app-network` tạo thành công

**Xong Phase 2!** Tiếp theo Phase 3: Chuẩn bị source code cho lab.

---

# **PHASE 3: CHUẨN BỊ SOURCE CODE CHO LAB**



## **BƯỚC 1: TRÊN DOCKER1 (MANAGER)**



### **1.1 Tạo cấu trúc thư mục đầy đủ**

```bash

# Đảm bảo đang ở home directory

cd ~



# Tạo cấu trúc thư mục chính

mkdir -p docker-lab/{frontend,backends,shared,scripts,stacks}



# Tạo thư mục cho từng backend stack

mkdir -p docker-lab/backends/{python,nodejs,java,php,dotnet,nestjs}



# Thư mục shared resources

mkdir -p docker-lab/shared/{nginx,mysql,secrets}



# Tạo init scripts cho MySQL

mkdir -p docker-lab/shared/mysql/init

```



### **1.2 Tạo React Frontend cơ bản**

```bash

# Tạo package.json cho React

cat > ~/docker-lab/frontend/package.json <<'EOF'

{

  "name": "notes-app-frontend",

  "version": "1.0.0",

  "private": true,

  "dependencies": {

    "react": "^18.2.0",

    "react-dom": "^18.2.0",

    "react-scripts": "5.0.1",

    "axios": "^1.5.0"

  },

  "scripts": {

    "start": "react-scripts start",

    "build": "react-scripts build",

    "test": "react-scripts test",

    "eject": "react-scripts eject"

  },

  "proxy": "http://backend:5000"

}

EOF



# Tạo Dockerfile cho React

cat > ~/docker-lab/frontend/Dockerfile <<'EOF'

# Build stage

FROM node:18-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm ci --only=production

COPY . .

RUN npm run build



# Production stage

FROM nginx:alpine

COPY --from=build /app/build /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

EOF



# Tạo nginx config

cat > ~/docker-lab/frontend/nginx.conf <<'EOF'

server {

    listen 80;

    server_name localhost;

    

    location / {

        root /usr/share/nginx/html;

        index index.html index.htm;

        try_files $uri $uri/ /index.html;

    }

    

    # Proxy API requests to backend

    location /api {

        rewrite ^/api/(.*) /$1 break;

        proxy_pass http://backend:5000;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;

    }

}

EOF



# Tạo React app đơn giản

cat > ~/docker-lab/frontend/public/index.html <<'EOF'

<!DOCTYPE html>

<html lang="en">

<head>

    <meta charset="utf-8" />

    <meta name="viewport" content="width=device-width, initial-scale=1" />

    <title>Notes App</title>

</head>

<body>

    <div id="root"></div>

</body>

</html>

EOF



# Tạo React component cơ bản

mkdir -p ~/docker-lab/frontend/src

cat > ~/docker-lab/frontend/src/App.js <<'EOF'

import React, { useState, useEffect } from 'react';

import axios from 'axios';



function App() {

  const [notes, setNotes] = useState([]);

  const [newNote, setNewNote] = useState({ title: '', content: '' });



  useEffect(() => {

    fetchNotes();

  }, []);



  const fetchNotes = async () => {

    try {

      const response = await axios.get('/api/notes');

      setNotes(response.data);

    } catch (error) {

      console.error('Error fetching notes:', error);

    }

  };



  const addNote = async () => {

    try {

      await axios.post('/api/notes', newNote);

      setNewNote({ title: '', content: '' });

      fetchNotes();

    } catch (error) {

      console.error('Error adding note:', error);

    }

  };



  const deleteNote = async (id) => {

    try {

      await axios.delete(`/api/notes/${id}`);

      fetchNotes();

    } catch (error) {

      console.error('Error deleting note:', error);

    }

  };



  return (

    <div style={{ padding: '20px' }}>

      <h1>Notes App</h1>

      

      <div>

        <h2>Add New Note</h2>

        <input

          type="text"

          placeholder="Title"

          value={newNote.title}

          onChange={(e) => setNewNote({...newNote, title: e.target.value})}

        />

        <input

          type="text"

          placeholder="Content"

          value={newNote.content}

          onChange={(e) => setNewNote({...newNote, content: e.target.value})}

        />

        <button onClick={addNote}>Add Note</button>

      </div>



      <div>

        <h2>Notes List</h2>

        {notes.length === 0 ? (

          <p>No notes yet. Add one above!</p>

        ) : (

          <ul>

            {notes.map(note => (

              <li key={note.id} style={{ marginBottom: '10px', border: '1px solid #ccc', padding: '10px' }}>

                <h3>{note.title}</h3>

                <p>{note.content}</p>

                <p>Status: {note.status}</p>

                <button onClick={() => deleteNote(note.id)}>Delete</button>

              </li>

            ))}

          </ul>

        )}

      </div>

    </div>

  );

}



export default App;

EOF



cat > ~/docker-lab/frontend/src/index.js <<'EOF'

import React from 'react';

import ReactDOM from 'react-dom/client';

import App from './App';



const root = ReactDOM.createRoot(document.getElementById('root'));

root.render(

  <React.StrictMode>

    <App />

  </React.StrictMode>

);

EOF

```



### **1.3 Tạo MySQL Init Scripts**

```bash

# Tạo database init script

cat > ~/docker-lab/shared/mysql/init/01-init.sql <<'EOF'

-- Create database

CREATE DATABASE IF NOT EXISTS notesdb

CHARACTER SET utf8mb4

COLLATE utf8mb4_unicode_ci;



-- Create app user

CREATE USER IF NOT EXISTS 'notes_app'@'%' IDENTIFIED BY 'AppPassword123!';

GRANT ALL PRIVILEGES ON notesdb.* TO 'notes_app'@'%';

FLUSH PRIVILEGES;



-- Use database

USE notesdb;



-- Create notes table

CREATE TABLE IF NOT EXISTS notes (

    id INT AUTO_INCREMENT PRIMARY KEY,

    title VARCHAR(255) NOT NULL,

    content TEXT,

    status ENUM('pending', 'in_progress', 'done') DEFAULT 'pending',

    is_completed BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_status (status),

    INDEX idx_created (created_at)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;



-- Insert sample data

INSERT INTO notes (title, content, status, is_completed) VALUES

('Buy groceries', 'Milk, Eggs, Bread, Vegetables', 'pending', FALSE),

('Complete Docker Lab', 'Deploy 6 backend stacks with React FE', 'in_progress', FALSE),

('Submit weekly report', 'Prepare slides and Q4 data', 'done', TRUE),

('Meeting with DevOps team', 'Discuss CI/CD pipeline', 'in_progress', FALSE),

('Fix login page bug', 'Email validation not working', 'pending', FALSE),

('Deploy to production', 'Deploy to main server', 'done', TRUE);



-- Update completed status

UPDATE notes SET is_completed = TRUE WHERE status = 'done';

EOF

```



## **BƯỚC 2: TẠO BACKEND PYTHON (LAB ĐẦU TIÊN)**



### **2.1 Tạo Python Backend cơ bản**

```bash

# Tạo requirements.txt

cat > ~/docker-lab/backends/python/requirements.txt <<'EOF'

Flask==2.3.3

Flask-CORS==4.0.0

mysql-connector-python==8.1.0

python-dotenv==1.0.0

EOF



# Tạo Dockerfile cho Python

cat > ~/docker-lab/backends/python/Dockerfile <<'EOF'

FROM python:3.11-alpine

WORKDIR /app



# Install system dependencies for mysql-connector

RUN apk add --no-cache gcc musl-dev mariadb-connector-c-dev



COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt



COPY . .



EXPOSE 5000

CMD ["python", "app.py"]

EOF



# Tạo Python app (Flask)

cat > ~/docker-lab/backends/python/app.py <<'EOF'

from flask import Flask, jsonify, request

from flask_cors import CORS

import mysql.connector

import os

from datetime import datetime



app = Flask(__name__)

CORS(app)



# Database configuration

DB_CONFIG = {

    'host': os.getenv('DB_HOST', 'mysql'),

    'user': os.getenv('DB_USER', 'notes_app'),

    'password': os.getenv('DB_PASSWORD', 'AppPassword123!'),

    'database': os.getenv('DB_NAME', 'notesdb'),

    'port': os.getenv('DB_PORT', 3306)

}



def get_db_connection():

    try:

        conn = mysql.connector.connect(**DB_CONFIG)

        return conn

    except mysql.connector.Error as err:

        print(f"Database connection error: {err}")

        return None



@app.route('/api/health', methods=['GET'])

def health_check():

    return jsonify({'status': 'healthy', 'service': 'Python Backend'})



@app.route('/api/notes', methods=['GET'])

def get_notes():

    conn = get_db_connection()

    if not conn:

        return jsonify({'error': 'Database connection failed'}), 500

    

    cursor = conn.cursor(dictionary=True)

    cursor.execute('SELECT * FROM notes ORDER BY created_at DESC')

    notes = cursor.fetchall()

    

    cursor.close()

    conn.close()

    

    return jsonify(notes)



@app.route('/api/notes/<int:note_id>', methods=['GET'])

def get_note(note_id):

    conn = get_db_connection()

    if not conn:

        return jsonify({'error': 'Database connection failed'}), 500

    

    cursor = conn.cursor(dictionary=True)

    cursor.execute('SELECT * FROM notes WHERE id = %s', (note_id,))

    note = cursor.fetchone()

    

    cursor.close()

    conn.close()

    

    if note:

        return jsonify(note)

    return jsonify({'error': 'Note not found'}), 404



@app.route('/api/notes', methods=['POST'])

def create_note():

    data = request.json

    if not data or 'title' not in data:

        return jsonify({'error': 'Title is required'}), 400

    

    conn = get_db_connection()

    if not conn:

        return jsonify({'error': 'Database connection failed'}), 500

    

    cursor = conn.cursor()

    query = '''

        INSERT INTO notes (title, content, status) 

        VALUES (%s, %s, %s)

    '''

    values = (

        data.get('title'),

        data.get('content', ''),

        data.get('status', 'pending')

    )

    

    cursor.execute(query, values)

    conn.commit()

    note_id = cursor.lastrowid

    

    cursor.close()

    conn.close()

    

    return jsonify({'id': note_id, 'message': 'Note created successfully'}), 201



@app.route('/api/notes/<int:note_id>', methods=['PUT'])

def update_note(note_id):

    data = request.json

    conn = get_db_connection()

    if not conn:

        return jsonify({'error': 'Database connection failed'}), 500

    

    cursor = conn.cursor()

    

    # Build dynamic update query

    updates = []

    values = []

    

    if 'title' in data:

        updates.append('title = %s')

        values.append(data['title'])

    if 'content' in data:

        updates.append('content = %s')

        values.append(data['content'])

    if 'status' in data:

        updates.append('status = %s')

        updates.append('is_completed = %s')

        values.append(data['status'])

        values.append(data['status'] == 'done')

    

    if not updates:

        return jsonify({'error': 'No fields to update'}), 400

    

    values.append(note_id)

    query = f'UPDATE notes SET {", ".join(updates)} WHERE id = %s'

    

    cursor.execute(query, values)

    conn.commit()

    affected_rows = cursor.rowcount

    

    cursor.close()

    conn.close()

    

    if affected_rows > 0:

        return jsonify({'message': 'Note updated successfully'})

    return jsonify({'error': 'Note not found'}), 404



@app.route('/api/notes/<int:note_id>', methods=['DELETE'])

def delete_note(note_id):

    conn = get_db_connection()

    if not conn:

        return jsonify({'error': 'Database connection failed'}), 500

    

    cursor = conn.cursor()

    cursor.execute('DELETE FROM notes WHERE id = %s', (note_id,))

    conn.commit()

    affected_rows = cursor.rowcount

    

    cursor.close()

    conn.close()

    

    if affected_rows > 0:

        return jsonify({'message': 'Note deleted successfully'})

    return jsonify({'error': 'Note not found'}), 404



@app.route('/api/notes/<int:note_id>/status', methods=['PUT'])

def update_note_status(note_id):

    data = request.json

    if 'status' not in data:

        return jsonify({'error': 'Status is required'}), 400

    

    status = data['status']

    if status not in ['pending', 'in_progress', 'done']:

        return jsonify({'error': 'Invalid status'}), 400

    

    conn = get_db_connection()

    if not conn:

        return jsonify({'error': 'Database connection failed'}), 500

    

    cursor = conn.cursor()

    query = '''

        UPDATE notes 

        SET status = %s, is_completed = %s 

        WHERE id = %s

    '''

    values = (status, status == 'done', note_id)

    

    cursor.execute(query, values)

    conn.commit()

    affected_rows = cursor.rowcount

    

    cursor.close()

    conn.close()

    

    if affected_rows > 0:

        return jsonify({'message': 'Status updated successfully'})

    return jsonify({'error': 'Note not found'}), 404



if __name__ == '__main__':

    app.run(host='0.0.0.0', port=5000, debug=True)

EOF



# Tạo .env file cho Python

cat > ~/docker-lab/backends/python/.env <<'EOF'

DB_HOST=mysql

DB_PORT=3306

DB_USER=notes_app

DB_PASSWORD=AppPassword123!

DB_NAME=notesdb

FLASK_ENV=development

EOF

```



### **2.2 Tạo docker-compose.yml cho Python stack**

```bash

cat > ~/docker-lab/backends/python/docker-compose.yml <<'EOF'

version: '3.8'



services:

  # MySQL Database

  mysql:

    image: mysql:8

    container_name: notes-mysql-python

    restart: unless-stopped

    environment:

      MYSQL_ROOT_PASSWORD: RootPass123!

      MYSQL_DATABASE: notesdb

    volumes:

      - mysql_data_python:/var/lib/mysql

      - ../../shared/mysql/init:/docker-entrypoint-initdb.d

    ports:

      - "3307:3306"  # Different port for each stack

    networks:

      - notes-network

    healthcheck:

      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]

      interval: 10s

      timeout: 5s

      retries: 5



  # Python Backend

  backend:

    build: .

    container_name: notes-backend-python

    restart: unless-stopped

    depends_on:

      mysql:

        condition: service_healthy

    environment:

      DB_HOST: mysql

      DB_USER: notes_app

      DB_PASSWORD: AppPassword123!

      DB_NAME: notesdb

      DB_PORT: 3306

    ports:

      - "5001:5000"  # Different port for each stack

    volumes:

      - ./:/app

    networks:

      - notes-network

    healthcheck:

      test: ["CMD", "curl", "-f", "http://localhost:5000/api/health"]

      interval: 30s

      timeout: 10s

      retries: 3



  # phpMyAdmin (optional)

  phpmyadmin:

    image: phpmyadmin/phpmyadmin

    container_name: notes-phpmyadmin-python

    restart: unless-stopped

    depends_on:

      - mysql

    environment:

      PMA_HOST: mysql

      PMA_PORT: 3306

      UPLOAD_LIMIT: 100M

    ports:

      - "8081:80"  # Different port for each stack

    networks:

      - notes-network



volumes:

  mysql_data_python:



networks:

  notes-network:

    driver: bridge

EOF

```



### **2.3 Tạo docker-stack.yml cho Swarm deployment**

```bash

cat > ~/docker-lab/backends/python/docker-stack.yml <<'EOF'

version: '3.8'



services:

  # MySQL Database (1 replica only for data consistency)

  mysql:

    image: 192.168.230.190:5000/mysql:8

    deploy:

      replicas: 1

      placement:

        constraints:

          - node.role == manager

      restart_policy:

        condition: on-failure

        delay: 5s

        max_attempts: 3

    environment:

      MYSQL_ROOT_PASSWORD: RootPass123!

      MYSQL_DATABASE: notesdb

    volumes:

      - mysql_data_python:/var/lib/mysql

      - /home/khai/docker-lab/shared/mysql/init:/docker-entrypoint-initdb.d

    networks:

      - notes-network-overlay



  # Python Backend (multiple replicas)

  backend:

    image: 192.168.230.190:5000/notes-backend-python:v1

    deploy:

      replicas: 3

      update_config:

        parallelism: 1

        delay: 10s

        order: start-first

      restart_policy:

        condition: on-failure

      placement:

        constraints:

          - node.role == worker

    environment:

      DB_HOST: mysql

      DB_USER: notes_app

      DB_PASSWORD: AppPassword123!

      DB_NAME: notesdb

      DB_PORT: 3306

    networks:

      - notes-network-overlay



  # Frontend (React)

  frontend:

    image: 192.168.230.190:5000/notes-frontend:v1

    deploy:

      replicas: 2

      update_config:

        parallelism: 1

        delay: 5s

      restart_policy:

        condition: on-failure

    ports:

      - target: 80

        published: 5000

        protocol: tcp

        mode: host

    networks:

      - notes-network-overlay



volumes:

  mysql_data_python:

    driver: local



networks:

  notes-network-overlay:

    driver: overlay

    attachable: true

EOF

```



## **BƯỚC 3: TẠO SCRIPT TIỆN ÍCH**



### **3.1 Tạo build script**

```bash

cat > ~/docker-lab/scripts/build-python.sh <<'EOF'

#!/bin/bash

echo "=== Building Python Notes App ==="



# Build backend image

echo "1. Building Python backend..."

cd ~/docker-lab/backends/python

docker build -t notes-backend-python:latest .



# Build frontend image

echo "2. Building React frontend..."

cd ~/docker-lab/frontend

docker build -t notes-frontend:latest .



echo "=== Build complete ==="

echo "Backend image: notes-backend-python:latest"

echo "Frontend image: notes-frontend:latest"

EOF



chmod +x ~/docker-lab/scripts/build-python.sh

```



### **3.2 Tạo deploy script**

```bash

cat > ~/docker-lab/scripts/deploy-python-stack.sh <<'EOF'

#!/bin/bash

echo "=== Deploying Python Stack to Swarm ==="



# Build images first

~/docker-lab/scripts/build-python.sh



# Deploy stack

echo "Deploying stack..."

cd ~/docker-lab/backends/python

docker stack deploy -c docker-stack.yml notes-python



echo "=== Deployment complete ==="

echo "Check services: docker stack services notes-python"

echo "Check tasks: docker service ps notes-python_backend"

echo ""

echo "Access points:"

echo "- Frontend: http://192.168.230.190:5000"

echo "- Backend API: http://192.168.230.190:5000/api/health"

EOF



chmod +x ~/docker-lab/scripts/deploy-python-stack.sh

```



## **BƯỚC 4: SAO CHÉP SANG CÁC NODES KHÁC**



### **4.1 Copy to docker2 và docker3**

```bash

# Tạo script để copy

cat > ~/copy-to-nodes.sh <<'EOF'

#!/bin/bash

echo "Copying lab structure to docker2..."

scp -r ~/docker-lab slave@docker2:~/



echo "Copying lab structure to docker3..."

scp -r ~/docker-lab khai@docker3:~/



echo "Copy complete!"

EOF



chmod +x ~/copy-to-nodes.sh



# Chạy copy

./copy-to-nodes.sh

```



## **KIỂM TRA:**



### **Trên docker1:**

```bash

# Kiểm tra cấu trúc

cd ~/docker-lab

tree -L 3



# Kết quả mong đợi:

# docker-lab/

# ├── backends

# │   ├── python

# │   │   ├── Dockerfile

# │   │   ├── app.py

# │   │   ├── docker-compose.yml

# │   │   ├── docker-stack.yml

# │   │   ├── requirements.txt

# │   │   └── .env

# ├── frontend

# │   ├── Dockerfile

# │   ├── nginx.conf

# │   ├── package.json

# │   ├── public

# │   └── src

# └── shared

#     └── mysql

#         └── init

```



**Xong Phase 3!** Đã chuẩn bị source code cho Python stack đầu tiên. Tiếp theo Phase 4: Build và test Python stack.

---

# **PHASE 5: BUILD VÀ TEST PYTHON STACK**



## **BƯỚC 1: BUILD IMAGES**



### **1.1 Build React Frontend Image**

```bash

# Trên docker1

cd ~/docker-lab/frontend



# Cài dependencies và build React app

npm install



# Build Docker image

docker build -t notes-frontend:latest .



# Kiểm tra image đã build

docker images | grep notes-frontend

```



### **1.2 Build Python Backend Image**

```bash

# Build Python backend

cd ~/docker-lab/backends/python



# Build Docker image

docker build -t notes-backend-python:latest .



# Kiểm tra

docker images | grep notes-backend-python

```



## **BƯỚC 2: TEST VỚI DOCKER COMPOSE (DEV MODE)**



### **2.1 Chạy stack với docker-compose**

```bash

cd ~/docker-lab/backends/python



# Chạy services

docker compose up -d



# Kiểm tra services đang chạy

docker compose ps



# Kết quả mong đợi:

# NAME                     COMMAND                  SERVICE             STATUS              PORTS

# notes-backend-python     "python app.py"          backend             running             0.0.0.0:5001->5000/tcp

# notes-mysql-python       "docker-entrypoint.s…"   mysql               running             0.0.0.0:3307->3306/tcp

# notes-phpmyadmin-python  "/docker-entrypoint.…"   phpmyadmin          running             0.0.0.0:8081->80/tcp

```



### **2.2 Kiểm tra logs**

```bash

# Xem logs MySQL (kiểm tra init script chạy)

docker compose logs mysql | tail -20



# Xem logs Python backend

docker compose logs backend | tail -20



# Xem tất cả logs

docker compose logs -f

```



### **2.3 Kiểm tra database init**

```bash

# Kiểm tra MySQL container

docker exec -it notes-mysql-python mysql -unotes_app -pAppPassword123! -D notesdb -e "SHOW TABLES;"



# Kết quả mong đợi:

# +-------------------+

# | Tables_in_notesdb |

# +-------------------+

# | notes             |

# +-------------------+



# Kiểm tra dữ liệu mẫu

docker exec -it notes-mysql-python mysql -unotes_app -pAppPassword123! -D notesdb \

  -e "SELECT id, title, status FROM notes;"

```



## **BƯỚC 3: TEST API ENDPOINTS**



### **3.1 Test Python Backend API**

```bash

# Health check

curl http://localhost:5001/api/health



# Kết quả mong đợi:

# {"status":"healthy","service":"Python Backend"}



# Get all notes

curl http://localhost:5001/api/notes



# Create new note

curl -X POST http://localhost:5001/api/notes \

  -H "Content-Type: application/json" \

  -d '{"title":"Test Note from API","content":"This is a test note","status":"pending"}'



# Update note status

curl -X PUT http://localhost:5001/api/notes/1/status \

  -H "Content-Type: application/json" \

  -d '{"status":"in_progress"}'

```



### **3.2 Test Frontend**

```bash

# Build và chạy frontend riêng để test

cd ~/docker-lab/frontend



# Chạy React dev server (tạm thời)

npm start &

# Hoặc truy cập trực tiếp qua container

```



## **BƯỚC 4: DEPLOY LÊN DOCKER SWARM**



### **4.1 Setup Docker Registry (trên docker1)**

```bash

# Trên docker1 (manager) - Chạy Docker Registry container

docker run -d \

  --name registry \

  -p 5000:5000 \

  --restart always \

  registry:2



# Kiểm tra registry đã chạy

curl http://localhost:5000/v2/_catalog



# Kết quả mong đợi: {"repositories":[]}

```



### **4.2 Tag và Push images lên Registry**

```bash

# Trên docker1 - Tag images với registry address

docker tag notes-frontend:latest localhost:5000/notes-frontend:v1

docker tag notes-backend-python:latest localhost:5000/notes-backend-python:v1

docker tag mysql:8 localhost:5000/mysql:8



# Push lên registry

docker push localhost:5000/notes-frontend:v1

docker push localhost:5000/notes-backend-python:v1

docker push localhost:5000/mysql:8



# Kiểm tra images đã push

curl http://localhost:5000/v2/_catalog



# Kết quả mong đợi: {"repositories":["mysql","notes-backend-python","notes-frontend"]}

```



### **4.3 Cấu hình Docker daemon trên tất cả nodes để trust registry**

```bash

# Trên docker1, docker2, docker3 - Cấu hình insecure registry

sudo tee /etc/docker/daemon.json << 'EOF'

{

  "insecure-registries": ["192.168.230.190:5000"]

}

EOF



# Khởi động lại Docker trên từng node

sudo systemctl restart docker



# Kiểm tra cấu hình đã áp dụng

docker info | grep -A 5 "Insecure Registries"

```



### **4.4 Pull images từ Registry trên worker nodes (tùy chọn)**

```bash

# Trên docker2 và docker3 - Pull images từ registry để tăng tốc deployment

# Swarm sẽ tự động pull nếu không có, nhưng pull trước giúp deployment nhanh hơn



# SSH vào docker2

ssh khai@docker2 "docker pull 192.168.230.190:5000/notes-frontend:v1"

ssh khai@docker2 "docker pull 192.168.230.190:5000/notes-backend-python:v1"

ssh khai@docker2 "docker pull 192.168.230.190:5000/mysql:8"



# SSH vào docker3

ssh khai@docker3 "docker pull 192.168.230.190:5000/notes-frontend:v1"

ssh khai@docker3 "docker pull 192.168.230.190:5000/notes-backend-python:v1"

ssh khai@docker3 "docker pull 192.168.230.190:5000/mysql:8"



# Kiểm tra images đã pull

ssh khai@docker2 "docker images | grep 192.168.230.190:5000"

ssh khai@docker3 "docker images | grep 192.168.230.190:5000"

```



### **4.5 Deploy stack lên Swarm**

```bash

# Trên docker1 (manager)

cd ~/docker-lab/backends/python



# Deploy stack với registry images

docker stack deploy -c docker-stack.yml notes-python



# Kiểm tra services

docker stack services notes-python



# Kết quả mong đợi:

# ID             NAME                       MODE         REPLICAS   IMAGE                                               PORTS

# xxx            notes-python_backend       replicated   3/3        192.168.230.190:5000/notes-backend-python:v1

# yyy            notes-python_frontend      replicated   2/2        192.168.230.190:5000/notes-frontend:v1              *:5000->80/tcp

# zzz            notes-python_mysql         replicated   1/1        192.168.230.190:5000/mysql:8

```



### **4.6 Kiểm tra Swarm deployment**

```bash

# Xem tất cả services trong stack

docker stack ps notes-python



# Xem logs của service

docker service logs notes-python_backend



# Scale backend service nếu cần

docker service scale notes-python_backend=5



# Kiểm tra phân bổ mới

docker service ps notes-python_backend

```



## **BƯỚC 5: KIỂM TRA TRUY CẬP**



### **5.1 Truy cập ứng dụng**

```bash

# Frontend (React)

echo "Frontend URL: http://192.168.230.190:5000"

echo "Backend API: http://192.168.230.190:5000/api/health"

```



### **5.2 Test từ browser**

```

1. Mở browser, truy cập: http://192.168.230.190:5000

2. Kiểm tra có hiển thị "Notes App"

3. Thêm, sửa, xóa notes

4. Kiểm tra status changes

```



### **5.3 Test load balancing**

```bash

# Test multiple requests để thấy load balancing

for i in {1..10}; do

  curl -s http://192.168.230.190:5000/api/health | grep -o '"service":".*"' 

done | sort | uniq -c

```



## **BƯỚC 6: TROUBLESHOOTING**



### **Nếu có lỗi:**

```bash

# 1. Kiểm tra registry

curl http://192.168.230.190:5000/v2/_catalog



# 2. Kiểm tra network

docker network ls | grep notes-network-overlay



# 3. Kiểm tra volumes

docker volume ls | grep mysql



# 4. Kiểm tra các nodes có pull được images từ registry

ssh docker2 "docker pull 192.168.230.190:5000/notes-frontend:v1"

ssh docker3 "docker pull 192.168.230.190:5000/notes-frontend:v1"



# 5. Xem detailed logs

docker service logs --tail 100 notes-python_backend

docker service logs --tail 100 notes-python_mysql



# 6. Kiểm tra service health

docker service inspect --pretty notes-python_backend

```



### **Common issues:**

```bash

# Nếu service không start

docker service update --force notes-python_backend



# Nếu network không tạo

docker network create -d overlay --attachable notes-network-overlay



# Nếu volume không tồn tại

docker volume create mysql_data_python



# Nếu registry không trust được

sudo systemctl restart docker  # Trên tất cả nodes sau khi sửa daemon.json

```



## **BƯỚC 7: CLEANUP (nếu cần)**



### **7.1 Dừng stack**

```bash

# Remove stack

docker stack rm notes-python



# Dừng docker compose

cd ~/docker-lab/backends/python

docker compose down



# Xóa volumes (cẩn thận - mất data)

docker compose down -v

```



### **7.2 Xóa images**

```bash

# Xóa images cục bộ

docker rmi notes-frontend:latest notes-backend-python:latest



# Xóa trên tất cả nodes

ssh docker2 "docker rmi notes-frontend:latest notes-backend-python:latest"

ssh docker3 "docker rmi notes-frontend:latest notes-backend-python:latest"

```



## **KIỂM TRA CUỐI CÙNG:**



### **Tạo test script**

```bash

cat > ~/test-python-stack.sh <<'EOF'

#!/bin/bash

echo "=== Testing Python Stack ==="

echo ""



echo "1. Docker Swarm Status:"

docker node ls

echo ""



echo "2. Stack Services:"

docker stack services notes-python 2>/dev/null || echo "Stack not deployed yet"

echo ""



echo "3. API Health Check:"

curl -s http://localhost:5001/api/health 2>/dev/null || echo "API not reachable"

echo ""



echo "4. Frontend Access:"

echo "URL: http://192.168.230.190:5000"

echo ""



echo "5. Database Connection:"

docker exec notes-mysql-python mysql -unotes_app -pAppPassword123! -D notesdb \

  -e "SELECT COUNT(*) as note_count FROM notes;" 2>/dev/null || echo "Database not reachable"

EOF



chmod +x ~/test-python-stack.sh

```



**Chạy test:**

```bash

./test-python-stack.sh

```



## **KẾT QUẢ THÀNH CÔNG KHI:**

- ✅ `docker stack services notes-python` hiện 3 services

- ✅ Frontend truy cập được qua browser

- ✅ API trả về `{"status":"healthy"}`

- ✅ Database có dữ liệu mẫu



**Xong Phase 5!** Bạn đã deploy thành công Python stack đầu tiên trên Docker Swarm!