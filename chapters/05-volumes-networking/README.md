# บทที่ 5: Docker Volumes และ Networking

## ภาพรวม

ในบทนี้เราจะเรียนรู้วิธีการจัดการข้อมูลถาวรด้วย Volumes และการสื่อสารระหว่าง containers ด้วย Networking

## Docker Volumes

### ปัญหาของการเก็บข้อมูลใน Container

เมื่อ container ถูกลบ ข้อมูลทั้งหมดภายในจะหายไปด้วย:

```bash
# รัน PostgreSQL container
docker run -d --name db postgres

# เพิ่มข้อมูล
docker exec -it db psql -U postgres -c "CREATE DATABASE myapp;"

# ลบ container
docker rm -f db

# ข้อมูลหายหมดแล้ว! 😢
```

**Solution:** ใช้ Volumes เพื่อเก็บข้อมูลถาวร

### ประเภทของ Data Storage

Docker มี 3 วิธีในการเก็บข้อมูล:

#### 1. Volumes (แนะนำ)

```
┌─────────────────────────────┐
│     Docker Host             │
│                             │
│  ┌──────────┐  ┌─────────┐ │
│  │Container │  │ Volume  │ │
│  │  /data ──┼──┼─ /data  │ │
│  └──────────┘  └─────────┘ │
│                             │
│  /var/lib/docker/volumes/   │
└─────────────────────────────┘
```

- จัดการโดย Docker
- เก็บใน `/var/lib/docker/volumes/`
- แยกออกจาก container lifecycle
- สามารถแชร์ระหว่าง containers

#### 2. Bind Mounts

```
┌─────────────────────────────┐
│     Docker Host             │
│                             │
│  ┌──────────┐               │
│  │Container │               │
│  │  /app  ──┼───┐           │
│  └──────────┘   │           │
│                 │           │
│  ┌──────────────▼────────┐  │
│  │ /home/user/project    │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

- Map directory จาก host เข้า container
- สามารถเข้าถึงจาก host ได้โดยตรง
- เหมาะสำหรับ development

#### 3. tmpfs Mounts

```
┌─────────────────────────────┐
│     Docker Host             │
│                             │
│  ┌──────────┐  ┌─────────┐ │
│  │Container │  │ Memory  │ │
│  │  /tmp  ──┼──┼─ RAM    │ │
│  └──────────┘  └─────────┘ │
│                             │
│  (ไม่เก็บใน disk)           │
└─────────────────────────────┘
```

- เก็บใน memory
- รวดเร็ว แต่ข้อมูลหายเมื่อ container หยุด
- เหมาะสำหรับข้อมูลชั่วคราว

## การใช้งาน Volumes

### สร้าง Volume

```bash
# สร้าง volume
docker volume create my-volume

# ดู volumes ทั้งหมด
docker volume ls

# ดูข้อมูลของ volume
docker volume inspect my-volume

# ลบ volume
docker volume rm my-volume
```

### ใช้ Volume กับ Container

```bash
# รันพร้อม volume (named volume)
docker run -d \
  --name myapp \
  -v my-volume:/data \
  nginx

# หรือใช้ --mount (ชัดเจนกว่า)
docker run -d \
  --name myapp \
  --mount source=my-volume,target=/data \
  nginx
```

### ตัวอย่างการใช้งาน

**ตัวอย่างที่ 1: PostgreSQL Database**

```bash
# สร้าง volume สำหรับ database
docker volume create postgres-data

# รัน PostgreSQL
docker run -d \
  --name postgres-db \
  -e POSTGRES_PASSWORD=secret \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# เพิ่มข้อมูล
docker exec -it postgres-db psql -U postgres -c "CREATE DATABASE myapp;"

# ลบ container
docker rm -f postgres-db

# รัน container ใหม่ (ข้อมูลยังอยู่!)
docker run -d \
  --name postgres-db \
  -e POSTGRES_PASSWORD=secret \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# ตรวจสอบข้อมูล (database ยังอยู่!)
docker exec -it postgres-db psql -U postgres -c "\l"
```

**ตัวอย่างที่ 2: MongoDB**

```bash
docker volume create mongo-data

docker run -d \
  --name mongodb \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  -v mongo-data:/data/db \
  mongo:6
```

**ตัวอย่างที่ 3: MySQL**

```bash
docker volume create mysql-data

docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  -v mysql-data:/var/lib/mysql \
  mysql:8
```

### Bind Mounts

```bash
# ใช้ absolute path
docker run -d \
  --name web \
  -v /home/user/website:/usr/share/nginx/html \
  nginx

# ใช้ relative path (current directory)
docker run -d \
  --name web \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx

# Read-only mount
docker run -d \
  --name web \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx
```

**ตัวอย่าง: Development Environment**

```bash
# สร้าง Node.js app
mkdir my-app && cd my-app
npm init -y
npm install express

# สร้าง server.js
cat > server.js << 'EOF'
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello from Docker!');
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

