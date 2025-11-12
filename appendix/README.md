# ภาคผนวก

## คำสั่ง Docker ที่ใช้บ่อย

### Container Management

```bash
# รัน container
docker run -d --name myapp -p 8080:80 nginx

# รันแบบ interactive
docker run -it ubuntu bash

# หยุดและเริ่ม container
docker stop myapp
docker start myapp
docker restart myapp

# ดู containers
docker ps              # กำลังรัน
docker ps -a           # ทั้งหมด

# ลบ container
docker rm myapp
docker rm -f myapp     # force remove

# ดู logs
docker logs myapp
docker logs -f myapp   # follow

# เข้าไปใน container
docker exec -it myapp bash

# คัดลอกไฟล์
docker cp file.txt myapp:/path/
docker cp myapp:/path/file.txt ./

# ดูการใช้ทรัพยากร
docker stats
docker stats myapp

# ดูข้อมูล
docker inspect myapp
```

### Image Management

```bash
# ดู images
docker images
docker image ls

# Pull image
docker pull nginx
docker pull nginx:1.24

# Build image
docker build -t myapp:latest .
docker build -f Dockerfile.dev -t myapp:dev .

# Tag image
docker tag myapp:latest myapp:v1.0
docker tag myapp:latest myregistry.com/myapp:latest

# Push image
docker push myapp:latest

# ลบ image
docker rmi nginx
docker rmi -f nginx

# ลบ images ที่ไม่ใช้
docker image prune
docker image prune -a

# ดูประวัติ image
docker history nginx

# Save/Load images
docker save myapp:latest > myapp.tar
docker load < myapp.tar

# Export/Import containers
docker export myapp > myapp.tar
docker import myapp.tar myapp:latest
```

### Volume Management

```bash
# สร้าง volume
docker volume create mydata

# ดู volumes
docker volume ls

# ดูข้อมูล volume
docker volume inspect mydata

# ลบ volume
docker volume rm mydata

# ลบ volumes ที่ไม่ใช้
docker volume prune

# ใช้ volume
docker run -v mydata:/data nginx

# Bind mount
docker run -v $(pwd):/app nginx

# Backup volume
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz /data

# Restore volume
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup.tar.gz -C /
```

### Network Management

```bash
# สร้าง network
docker network create mynetwork

# ดู networks
docker network ls

# ดูข้อมูล network
docker network inspect mynetwork

# เชื่อมต่อ container เข้า network
docker network connect mynetwork myapp

# ตัดการเชื่อมต่อ
docker network disconnect mynetwork myapp

# ลบ network
docker network rm mynetwork

# ลบ networks ที่ไม่ใช้
docker network prune

# รันพร้อม network
docker run --network mynetwork nginx
```

### Docker Compose Commands

```bash
# เริ่มต้น services
docker compose up
docker compose up -d               # detached mode
docker compose up --build          # build ก่อน

# หยุด services
docker compose stop
docker compose down                # หยุดและลบ
docker compose down -v             # ลบพร้อม volumes

# ดูสถานะ
docker compose ps
docker compose top

# ดู logs
docker compose logs
docker compose logs -f
docker compose logs -f web         # เฉพาะ service

# รัน command
docker compose exec web bash
docker compose run web npm test

# Build
docker compose build
docker compose build --no-cache

# Pull/Push images
docker compose pull
docker compose push

# Scale services
docker compose up -d --scale web=3

# Restart services
docker compose restart
docker compose restart web

# ดูการตั้งค่า
docker compose config

# Pause/Unpause
docker compose pause
docker compose unpause
```

### System Management

```bash
# ดูข้อมูลระบบ
docker info

# ดูการใช้พื้นที่
docker system df

# ทำความสะอาดระบบ
docker system prune                # ลบทุกอย่างที่ไม่ใช้
docker system prune -a             # รวม images
docker system prune --volumes      # รวม volumes

# ดู events
docker events
docker events --filter 'type=container'

# Check version
docker --version
docker version
docker compose version
```

## Troubleshooting Guide

### ปัญหาที่พบบ่อยและวิธีแก้ไข

#### 1. Container ไม่สามารถเริ่มต้นได้

**อาการ:**
```bash
$ docker run myapp
Error: Container exited immediately
```

**วิธีแก้:**
```bash
# ดู logs
docker logs myapp

# ดูข้อมูล
docker inspect myapp

# ลองรันแบบ interactive
docker run -it myapp sh

# ตรวจสอบ ENTRYPOINT และ CMD
docker inspect myapp | grep -A 10 "Entrypoint\|Cmd"
```

#### 2. Port Already in Use

**อาการ:**
```bash
Error: Bind for 0.0.0.0:8080 failed: port is already allocated
```

**วิธีแก้:**
```bash
# หา process ที่ใช้ port
lsof -i :8080                    # Mac/Linux
netstat -ano | findstr :8080    # Windows

# ใช้ port อื่น
docker run -p 8081:80 nginx

# หยุด container ที่ใช้ port นั้น
docker ps | grep 8080
docker stop <container_id>
```

