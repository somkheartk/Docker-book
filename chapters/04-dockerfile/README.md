# บทที่ 4: การสร้าง Docker Images ด้วย Dockerfile

## ภาพรวม

ในบทนี้เราจะเรียนรู้วิธีการสร้าง Docker Images ของเราเองโดยใช้ Dockerfile ซึ่งเป็นไฟล์ที่กำหนดขั้นตอนการสร้าง image

## Dockerfile คืออะไร

Dockerfile เป็นไฟล์ text ที่ประกอบด้วยคำสั่ง (instructions) สำหรับสร้าง Docker image โดยอัตโนมัติ

**คุณสมบัติ:**
- เป็น text file ชื่อ `Dockerfile` (ไม่มีนามสกุล)
- มีคำสั่งเฉพาะที่ Docker เข้าใจ
- รันคำสั่งทีละบรรทัดจากบนลงล่าง
- แต่ละคำสั่งสร้าง layer ใหม่

## โครงสร้างพื้นฐาน

```dockerfile
# คำอธิบาย (comment)
FROM base_image:tag
WORKDIR /app
COPY source destination
RUN command
EXPOSE port
CMD ["executable", "param1", "param2"]
```

## คำสั่งพื้นฐานใน Dockerfile

### 1. FROM - กำหนด Base Image

```dockerfile
# ใช้ official image
FROM node:18

# ใช้ Alpine variant (เบา)
FROM node:18-alpine

# ใช้ specific version
FROM python:3.11.4-slim

# Multi-stage build
FROM node:18 AS builder
```

**Best Practice:**
- ใช้ official images เมื่อเป็นไปได้
- ระบุ version แทนการใช้ `latest`
- พิจารณาใช้ Alpine variants เพื่อลดขนาด

### 2. WORKDIR - กำหนดไดเรกทอรีทำงาน

```dockerfile
# กำหนด working directory
WORKDIR /app

# คำสั่งต่อไปจะทำงานใน /app
COPY . .
RUN npm install
```

### 3. COPY - คัดลอกไฟล์

```dockerfile
# คัดลอกไฟล์เดียว
COPY package.json /app/

# คัดลอกหลายไฟล์
COPY package.json package-lock.json /app/

# คัดลอกทุกอย่างใน directory
COPY . /app/

# คัดลอกโดยเปลี่ยนเจ้าของ
COPY --chown=node:node . /app/
```

### 4. ADD - คัดลอกไฟล์ (แบบขยาย)

```dockerfile
# เหมือน COPY
ADD package.json /app/

# สามารถ extract tar files
ADD myapp.tar.gz /app/

# สามารถดาวน์โหลดจาก URL (ไม่แนะนำ)
ADD https://example.com/file.txt /app/
```

**Best Practice:** ใช้ `COPY` แทน `ADD` เว้นแต่ต้องการความสามารถพิเศษ

### 5. RUN - รันคำสั่งขณะ Build

```dockerfile
# รูปแบบ shell
RUN apt-get update && apt-get install -y curl

# รูปแบบ exec
RUN ["npm", "install"]

# หลายคำสั่งในบรรทัดเดียว (ลด layers)
RUN apt-get update && \
    apt-get install -y \
        curl \
        vim \
        git && \
    rm -rf /var/lib/apt/lists/*
```

### 6. CMD - คำสั่งเริ่มต้นเมื่อรัน Container

```dockerfile
# รูปแบบ exec (แนะนำ)
CMD ["node", "server.js"]

# รูปแบบ shell
CMD node server.js

# ใช้กับ ENTRYPOINT
CMD ["--port", "3000"]
```

**หมายเหตุ:** มีได้เพียง CMD เดียวใน Dockerfile

### 7. ENTRYPOINT - กำหนด executable หลัก

```dockerfile
# รูปแบบ exec
ENTRYPOINT ["python", "app.py"]

# ใช้ร่วมกับ CMD
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]

# เมื่อรัน: python app.py --port 8000
# สามารถ override CMD: docker run myapp --port 9000
```

### 8. EXPOSE - ระบุ Port ที่เปิดใช้

```dockerfile
# เปิด port เดียว
EXPOSE 80

# เปิดหลาย ports
EXPOSE 80 443

# ระบุ protocol
EXPOSE 8080/tcp
EXPOSE 8080/udp
```

**หมายเหตุ:** EXPOSE เป็นเพียง documentation ต้องใช้ `-p` เมื่อ run

### 9. ENV - กำหนด Environment Variables