# รันพร้อม bind mount (แก้โค้ดบน host = อัปเดตใน container)
docker run -d \
  --name node-dev \
  -p 3000:3000 \
  -v $(pwd):/app \
  -w /app \
  node:18 \
  sh -c "npm install && node server.js"
```

### แชร์ Volume ระหว่าง Containers

```bash
# สร้าง volume
docker volume create shared-data

# Container 1: เขียนข้อมูล
docker run -d \
  --name writer \
  -v shared-data:/data \
  alpine \
  sh -c "while true; do echo $(date) >> /data/log.txt; sleep 5; done"

# Container 2: อ่านข้อมูล
docker run -d \
  --name reader \
  -v shared-data:/data:ro \
  alpine \
  sh -c "tail -f /data/log.txt"

# ดู logs
docker logs -f reader
```

### จัดการ Volumes

```bash
# ลบ volumes ที่ไม่ได้ใช้งาน
docker volume prune

# ลบทั้งหมด (ระวัง!)
docker volume prune -a

# Backup volume
docker run --rm \
  -v my-volume:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/my-volume-backup.tar.gz /data

# Restore volume
docker run --rm \
  -v my-volume:/data \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/my-volume-backup.tar.gz -C /
```

## Docker Networking

### ประเภทของ Networks

#### 1. Bridge Network (Default)

```
┌─────────────────────────────────────┐
│         Docker Host                 │
│                                     │
│  ┌──────────┐      ┌──────────┐    │
│  │Container │      │Container │    │
│  │    A     │──────│    B     │    │
│  └──────────┘      └──────────┘    │
│        │                │           │
│        └────────┬───────┘           │
│                 │                   │
│          ┌──────▼──────┐            │
│          │   Bridge    │            │
│          └──────┬──────┘            │
│                 │                   │
│          ┌──────▼──────┐            │
│          │    Host     │            │
│          └─────────────┘            │
└─────────────────────────────────────┘
```

- Default network
- Container สื่อสารกันได้ภายใน network เดียวกัน
- ต้อง publish port เพื่อเข้าถึงจากภายนอก

#### 2. Host Network

```
┌─────────────────────────────────────┐
│         Docker Host                 │
│                                     │
│  ┌──────────────────────────────┐   │
│  │    Container (Host Network)  │   │
│  │  ใช้ network stack ของ host │   │
│  └──────────────────────────────┘   │
│                                     │
│  ไม่มี network isolation            │
└─────────────────────────────────────┘
```

- Container ใช้ network ของ host โดยตรง
- ไม่ต้อง publish port
- Performance ดีที่สุด

#### 3. None Network

- ไม่มี network
- Container แยกอย่างสมบูรณ์

#### 4. Overlay Network

- สำหรับ Docker Swarm
- Container บนหลาย hosts สื่อสารกันได้

### การจัดการ Networks

```bash
# ดู networks ทั้งหมด
docker network ls

# สร้าง network
docker network create my-network

# สร้าง network แบบกำหนด subnet
docker network create \
  --subnet=172.20.0.0/16 \
  --gateway=172.20.0.1 \
  my-custom-network

# ดูข้อมูล network
docker network inspect my-network

# ลบ network
docker network rm my-network

# ลบ networks ที่ไม่ได้ใช้
docker network prune
```

### เชื่อมต่อ Container เข้า Network

```bash
# รัน container ใน network
docker run -d \
  --name web \
  --network my-network \
  nginx

# เชื่อมต่อ container ที่มีอยู่เข้า network
docker network connect my-network existing-container

# ตัดการเชื่อมต่อ
docker network disconnect my-network existing-container
```

### ตัวอย่างการใช้งาน

**ตัวอย่างที่ 1: Web + Database**

```bash
# สร้าง network
docker network create app-network

# รัน database
docker run -d \
  --name postgres \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  postgres:15

# รัน web application
docker run -d \
  --name web \
  --network app-network \
  -p 8080:80 \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  -e DB_NAME=myapp \
  my-web-app

# Web app สามารถเชื่อมต่อ database ด้วยชื่อ "postgres"
```

**ตัวอย่างที่ 2: Microservices**

```bash
# สร้าง network
docker network create microservices

# User Service
docker run -d \
  --name user-service \
  --network microservices \
  -e PORT=3001 \
  user-service:latest

# Order Service
docker run -d \
  --name order-service \
  --network microservices \
  -e PORT=3002 \
  -e USER_SERVICE_URL=http://user-service:3001 \
  order-service:latest

# API Gateway
docker run -d \
  --name api-gateway \
  --network microservices \
  -p 8080:8080 \
  -e USER_SERVICE_URL=http://user-service:3001 \
  -e ORDER_SERVICE_URL=http://order-service:3002 \
  api-gateway:latest
```

**ตัวอย่างที่ 3: Frontend + Backend + Database**

```bash
# สร้าง network
docker network create fullstack

# Database
docker run -d \
  --name db \
  --network fullstack \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Backend API
