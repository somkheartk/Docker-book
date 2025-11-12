# บทที่ 7: หัวข้อขั้นสูง

## ภาพรวม

ในบทนี้เราจะเรียนรู้เทคนิคขั้นสูงที่จะช่วยให้คุณใช้งาน Docker ได้อย่างมืออาชีพ

## Multi-Stage Builds

### ปัญหาของ Single-Stage Build

```dockerfile
# ❌ ปัญหา: Image ขนาดใหญ่เกินไป
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install          # รวม devDependencies
COPY . .
RUN npm run build        # build tools ยังอยู่ใน image
CMD ["node", "dist/server.js"]

# ผลลัพธ์: Image ขนาด 1.2GB!
```

### Solution: Multi-Stage Build

```dockerfile
# ✅ แยก build stage และ runtime stage
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production
CMD ["node", "dist/server.js"]

# ผลลัพธ์: Image ขนาด 180MB!
```

### ตัวอย่าง Multi-Stage Builds

**ตัวอย่างที่ 1: Go Application**

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# Runtime stage
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /build/app .
EXPOSE 8080
CMD ["./app"]

# ลดขนาดจาก 800MB เหลือ 15MB!
```

**ตัวอย่างที่ 2: React Application**

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**ตัวอย่างที่ 3: Python Application**

```dockerfile
# Build stage
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

### การใช้ Multiple Build Stages

```dockerfile
# Testing stage
FROM node:18 AS tester
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm test

# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

```bash
# Build แบบปกติ (ผ่านทุก stage)
docker build -t myapp:latest .

# Build เฉพาะ stage
docker build --target tester -t myapp:test .
docker build --target builder -t myapp:builder .
```

## Docker Security Best Practices

### 1. ใช้ Non-Root User

```dockerfile
# ❌ ไม่ดี: รันด้วย root
FROM node:18
WORKDIR /app
COPY . .
CMD ["node", "server.js"]

# ✅ ดี: รันด้วย non-root user
FROM node:18-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "server.js"]
```

### 2. Scan Images สำหรับ Vulnerabilities

```bash
# ใช้ Docker Scout
docker scout cves myapp:latest

# ใช้ Trivy
docker run aquasec/trivy image myapp:latest

# ใช้ Snyk
snyk container test myapp:latest
```

### 3. ใช้ Official และ Verified Images

```dockerfile
# ✅ ดี: Official image
FROM node:18-alpine

# ✅ ดี: Verified publisher
FROM bitnami/postgresql:15

# ❌ ระวัง: Unknown source
FROM random-user/custom-image
```

### 4. ระบุ Version แทนการใช้ latest

```dockerfile
# ❌ ไม่ดี
FROM node:latest

# ✅ ดี
FROM node:18.16.0-alpine3.17
```

### 5. อย่าเก็บ Secrets ใน Images

```dockerfile
# ❌ อันตราย!
ENV API_KEY=sk-1234567890abcdef
ENV DB_PASSWORD=supersecret

# ✅ ใช้ environment variables เมื่อ run
# docker run -e API_KEY=$API_KEY myapp
```

**ใช้ Docker Secrets (Swarm) หรือ External Secret Management:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: myapp
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### 6. Minimize Attack Surface

```dockerfile
# ใช้ distroless หรือ minimal base images
FROM gcr.io/distroless/nodejs18
COPY --from=builder /app .
CMD ["server.js"]

# หรือใช้ Alpine
FROM alpine:latest
RUN apk add --no-cache nodejs
```

### 7. ใช้ Read-Only Filesystem

```bash
# รัน container แบบ read-only
docker run --read-only --tmpfs /tmp myapp
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

### 8. จำกัด Capabilities

```bash
# ลบ capabilities ที่ไม่จำเป็น
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

```yaml
# docker-compose.yml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

### 9. ตั้งค่า Security Options

```bash
# เปิดใช้ AppArmor/SELinux
docker run --security-opt apparmor=docker-default myapp

# ไม่อนุญาตให้ escalate privileges
docker run --security-opt no-new-privileges:true myapp
```

### 10. Network Security

```yaml
# แยก networks
services:
  frontend:
    networks:
      - public
      
  backend:
    networks:
      - private
      
  database:
    networks:
      - private

networks:
  public:
  private:
    internal: true  # ไม่สามารถเข้าถึงจาก internet
```

## Health Checks

### ใน Dockerfile

```dockerfile
FROM nginx:alpine

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

# หรือใช้ curl
HEALTHCHECK CMD curl -f http://localhost/ || exit 1
```

### ใน Docker Compose

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: .
    depends_on:
      db:
        condition: service_healthy  # รอจน db healthy
```

### Custom Health Check Script

```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Health check script
COPY healthcheck.js ./
RUN npm install -g wait-on

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s \
  CMD node healthcheck.js

CMD ["node", "server.js"]
```

```javascript
// healthcheck.js
const http = require('http');

const options = {
  host: 'localhost',
  port: 3000,
  path: '/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', () => {
  process.exit(1);
});

request.end();
```

## Docker BuildKit

BuildKit เป็น build engine รุ่นใหม่ที่เร็วและมีความสามารถมากกว่า

### เปิดใช้งาน BuildKit

```bash
# Linux/Mac
export DOCKER_BUILDKIT=1
docker build .

# Windows PowerShell
$env:DOCKER_BUILDKIT=1
docker build .

# หรือตั้งค่าถาวรใน daemon.json
# /etc/docker/daemon.json
{
  "features": {
    "buildkit": true
  }
}
```

### ความสามารถของ BuildKit

**1. Cache Mounts**

```dockerfile
# แชร์ cache ระหว่าง builds
FROM node:18
WORKDIR /app

COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

COPY . .
CMD ["node", "server.js"]
```

**2. Secret Mounts**

```dockerfile
# ใช้ secrets โดยไม่เก็บใน image
FROM alpine
RUN --mount=type=secret,id=mysecret \
    cat /run/secrets/mysecret
```

```bash
# Build พร้อม secret
docker build --secret id=mysecret,src=./secret.txt .
```

**3. SSH Mounts**

```dockerfile
# Clone private git repos
FROM alpine
RUN apk add git openssh-client
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git
```

```bash
# Build พร้อม SSH
docker build --ssh default .
```

**4. Parallel Builds**

BuildKit build หลาย stages พร้อมกันอัตโนมัติ

## Optimization Techniques

### 1. Layer Caching

```dockerfile
# ✅ ดี: แยก layers ตามความถี่ของการเปลี่ยนแปลง
FROM node:18
WORKDIR /app

# Layer 1: Dependencies (เปลี่ยนน้อย)
COPY package*.json ./
RUN npm ci

# Layer 2: Source code (เปลี่ยนบ่อย)
COPY . .

CMD ["node", "server.js"]
```

### 2. ลดขนาด Images

```dockerfile
# ใช้ Alpine variants
FROM node:18-alpine          # แทน node:18

# ใช้ distroless
FROM gcr.io/distroless/nodejs18

# รวมคำสั่ง RUN
RUN apt-get update && \
    apt-get install -y package1 package2 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ลบไฟล์ที่ไม่จำเป็น
RUN npm ci --only=production && \
    npm cache clean --force
```

### 3. ใช้ .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
README.md
*.md
.vscode
.idea
dist
coverage
.DS_Store
*.log
```

### 4. Multithreading ใน Build

```bash
# ใช้ BuildKit สำหรับ parallel execution
DOCKER_BUILDKIT=1 docker build .
```

### 5. Use Specific COPY

```dockerfile
# ❌ ไม่ดี: คัดลอกทุกอย่าง
COPY . .

# ✅ ดี: คัดลอกเฉพาะที่จำเป็น
COPY package*.json ./
COPY src ./src
COPY public ./public
```

## Container Resource Management

### จำกัด CPU

```bash
# จำกัด CPU
docker run --cpus="1.5" myapp           # 1.5 cores
docker run --cpu-shares=512 myapp        # relative weight
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.5'
        reservations:
          cpus: '0.25'
```

### จำกัด Memory

```bash
# จำกัด memory
docker run -m 512m myapp                 # max 512MB
docker run -m 512m --memory-swap 1g myapp  # + swap
```

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

### จำกัด I/O

```bash
# จำกัด disk I/O
docker run --device-write-bps /dev/sda:1mb myapp
docker run --device-read-bps /dev/sda:1mb myapp
```

## Logging Best Practices

### 1. ใช้ Logging Drivers

```bash
# JSON file (default)
docker run --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Syslog
docker run --log-driver syslog \
  --log-opt syslog-address=tcp://192.168.0.42:123 \
  myapp
```

### 2. Centralized Logging

```yaml
# docker-compose.yml with logging
version: '3.8'

services:
  app:
    image: myapp
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: myapp

  fluentd:
    image: fluent/fluentd
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd/conf:/fluentd/etc
```

### 3. Structured Logging

```javascript
// ใช้ structured logging ใน application
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [
    new winston.transports.Console()
  ]
});

logger.info('User logged in', {
  userId: 123,
  ip: '192.168.1.1',
  timestamp: new Date()
});
```

## Monitoring และ Observability

### 1. Container Stats

```bash
# ดูการใช้ทรัพยากร
docker stats

# ดูเฉพาะบาง containers
docker stats web db cache
```

### 2. Docker Events

```bash
# ดู events real-time
docker events

# กรอง events
docker events --filter 'type=container'
docker events --filter 'event=start'
```

### 3. Prometheus + Grafana

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

volumes:
  prometheus-data:
  grafana-data:
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Multi-Stage Build

สร้าง Dockerfile สำหรับ Go application ด้วย multi-stage build:

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY . .
RUN go mod download
RUN CGO_ENABLED=0 go build -o app .

# Runtime stage
FROM scratch
COPY --from=builder /build/app /app
ENTRYPOINT ["/app"]
```

### แบบฝึกหัดที่ 2: Security Hardening

ปรับปรุง Dockerfile ให้ปลอดภัยยิ่งขึ้น:

```dockerfile
FROM node:18-alpine

# สร้าง non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# คัดลอกและติดตั้ง dependencies
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# คัดลอก application
COPY --chown=appuser:appgroup . .

# เปลี่ยนเป็น non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

EXPOSE 3000
CMD ["node", "server.js"]
```

### แบบฝึกหัดที่ 3: Monitoring Stack

สร้าง monitoring stack พร้อม Prometheus และ Grafana

## สรุป

ในบทนี้เราได้เรียนรู้:

- ✅ Multi-Stage Builds สำหรับลดขนาด images
- ✅ Security Best Practices
- ✅ Health Checks
- ✅ Docker BuildKit
- ✅ Optimization Techniques
- ✅ Resource Management
- ✅ Logging และ Monitoring

## ถัดไป

ในบทต่อไปเราจะดูตัวอย่างการใช้งานจริงในหลายๆ สถานการณ์

➡️ [บทที่ 8: ตัวอย่างการใช้งานจริง](../08-real-world-examples/README.md)

⬅️ [บทที่ 6: Docker Compose](../06-docker-compose/README.md)