#### 3. Cannot Connect to Docker Daemon

**อาการ:**
```bash
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

**วิธีแก้:**
```bash
# เช็คว่า Docker service รันอยู่
systemctl status docker          # Linux
sudo systemctl start docker      # เริ่ม service

# เช็คสิทธิ์ user
sudo usermod -aG docker $USER
newgrp docker

# Docker Desktop
# เปิด Docker Desktop application
```

#### 4. Out of Disk Space

**อาการ:**
```bash
Error: No space left on device
```

**วิธีแก้:**
```bash
# ดูการใช้พื้นที่
docker system df

# ทำความสะอาด
docker system prune -a --volumes

# ลบ stopped containers
docker container prune

# ลบ unused images
docker image prune -a

# ลบ unused volumes
docker volume prune

# ลบ unused networks
docker network prune
```

#### 5. Container ช้าหรือค้าง

**อาการ:**
Container ทำงานช้าหรือค้าง

**วิธีแก้:**
```bash
# ดูการใช้ทรัพยากร
docker stats myapp

# จำกัด memory
docker update --memory 512m myapp

# จำกัด CPU
docker update --cpus 1 myapp

# ตรวจสอบ logs
docker logs myapp

# ดู processes
docker top myapp
```

#### 6. Network Connection Issues

**อาการ:**
Containers ไม่สามารถเชื่อมต่อกันได้

**วิธีแก้:**
```bash
# ตรวจสอบ network
docker network inspect mynetwork

# ดู IP ของ container
docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp

# ทดสอบ ping
docker exec myapp ping other-container

# ตรวจสอบว่า containers อยู่ใน network เดียวกัน
docker inspect myapp | grep NetworkMode

# เพิ่ม container เข้า network
docker network connect mynetwork myapp
```

#### 7. Permission Denied

**อาการ:**
```bash
Permission denied while trying to connect to the Docker daemon socket
```

**วิธีแก้:**
```bash
# เพิ่ม user เข้า docker group
sudo usermod -aG docker $USER
newgrp docker

# หรือใช้ sudo (ไม่แนะนำ)
sudo docker run nginx

# ตรวจสอบสิทธิ์ socket
ls -l /var/run/docker.sock
sudo chmod 666 /var/run/docker.sock  # ชั่วคราว
```

#### 8. Image Pull Fails

**อาการ:**
```bash
Error pulling image: connection timeout
```

**วิธีแก้:**
```bash
# ตรวจสอบ internet connection
ping docker.io

# ใช้ mirror อื่น
docker pull registry.hub.docker.com/library/nginx

# เพิ่ม timeout
export DOCKER_CLIENT_TIMEOUT=120
export COMPOSE_HTTP_TIMEOUT=120

# ใช้ proxy (ถ้ามี)
# ~/.docker/config.json
{
  "proxies": {
    "default": {
      "httpProxy": "http://proxy:port",
      "httpsProxy": "http://proxy:port"
    }
  }
}
```

#### 9. Build Context Too Large

**อาการ:**
```bash
Sending build context to Docker daemon  2.5GB
```

**วิธีแก้:**
```bash
# สร้างไฟล์ .dockerignore
cat > .dockerignore << EOF
node_modules
.git
*.log
dist
coverage
.env
EOF

# ตรวจสอบขนาด build context
du -sh .

# ใช้ specific COPY แทน COPY . .
COPY package*.json ./
COPY src ./src
```

#### 10. Container Keeps Restarting

**อาการ:**
Container restart loop

**วิธีแก้:**
```bash
# ดู logs
docker logs --tail 100 myapp

# ดูสาเหตุ
docker inspect myapp | grep -A 10 "State"

# หยุด restart policy ชั่วคราว
docker update --restart=no myapp

# รันแบบ interactive เพื่อ debug
docker run -it myapp sh

# ตรวจสอบ health check
docker inspect myapp | grep -A 20 "Health"
```

#### 11. DNS Resolution Problems

**อาการ:**
Cannot resolve hostnames inside container

**วิธีแก้:**
```bash
# ตรวจสอบ DNS
docker exec myapp cat /etc/resolv.conf

# กำหนด DNS server
docker run --dns 8.8.8.8 --dns 8.8.4.4 myapp

# ใน docker-compose.yml
services:
  app:
    dns:
      - 8.8.8.8
      - 8.8.4.4

# ตั้งค่าใน daemon.json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

#### 12. Volume Mount Issues

**อาการ:**
Files not showing in container

**วิธีแก้:**
```bash
# ตรวจสอบ mount
docker inspect -f '{{.Mounts}}' myapp

# ใช้ absolute path
docker run -v $(pwd)/data:/data myapp

# ตรวจสอบสิทธิ์
ls -la /path/to/host/directory

# บน Windows ต้องแชร์ drive ใน Docker Desktop Settings

# ตรวจสอบว่าไฟล์อยู่จริง
docker exec myapp ls -la /data
```

## Performance Optimization Tips