docker run -d \
  --name backend \
  --network fullstack \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
  backend:latest

# Frontend
docker run -d \
  --name frontend \
  --network fullstack \
  -p 80:80 \
  -e API_URL=http://backend:3000 \
  frontend:latest
```

### การใช้ Host Network

```bash
# รันด้วย host network
docker run -d \
  --name web \
  --network host \
  nginx

# เข้าถึงได้ที่ http://localhost (ไม่ต้อง -p)
```

**เมื่อไหร่ควรใช้ Host Network:**
- ต้องการ performance สูงสุด
- ต้องการเข้าถึง port หลายพันพร้อมกัน
- ไม่ต้องการ port mapping

### DNS และ Service Discovery

```bash
# Container สามารถหากันด้วยชื่อ
docker network create mynet

docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet \
  -e DB_HOST=db \
  my-app

# ใน app container สามารถใช้ "db" แทน IP address
```

### Network Aliases

```bash
# ให้ชื่อเล่นแก่ container
docker run -d \
  --name db \
  --network mynet \
  --network-alias database \
  --network-alias postgres-server \
  postgres

# สามารถเข้าถึงด้วย: db, database, หรือ postgres-server
```

## Best Practices

### Volumes

1. **ใช้ Named Volumes แทน Anonymous Volumes**
```bash
# ✅ ดี
docker run -v my-data:/data nginx

# ❌ ไม่ดี
docker run -v /data nginx
```

2. **Backup Volumes สม่ำเสมอ**
```bash
# Backup script
docker run --rm \
  -v my-volume:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/backup-$(date +%Y%m%d).tar.gz /data
```

3. **ใช้ Read-Only Mounts เมื่อเป็นไปได้**
```bash
docker run -v $(pwd)/config:/config:ro nginx
```

### Networking

1. **สร้าง Custom Networks แทนการใช้ Default Bridge**
```bash
# ✅ ดี
docker network create my-app
docker run --network my-app nginx

# ❌ ไม่ดี
docker run nginx  # ใช้ default bridge
```

2. **แยก Networks ตาม Security Zones**
```bash
# Frontend network
docker network create frontend

# Backend network
docker network create backend

# Database อยู่ใน backend เท่านั้น
docker run --network backend postgres
```

3. **ใช้ Environment Variables สำหรับ Configuration**
```bash
docker run \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  my-app
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Persistent Database

```bash
# 1. สร้าง volume
docker volume create wordpress-db

# 2. รัน MySQL
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=wordpress \
  -v wordpress-db:/var/lib/mysql \
  mysql:8

# 3. ทดสอบ persistence
docker exec -it mysql mysql -p -e "CREATE TABLE test (id INT);"
docker rm -f mysql
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=secret -v wordpress-db:/var/lib/mysql mysql:8
docker exec -it mysql mysql -p -e "SHOW TABLES;"
```

### แบบฝึกหัดที่ 2: Multi-Container Application

```bash
# 1. สร้าง network
docker network create wordpress-net

# 2. รัน database
docker run -d \
  --name wp-db \
  --network wordpress-net \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=wordpress \
  -v wp-db-data:/var/lib/mysql \
  mysql:8

# 3. รัน WordPress
docker run -d \
  --name wordpress \
  --network wordpress-net \
  -p 8080:80 \
  -e WORDPRESS_DB_HOST=wp-db \
  -e WORDPRESS_DB_NAME=wordpress \
  -e WORDPRESS_DB_PASSWORD=secret \
  wordpress:latest

# 4. เข้าใช้งานที่ http://localhost:8080
```

### แบบฝึกหัดที่ 3: Development Environment

```bash
# 1. สร้าง network
docker network create dev-net

# 2. รัน Redis
docker run -d --name redis --network dev-net redis:alpine

# 3. รัน PostgreSQL
docker run -d \
  --name postgres \
  --network dev-net \
  -e POSTGRES_PASSWORD=dev \
  postgres:15

# 4. รัน application
docker run -it \
  --name app \
  --network dev-net \
  -v $(pwd):/app \
  -w /app \
  -p 3000:3000 \
  node:18 bash
```

## สรุป

ในบทนี้เราได้เรียนรู้:

- ✅ Docker Volumes: การสร้าง, การใช้งาน, bind mounts, backup/restore
- ✅ Docker Networking: ประเภทของ networks, การสร้าง, การเชื่อมต่อ containers
- ✅ Service Discovery และ DNS
- ✅ Best Practices สำหรับ volumes และ networking
- ✅ ตัวอย่างการใช้งานจริง

## ถัดไป

ในบทต่อไปเราจะเรียนรู้ Docker Compose ซึ่งช่วยให้จัดการ multi-container applications ได้ง่ายขึ้น

➡️ [บทที่ 6: Docker Compose](../06-docker-compose/README.md)

⬅️ [บทที่ 4: การสร้าง Docker Images ด้วย Dockerfile](../04-dockerfile/README.md)
