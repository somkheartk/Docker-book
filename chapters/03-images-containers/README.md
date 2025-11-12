# บทที่ 3: Docker Images และ Containers

## ภาพรวม

ในบทนี้เราจะเจาะลึกเกี่ยวกับ Docker Images และ Containers ซึ่งเป็นแนวคิดหลักของ Docker

## Docker Images คืออะไร

Docker Image เป็น template แบบ read-only ที่ใช้สำหรับสร้าง containers มันประกอบด้วย:

- Operating system (หรือ base OS)
- Application code
- Libraries และ dependencies
- Environment variables
- Configuration files

### โครงสร้างของ Image

Images ประกอบด้วย layers หลายๆ ชั้นซ้อนกัน:

```
┌─────────────────────────┐
│  Application Layer      │  <- Your app
├─────────────────────────┤
│  Dependencies Layer     │  <- npm, pip packages
├─────────────────────────┤
│  Runtime Layer          │  <- Node.js, Python
├─────────────────────────┤
│  OS Layer              │  <- Ubuntu, Alpine
└─────────────────────────┘
```

แต่ละ layer:
- เป็น read-only
- สามารถแชร์กันระหว่าง images ได้
- ช่วยประหยัดพื้นที่และเวลา

## การทำงานกับ Docker Images

### ค้นหา Images บน Docker Hub

```bash
# ค้นหา image
docker search nginx

# ค้นหา official images เท่านั้น
docker search --filter "is-official=true" nginx

# จำกัดจำนวนผลลัพธ์
docker search --limit 5 python
```

### ดึง (Pull) Images

```bash
# ดึง image version ล่าสุด
docker pull nginx

# ดึง image version เฉพาะ
docker pull nginx:1.24

# ดึงจาก registry อื่น
docker pull gcr.io/google-samples/hello-app:1.0
```

### ดู Images ที่มีในเครื่อง

```bash
# แสดงทั้งหมด
docker images

# หรือ
docker image ls

# แสดงเฉพาะ image IDs
docker images -q

# แสดงรายละเอียดเพิ่มเติม
docker images --no-trunc
```

**ตัวอย่างผลลัพธ์:**
```
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    605c77e624dd   2 weeks ago    141MB
ubuntu       22.04     3b418d7b466a   3 weeks ago    77.8MB
python       3.11      a5d7930b60cc   1 month ago    917MB
```

### ดูรายละเอียด Image

```bash
# ดูข้อมูล metadata
docker image inspect nginx

# ดูประวัติการสร้าง layers
docker history nginx
```

### ลบ Images

```bash
# ลบ image ตามชื่อและ tag
docker rmi nginx:latest

# ลบ image ตาม ID
docker rmi 605c77e624dd

# ลบหลาย images พร้อมกัน
docker rmi nginx ubuntu python

# ลบ images ที่ไม่ได้ใช้งาน (dangling)
docker image prune

# ลบ images ทั้งหมดที่ไม่ได้ใช้งาน
docker image prune -a
```

### การติด Tag ให้ Images

```bash
# สร้าง tag ใหม่สำหรับ image ที่มีอยู่
docker tag nginx:latest myregistry.com/nginx:v1.0

# Tag ตาม image ID
docker tag 605c77e624dd mynginx:prod
```

## Docker Containers คืออะไร

Container เป็น running instance ของ image มันคือ:

- Isolated process บนเครื่อง host
- มี filesystem, networking, และ process tree ของตัวเอง
- เบาและรันเร็ว
- สามารถ start, stop, move, delete ได้

### Container Lifecycle

```
┌──────────┐
│ Created  │
└────┬─────┘
     │ docker start
     ▼
┌──────────┐
│ Running  │◄──────┐
└────┬─────┘       │
     │             │ docker restart
     │ docker stop │
     ▼             │
┌──────────┐       │
│ Stopped  │───────┘
└────┬─────┘
     │ docker rm
     ▼
┌──────────┐
│ Removed  │
└──────────┘
```

## การทำงานกับ Containers

### สร้างและรัน Container

**รูปแบบพื้นฐาน:**
```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

**ตัวอย่าง:**

```bash
# รัน container แบบง่าย
docker run nginx

# รันในโหมด detached (background)
docker run -d nginx

# ตั้งชื่อ container
docker run -d --name my-nginx nginx

# Map port
docker run -d -p 8080:80 nginx

# Map หลาย ports
docker run -d -p 8080:80 -p 8443:443 nginx