```dockerfile
# กำหนดตัวแปรเดียว
ENV NODE_ENV production

# กำหนดหลายตัวแปร
ENV APP_HOME /app \
    APP_PORT 3000 \
    APP_DEBUG false
```

### 10. ARG - กำหนดตัวแปรสำหรับ Build

```dockerfile
# กำหนด build argument
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}

ARG APP_PORT=3000
EXPOSE ${APP_PORT}

# ใช้เมื่อ build
# docker build --build-arg NODE_VERSION=20 .
```

### 11. VOLUME - กำหนด Mount Point

```dockerfile
# สร้าง volume mount point
VOLUME /data

# หลาย volumes
VOLUME /data /logs
```

### 12. USER - เปลี่ยน User

```dockerfile
# สร้าง user ใหม่
RUN adduser --disabled-password --gecos '' appuser

# เปลี่ยนเป็น user นั้น
USER appuser

# คำสั่งต่อไปจะรันด้วย user นี้
WORKDIR /app
CMD ["node", "server.js"]
```

### 13. LABEL - เพิ่ม Metadata

```dockerfile
LABEL maintainer="your.email@example.com"
LABEL version="1.0"
LABEL description="My application"
```

### 14. HEALTHCHECK - ตรวจสอบสุขภาพ Container

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

## ตัวอย่าง Dockerfiles

### ตัวอย่างที่ 1: Node.js Application

```dockerfile
# ใช้ official Node.js image
FROM node:18-alpine

# กำหนด working directory
WORKDIR /app

# คัดลอก package files
COPY package*.json ./

# ติดตั้ง dependencies
RUN npm ci --only=production

# คัดลอก source code
COPY . .

# เปิด port
EXPOSE 3000

# เริ่มต้น application
CMD ["node", "server.js"]
```

### ตัวอย่างที่ 2: Python Flask Application

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# คัดลอก requirements และติดตั้ง
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# คัดลอก application
COPY . .

# สร้าง non-root user
RUN useradd -m -u 1000 flaskuser && \
    chown -R flaskuser:flaskuser /app

USER flaskuser

EXPOSE 5000

ENV FLASK_APP=app.py
ENV FLASK_ENV=production

CMD ["flask", "run", "--host=0.0.0.0"]
```

### ตัวอย่างที่ 3: Go Application

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /build

# คัดลอก go mod files
COPY go.mod go.sum ./
RUN go mod download

# คัดลอกและ build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app .

# ขั้นตอนที่ 2: Runtime
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /build/app .

EXPOSE 8080

CMD ["./app"]
```

### ตัวอย่างที่ 4: Static Website (Nginx)

```dockerfile
FROM nginx:alpine

# ลบ default nginx config
RUN rm /etc/nginx/conf.d/default.conf

# คัดลอก custom config
COPY nginx.conf /etc/nginx/conf.d/

# คัดลอก static files
COPY ./dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### ตัวอย่างที่ 5: Multi-Stage Build

```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: Production
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## การ Build Images

### คำสั่งพื้นฐาน

```bash
# Build จาก Dockerfile ใน directory ปัจจุบัน
docker build .

# Build และตั้งชื่อ/tag
docker build -t myapp:latest .

# Build และตั้งหลาย tags
docker build -t myapp:latest -t myapp:v1.0 .

# ระบุ Dockerfile อื่น
docker build -f Dockerfile.dev -t myapp:dev .

# ส่ง build arguments
docker build --build-arg NODE_VERSION=20 -t myapp .

# ไม่ใช้ cache
docker build --no-cache -t myapp .

# Build และดู process ละเอียด
docker build -t myapp --progress=plain .
```

### .dockerignore File

สร้างไฟล์ `.dockerignore` เพื่อไม่ให้ส่งไฟล์ที่ไม่จำเป็นเข้า build context:

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.vscode
.idea
*.log
dist
coverage
.DS_Store
```

### การ Tag Images

```bash
# Tag ด้วยชื่อ repository
docker tag myapp:latest username/myapp:latest

# Tag ด้วย version
docker tag myapp:latest myapp:v1.0.0

# Tag สำหรับ registry อื่น
docker tag myapp:latest registry.example.com/myapp:latest
```

## Best Practices สำหรับการเขียน Dockerfile

### 1. ใช้ .dockerignore

```
# ลดขนาด build context
node_modules/
.git/
*.md
```

### 2. เรียงลำดับคำสั่งตามความถี่ของการเปลี่ยนแปลง

```dockerfile
# ✅ ดี: สิ่งที่เปลี่ยนน้อยไว้ข้างบน
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]