### 1. ลดขนาด Images

```dockerfile
# ใช้ Alpine base images
FROM node:18-alpine

# Multi-stage builds
FROM node:18 AS builder
...
FROM node:18-alpine
COPY --from=builder ...

# ลบ unnecessary files
RUN apt-get update && \
    apt-get install -y package && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 2. Cache Optimization

```dockerfile
# คัดลอก dependencies ก่อน
COPY package*.json ./
RUN npm ci

# คัดลอก source code ทีหลัง
COPY . .
```

### 3. Build Performance

```bash
# ใช้ BuildKit
export DOCKER_BUILDKIT=1

# Parallel builds
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .
```

### 4. Resource Limits

```bash
# จำกัด memory และ CPU
docker run -m 512m --cpus=0.5 myapp

# ใน docker-compose.yml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

### 5. Logging

```bash
# จำกัดขนาด log files
docker run --log-opt max-size=10m --log-opt max-file=3 myapp
```

## Best Practices Checklist

### Development
- [ ] ใช้ .dockerignore
- [ ] ใช้ bind mounts สำหรับ live reload
- [ ] ใช้ docker-compose.dev.yml
- [ ] เปิด debugger ports
- [ ] ใช้ environment variables

### Production
- [ ] ใช้ multi-stage builds
- [ ] รันด้วย non-root user
- [ ] ระบุ versions อย่างชัดเจน
- [ ] ตั้งค่า health checks
- [ ] จำกัด resources
- [ ] ตั้งค่า restart policy
- [ ] ใช้ secrets management
- [ ] เปิด logging
- [ ] Backup volumes
- [ ] Monitor containers

### Security
- [ ] Scan images for vulnerabilities
- [ ] ไม่เก็บ secrets ใน images
- [ ] ใช้ official images
- [ ] Update images เป็นประจำ
- [ ] ใช้ read-only filesystem
- [ ] จำกัด capabilities
- [ ] แยก networks
- [ ] ใช้ TLS/SSL

## แหล่งข้อมูลเพิ่มเติม

### Official Documentation
- Docker Documentation: https://docs.docker.com
- Docker Hub: https://hub.docker.com
- Docker Compose: https://docs.docker.com/compose

### Online Resources
- Docker Blog: https://www.docker.com/blog
- Play with Docker: https://labs.play-with-docker.com
- Docker Training: https://training.docker.com

### Communities
- Docker Community Forums: https://forums.docker.com
- Docker Slack: https://dockercommunity.slack.com
- Stack Overflow: https://stackoverflow.com/questions/tagged/docker

### Tools
- Docker Desktop: https://www.docker.com/products/docker-desktop
- Docker Scout: Security scanning
- Trivy: Vulnerability scanner
- Portainer: Container management UI
- Lazydocker: Terminal UI

### Books and Courses
- Docker Deep Dive by Nigel Poulton
- Docker in Action by Jeff Nickoloff
- Docker Mastery Course (Udemy)

## คำศัพท์

| ศัพท์ | คำแปล/คำอธิบาย |
|-------|----------------|
| Container | ตู้คอนเทนเนอร์ (โปรแกรมที่ทำงานแยกอิสระ) |
| Image | อิมเมจ (แม่แบบสำหรับสร้าง container) |
| Dockerfile | ไฟล์คำสั่งสร้าง image |
| Volume | โวลุ่ม (พื้นที่เก็บข้อมูลถาวร) |
| Network | เครือข่าย (สำหรับเชื่อมต่อ containers) |
| Registry | รีจิสทรี (ที่เก็บ images) |
| Repository | รีพอสิทอรี (คลัง images) |
| Tag | แท็ก (ป้ายระบุเวอร์ชัน) |
| Layer | เลเยอร์ (ชั้นของ image) |
| Build Context | บิลด์คอนเท็กซ์ (ไฟล์ที่ส่งไปยัง Docker daemon) |
| Bind Mount | ไบน์ด์เมานท์ (เชื่อมโฟลเดอร์จาก host) |
| Port Mapping | พอร์ตแมปปิ้ง (เชื่อม port ระหว่าง host และ container) |
| Orchestration | ออเคสเทรชัน (การจัดการ containers หลายตัว) |

## สรุป

ภาคผนวกนี้รวบรวม:

- ✅ คำสั่ง Docker ที่ใช้บ่อยทั้งหมด
- ✅ Troubleshooting guide สำหรับปัญหาที่พบบ่อย
- ✅ Performance optimization tips
- ✅ Best practices checklist
- ✅ แหล่งข้อมูลเพิ่มเติม

---

🎉 **ขอบคุณที่อ่านจนจบ!** 

หวังว่าหนังสือเล่มนี้จะช่วยให้คุณเข้าใจและใช้งาน Docker ได้อย่างมืออาชีพ

⬅️ [กลับไปบทที่ 8: ตัวอย่างการใช้งานจริง](../chapters/08-real-world-examples/README.md)

⬅️ [กลับไปหน้าแรก](../README.md)