# รันแบบ interactive
docker run -it ubuntu bash

# ลบอัตโนมัติเมื่อหยุดทำงาน
docker run --rm ubuntu echo "Hello Docker"

# กำหนด environment variables
docker run -e "DB_HOST=localhost" -e "DB_PORT=5432" postgres

# Mount volume
docker run -v /host/path:/container/path nginx
```

### ดูสถานะ Containers

```bash
# ดู containers ที่กำลังรัน
docker ps

# ดูทั้งหมด (รวมที่หยุดแล้ว)
docker ps -a

# แสดงเฉพาะ IDs
docker ps -q

# แสดงเฉพาะ container ล่าสุด
docker ps -l

# กรองด้วย filters
docker ps --filter "status=running"
docker ps --filter "name=nginx"
```

**ตัวอย่างผลลัพธ์:**
```
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                  NAMES
abc123def456   nginx     "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->80/tcp   my-nginx
```

### จัดการ Containers

**เริ่มต้นและหยุด:**

```bash
# เริ่มต้น container
docker start my-nginx

# หยุด container (graceful shutdown)
docker stop my-nginx

# บังคับหยุด (kill)
docker kill my-nginx

# รีสตาร์ท
docker restart my-nginx
```

**หยุดชั่วคราวและทำงานต่อ:**

```bash
# หยุดชั่วคราว (pause)
docker pause my-nginx

# ทำงานต่อ (unpause)
docker unpause my-nginx
```

**ลบ Containers:**

```bash
# ลบ container (ต้องหยุดก่อน)
docker rm my-nginx

# บังคับลบ (ถึงแม้กำลังรัน)
docker rm -f my-nginx

# ลบหลาย containers
docker rm container1 container2 container3

# ลบ containers ที่หยุดแล้วทั้งหมด
docker container prune
```

### ดูข้อมูลและ Logs

**ดู logs:**

```bash
# แสดง logs
docker logs my-nginx

# แสดงแบบ real-time (follow)
docker logs -f my-nginx

# แสดง n บรรทัดท้ายสุด
docker logs --tail 100 my-nginx

# แสดงพร้อม timestamp
docker logs -t my-nginx
```

**ดูข้อมูลรายละเอียด:**

```bash
# ดู metadata
docker inspect my-nginx

# ดูเฉพาะ IP address
docker inspect -f '{{.NetworkSettings.IPAddress}}' my-nginx

# ดูการใช้ทรัพยากร
docker stats my-nginx

# ดูทุก containers
docker stats
```

**ดู processes:**

```bash
# แสดง processes ใน container
docker top my-nginx
```

### การโต้ตอบกับ Running Container

**เข้าไปทำงานใน container:**

```bash
# เปิด bash shell
docker exec -it my-nginx bash

# หรือ sh สำหรับ Alpine-based images
docker exec -it my-nginx sh

# รันคำสั่งเดียว
docker exec my-nginx ls /usr/share/nginx/html

# รันคำสั่งด้วยสิทธิ์ root
docker exec -u root -it my-nginx bash
```

**คัดลอกไฟล์:**

```bash
# จาก host ไป container
docker cp /path/on/host my-nginx:/path/in/container

# จาก container มา host
docker cp my-nginx:/path/in/container /path/on/host
```

### ตัวอย่างการใช้งานจริง

**ตัวอย่างที่ 1: รัน Web Server**

```bash
# รัน Nginx
docker run -d \
  --name web-server \
  -p 80:80 \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx

# ทดสอบ
curl http://localhost
```

**ตัวอย่างที่ 2: รัน Database**

```bash
# รัน PostgreSQL
docker run -d \
  --name postgres-db \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:15

# เชื่อมต่อด้วย psql
docker exec -it postgres-db psql -U postgres -d myapp
```

**ตัวอย่างที่ 3: Development Environment**

```bash
# รัน Node.js development container
docker run -it \
  --name node-dev \
  -v $(pwd):/app \
  -w /app \
  -p 3000:3000 \
  node:18 \
  bash

# ใน container
# npm install
# npm start
```

**ตัวอย่างที่ 4: รัน Multiple Containers**

```bash
# รัน Redis
docker run -d --name redis-cache redis:alpine

# รัน Application ที่เชื่อมต่อกับ Redis
docker run -d \
  --name my-app \
  --link redis-cache:redis \
  -p 8080:8080 \
  my-app-image
