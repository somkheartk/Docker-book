# บทที่ 1: รู้จักกับ Docker

## Docker คืออะไร

Docker เป็นแพลตฟอร์มสำหรับการพัฒนา ส่งมอบ และรันแอปพลิเคชันในรูปแบบของ Container ซึ่งช่วยให้นักพัฒนาสามารถแพ็กเกจแอปพลิเคชันพร้อมกับ dependencies ทั้งหมดไว้ในหน่วยที่เรียกว่า Container

### ประวัติความเป็นมา

- Docker เปิดตัวในปี 2013 โดย Solomon Hykes
- เป็น Open Source Project
- พัฒนาด้วยภาษา Go
- ปัจจุบันเป็นมาตรฐานในการทำ Containerization

## ทำไมต้องใช้ Docker

### ปัญหาที่ Docker แก้ไข

**1. "It works on my machine" Problem**
```
นักพัฒนา: "โค้ดทำงานได้บนเครื่องผม"
DevOps: "แต่มันไม่ทำงานบน Production"
```

Docker แก้ปัญหานี้โดยการสร้างสภาพแวดล้อมที่เหมือนกันทุกที่

**2. การตั้งค่าที่ซับซ้อน**
- ไม่ต้องติดตั้ง dependencies หลายๆ อย่างบนเครื่อง
- สภาพแวดล้อมที่สอดคล้องกันระหว่าง Development, Testing, และ Production

**3. การใช้ทรัพยากรอย่างมีประสิทธิภาพ**
- เบากว่า Virtual Machines
- เริ่มต้นและหยุดทำงานได้เร็ว
- ใช้ระบบปฏิบัติการร่วมกัน

### ข้อดีของ Docker

1. **Portability**: รันได้ทุกที่ที่มี Docker
2. **Isolation**: แยกแอปพลิเคชันออกจากกัน
3. **Lightweight**: ใช้ทรัพยากรน้อยกว่า VM
4. **Fast**: สตาร์ทได้ภายในวินาที
5. **Scalability**: ขยายและลดขนาดได้ง่าย
6. **Version Control**: จัดการ images ด้วย tags

## Docker vs Virtual Machines

### Virtual Machines (VMs)

```
┌─────────────────────────────────────┐
│         Application 1               │
├─────────────────────────────────────┤
│         Guest OS (Linux)            │
├─────────────────────────────────────┤
│         Application 2               │
├─────────────────────────────────────┤
│         Guest OS (Windows)          │
├─────────────────────────────────────┤
│          Hypervisor                 │
├─────────────────────────────────────┤
│          Host OS                    │
├─────────────────────────────────────┤
│          Hardware                   │
└─────────────────────────────────────┘
```

### Docker Containers

```
┌─────────────────────────────────────┐
│  App1  │  App2  │  App3  │  App4   │
├────────┴────────┴────────┴──────────┤
│      Docker Engine                  │
├─────────────────────────────────────┤
│          Host OS                    │
├─────────────────────────────────────┤
│          Hardware                   │
└─────────────────────────────────────┘
```

### เปรียบเทียบ

| คุณสมบัติ | Virtual Machines | Docker Containers |
|-----------|------------------|-------------------|
| ขนาด | GB (Gigabytes) | MB (Megabytes) |
| เวลาในการเริ่มต้น | นาที | วินาที |
| การใช้ Memory | มาก | น้อย |
| Isolation | Complete | Process-level |
| OS | แยก OS แต่ละ VM | ใช้ Host OS ร่วมกัน |
| Performance | ช้ากว่า | เร็วกว่า |

## สถาปัตยกรรมของ Docker

Docker ประกอบด้วยส่วนหลักดังนี้:

### 1. Docker Client

- Interface สำหรับผู้ใช้
- รับคำสั่งจากผู้ใช้ (docker run, docker build)
- ส่งคำสั่งไปยัง Docker Daemon

### 2. Docker Daemon (dockerd)

- ทำงานบน Host machine
- จัดการ Docker objects (images, containers, networks, volumes)
- ติดต่อกับ Docker Daemons อื่นๆ

### 3. Docker Registry

- เก็บ Docker images
- Docker Hub เป็น public registry
- สามารถสร้าง private registry ได้

### โครงสร้างการทำงาน