# ❌ ไม่ดี: COPY . . อยู่ก่อน npm install
FROM node:18-alpine
WORKDIR /app
COPY . .  # เปลี่ยนบ่อย = cache miss
RUN npm install
CMD ["node", "server.js"]
```

### 3. รวมคำสั่ง RUN เพื่อลด Layers

```dockerfile
# ✅ ดี: รวมคำสั่ง
RUN apt-get update && \
    apt-get install -y curl vim && \
    rm -rf /var/lib/apt/lists/*

# ❌ ไม่ดี: แยกคำสั่ง
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
```

### 4. ใช้ Multi-Stage Builds

```dockerfile
# ✅ ดี: แยก build และ runtime
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

### 5. ใช้ Non-Root User

```dockerfile
# ✅ ดี: รันด้วย non-root user
FROM node:18-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
CMD ["node", "server.js"]
```

### 6. ระบุ Versions อย่างชัดเจน

```dockerfile
# ✅ ดี
FROM node:18.16.0-alpine3.17

# ❌ ไม่ดี
FROM node:latest
```

### 7. ลบไฟล์ชั่วคราวในชั้นเดียวกัน

```dockerfile
# ✅ ดี
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

# ❌ ไม่ดี (ขยะยังอยู่ใน layer ก่อนหน้า)
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

### 8. ใช้ COPY แทน ADD

```dockerfile
# ✅ ดี
COPY package.json ./

# ❌ ไม่ดี (เว้นแต่ต้องการ auto-extract)
ADD package.json ./
```

## การ Push Images

### Push ไป Docker Hub

```bash
# 1. Login
docker login

# 2. Tag image
docker tag myapp:latest username/myapp:v1.0

# 3. Push
docker push username/myapp:v1.0

# Push all tags
docker push username/myapp --all-tags
```

### Push ไป Private Registry

```bash
# 1. Tag for private registry
docker tag myapp:latest registry.company.com/myapp:v1.0

# 2. Login to private registry
docker login registry.company.com

# 3. Push
docker push registry.company.com/myapp:v1.0
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Simple Node.js App

สร้าง Node.js application และ Dockerfile:

**1. สร้างไฟล์ `app.js`:**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from Docker!\n');
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

**2. สร้าง `package.json`:**
```json
{
  "name": "docker-exercise",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
```

**3. สร้าง `Dockerfile`:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json .
COPY app.js .
EXPOSE 3000
CMD ["npm", "start"]
```

**4. Build และ Run:**
```bash
docker build -t my-node-app .
docker run -p 3000:3000 my-node-app
```

### แบบฝึกหัดที่ 2: Python Flask App

**1. สร้าง `app.py`:**
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Flask in Docker!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**2. สร้าง `requirements.txt`:**
```
Flask==2.3.0
```

**3. สร้าง `Dockerfile`:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

**4. Build และ Run:**
```bash
docker build -t my-flask-app .
docker run -p 5000:5000 my-flask-app
```

### แบบฝึกหัดที่ 3: Multi-Stage Build

สร้าง Go application ด้วย multi-stage build:

**1. สร้าง `main.go`:**
```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello from Go in Docker!")
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

**2. สร้าง `Dockerfile`:**
```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY main.go .
RUN go build -o app main.go

# Runtime stage
FROM alpine:latest
WORKDIR /root/
COPY --from=builder /build/app .
EXPOSE 8080
CMD ["./app"]
```

**3. Build และ Run:**
```bash
docker build -t my-go-app .
docker run -p 8080:8080 my-go-app
```

## สรุป

ในบทนี้เราได้เรียนรู้:

- ✅ Dockerfile คืออะไรและโครงสร้างพื้นฐาน
- ✅ คำสั่งสำคัญใน Dockerfile: FROM, RUN, COPY, CMD, ENTRYPOINT, etc.
- ✅ ตัวอย่าง Dockerfiles สำหรับภาษาและ frameworks ต่างๆ
- ✅ การ build images และการใช้ tags
- ✅ Best Practices สำหรับการเขียน Dockerfile
- ✅ การ push images ไป registry

## ถัดไป

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ Docker Volumes และ Networking

➡️ [บทที่ 5: Docker Volumes และ Networking](../05-volumes-networking/README.md)

⬅️ [บทที่ 3: Docker Images และ Containers](../03-images-containers/README.md)