```

## Docker Hub และ Registry

### Docker Hub

Docker Hub เป็น public registry ที่ใหญ่ที่สุดสำหรับ Docker images

**Official Images:**
- `nginx` - Web server
- `node` - Node.js runtime
- `python` - Python runtime
- `postgres` - PostgreSQL database
- `redis` - Redis cache
- `ubuntu` - Ubuntu OS

**การใช้งาน:**

```bash
# ค้นหา
docker search nodejs

# Pull
docker pull node:18-alpine

# ดูข้อมูลเพิ่มเติมที่ hub.docker.com
```

### Push Image ขึ้น Docker Hub

```bash
# 1. Login
docker login

# 2. Tag image (username/image:tag)
docker tag my-app:latest myusername/my-app:v1.0

# 3. Push
docker push myusername/my-app:v1.0
```

### Private Registry

```bash
# รัน private registry
docker run -d -p 5000:5000 --name registry registry:2

# Tag image สำหรับ private registry
docker tag my-app localhost:5000/my-app:v1.0

# Push ไป private registry
docker push localhost:5000/my-app:v1.0

# Pull จาก private registry
docker pull localhost:5000/my-app:v1.0
```

## Best Practices

### 1. การตั้งชื่อ Containers และ Images

```bash
# ดี: ชื่อที่มีความหมาย
docker run --name user-api -d user-service:v1.0

# ไม่ดี: ปล่อยให้ Docker ตั้งชื่อให้เอง
docker run -d user-service:v1.0
```

### 2. การใช้ Tags

```bash
# ดี: ระบุ version
docker pull node:18.16.0-alpine

# ไม่ค่อยดี: ใช้ latest (อาจเปลี่ยนได้)
docker pull node:latest
```

### 3. การลบ Resources ที่ไม่ใช้

```bash
# ลบ stopped containers
docker container prune

# ลบ unused images
docker image prune

# ลบทุกอย่างที่ไม่ใช้
docker system prune -a
```

### 4. Resource Limits

```bash
# กำหนด memory limit
docker run -m 512m nginx

# กำหนด CPU limit
docker run --cpus=".5" nginx

# กำหนดทั้งคู่
docker run -m 512m --cpus=".5" nginx
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: จัดการ Images

```bash
# 1. ค้นหา images
docker search ubuntu

# 2. Pull หลาย versions
docker pull ubuntu:20.04
docker pull ubuntu:22.04
docker pull ubuntu:latest

# 3. ดูรายการ
docker images ubuntu

# 4. ลบ version 20.04
docker rmi ubuntu:20.04
```

### แบบฝึกหัดที่ 2: จัดการ Containers

```bash
# 1. รัน nginx container
docker run -d --name ex-nginx -p 8888:80 nginx

# 2. ตรวจสอบว่ารันอยู่
docker ps

# 3. ดู logs
docker logs ex-nginx

# 4. เข้าไปใน container
docker exec -it ex-nginx bash

# 5. แก้ไขไฟล์ (ใน container)
echo "<h1>Hello Docker!</h1>" > /usr/share/nginx/html/index.html
exit

# 6. ทดสอบในเบราว์เซอร์
# เปิด http://localhost:8888

# 7. หยุดและลบ
docker stop ex-nginx
docker rm ex-nginx
```

### แบบฝึกหัดที่ 3: Container Networking

```bash
# 1. รัน database container
docker run -d --name mydb -e MYSQL_ROOT_PASSWORD=secret mysql:8

# 2. รัน application container ที่เชื่อมต่อกับ database
docker run -d --name myapp --link mydb:mysql my-app-image

# 3. ตรวจสอบการเชื่อมต่อ
docker exec myapp ping -c 3 mydb
```

## สรุป

ในบทนี้เราได้เรียนรู้:

- ✅ Docker Images: โครงสร้าง การ pull, การดู, การลบ, การ tag
- ✅ Docker Containers: lifecycle, การสร้าง, การจัดการ, การโต้ตอบ
- ✅ Docker Hub และ Registry
- ✅ Best Practices สำหรับการใช้งาน Images และ Containers

## ถัดไป

ในบทต่อไปเราจะเรียนรู้วิธีการสร้าง Docker Images ของตัวเองด้วย Dockerfile

➡️ [บทที่ 4: การสร้าง Docker Images ด้วย Dockerfile](../04-dockerfile/README.md)

⬅️ [บทที่ 2: การติดตั้ง Docker](../02-installation/README.md)