```
┌──────────────┐         ┌──────────────┐
│ Docker CLI   │────────▶│ Docker       │
│ (client)     │         │ Daemon       │
└──────────────┘         └──────┬───────┘
                                │
                    ┌───────────┼───────────┐
                    │           │           │
                    ▼           ▼           ▼
              ┌──────────┐ ┌────────┐ ┌────────┐
              │ Images   │ │Contain │ │Network │
              │          │ │ ers    │ │Volumes │
              └──────────┘ └────────┘ └────────┘
                    │
                    ▼
              ┌──────────┐
              │ Docker   │
              │ Registry │
              │(Hub)     │
              └──────────┘
```

## คำศัพท์และแนวคิดพื้นฐาน

### 1. Image

- Template แบบ read-only สำหรับสร้าง Container
- ประกอบด้วย OS, Application code, และ Dependencies
- สร้างจาก Dockerfile
- เก็บไว้ใน Registry

**ตัวอย่าง**: `nginx:latest`, `node:18-alpine`, `python:3.11`

### 2. Container

- Instance ของ Image ที่กำลังทำงาน
- Isolated process บน Host machine
- สามารถ start, stop, delete ได้
- มีระบบไฟล์ของตัวเอง

**คิดง่ายๆ**: 
- Image = Class
- Container = Object (Instance)

### 3. Dockerfile

- ไฟล์ text ที่บอกวิธีการสร้าง Image
- มีคำสั่งทีละขั้นตอน
- Version control ได้

### 4. Docker Hub

- Public registry สำหรับ Docker images
- มี official images จากหลายๆ องค์กร
- สามารถ push/pull images

### 5. Volume

- กลไกในการเก็บข้อมูลถาวร (persistent data)
- ข้อมูลจะไม่หายเมื่อ Container ถูกลบ

### 6. Network

- ช่วยให้ Containers สื่อสารกันได้
- มีหลายประเภท: bridge, host, overlay

## ตัวอย่างการใช้งานพื้นฐาน

### ตัวอย่างที่ 1: Hello World

```bash
docker run hello-world
```

**สิ่งที่เกิดขึ้น:**
1. Docker client ติดต่อ Docker daemon
2. Docker daemon ดึง image "hello-world" จาก Docker Hub
3. Docker daemon สร้าง container จาก image
4. Docker daemon รัน container
5. Container แสดงข้อความและจบการทำงาน

### ตัวอย่างที่ 2: รัน Web Server

```bash
docker run -d -p 8080:80 nginx
```

**คำอธิบาย:**
- `-d`: รันในโหมด detached (background)
- `-p 8080:80`: map port 8080 (host) ไปยัง port 80 (container)
- `nginx`: ชื่อ image

เปิดเบราว์เซอร์ไปที่ `http://localhost:8080` จะเห็น Nginx welcome page

### ตัวอย่างที่ 3: รัน Container แบบ Interactive

```bash
docker run -it ubuntu bash
```

**คำอธิบาย:**
- `-i`: Interactive mode
- `-t`: Allocate pseudo-TTY
- `ubuntu`: image ชื่อ ubuntu
- `bash`: คำสั่งที่จะรันใน container

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ทดสอบติดตั้ง Docker

ตรวจสอบว่า Docker ติดตั้งสำเร็จแล้ว:

```bash
# เช็คเวอร์ชัน
docker --version

# เช็คข้อมูลระบบ
docker info

# รัน hello-world
docker run hello-world
```

### แบบฝึกหัดที่ 2: ทดลองรัน Container พื้นฐาน

ลองรัน container ต่างๆ:

```bash
# รัน nginx
docker run -d -p 8080:80 nginx

# รัน Apache
docker run -d -p 8081:80 httpd

# รัน Python interactive
docker run -it python:3.11
```

### แบบฝึกหัดที่ 3: ดูสถานะ Container

```bash
# ดู container ที่กำลังรัน
docker ps

# ดู container ทั้งหมด (รวมที่หยุดแล้ว)
docker ps -a

# ดู images
docker images
```

## สรุป

ในบทนี้เราได้เรียนรู้:

- ✅ Docker คืออะไรและทำไมต้องใช้
- ✅ ความแตกต่างระหว่าง Docker กับ Virtual Machines
- ✅ สถาปัตยกรรมและส่วนประกอบของ Docker
- ✅ คำศัพท์พื้นฐาน: Image, Container, Dockerfile, Volume, Network
- ✅ ตัวอย่างการใช้งานเบื้องต้น

## ถัดไป

ในบทต่อไปเราจะเรียนรู้วิธีการติดตั้ง Docker บนระบบปฏิบัติการต่างๆ

➡️ [บทที่ 2: การติดตั้ง Docker](../02-installation/README.md)
