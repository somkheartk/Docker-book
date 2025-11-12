# บทที่ 2: การติดตั้ง Docker

## ภาพรวม

ในบทนี้เราจะเรียนรู้วิธีการติดตั้ง Docker บนระบบปฏิบัติการต่างๆ รวมถึงการตั้งค่าเบื้องต้นและการตรวจสอบว่าการติดตั้งสำเร็จ

## ข้อกำหนดของระบบ

### Windows
- Windows 10 64-bit: Pro, Enterprise, หรือ Education (Build 16299 หรือใหม่กว่า)
- Windows 11 64-bit
- WSL 2 feature enabled
- 4GB RAM ขั้นต่ำ (แนะนำ 8GB)

### macOS
- macOS 10.15 หรือใหม่กว่า
- Mac hardware ต้องเป็น 2010 หรือใหม่กว่า
- 4GB RAM ขั้นต่ำ

### Linux
- 64-bit version ของ Linux
- Kernel version 3.10 หรือใหม่กว่า
- สนับสนุน: Ubuntu, Debian, CentOS, Fedora, RHEL

## ติดตั้ง Docker บน Windows

### วิธีที่ 1: Docker Desktop (แนะนำ)

**ขั้นตอนที่ 1: เปิดใช้งาน WSL 2**

1. เปิด PowerShell ด้วยสิทธิ์ Administrator และรันคำสั่ง:

```powershell
# เปิดใช้งาน WSL
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# เปิดใช้งาน Virtual Machine Platform
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# รีสตาร์ทเครื่อง
Restart-Computer
```

2. ติดตั้ง WSL 2 Linux kernel update package จาก [Microsoft](https://aka.ms/wsl2kernel)

3. ตั้งค่า WSL 2 เป็น default:

```powershell
wsl --set-default-version 2
```

**ขั้นตอนที่ 2: ดาวน์โหลดและติดตั้ง Docker Desktop**

1. ดาวน์โหลด Docker Desktop จาก https://www.docker.com/products/docker-desktop
2. รันไฟล์ `Docker Desktop Installer.exe`
3. ทำตามขั้นตอนการติดตั้ง
4. รีสตาร์ทเครื่องถ้าจำเป็น
5. เปิด Docker Desktop

**ขั้นตอนที่ 3: ตรวจสอบการติดตั้ง**

เปิด PowerShell หรือ Command Prompt และรันคำสั่ง:

```powershell
docker --version
docker run hello-world
```

### การตั้งค่าเพิ่มเติมสำหรับ Windows

**ตั้งค่า Resources:**

1. เปิด Docker Desktop
2. ไปที่ Settings → Resources
3. ปรับ Memory และ CPU ตามต้องการ
4. คลิก "Apply & Restart"

**ตั้งค่า WSL Integration:**

1. Settings → Resources → WSL Integration
2. เลือก Linux distributions ที่ต้องการใช้
3. คลิก "Apply & Restart"

## ติดตั้ง Docker บน macOS

### Docker Desktop for Mac

**ขั้นตอนที่ 1: ดาวน์โหลดและติดตั้ง**

1. ดาวน์โหลด Docker Desktop จาก https://www.docker.com/products/docker-desktop
   - สำหรับ Mac with Intel chip: Docker Desktop for Mac (Intel)
   - สำหรับ Mac with Apple Silicon (M1/M2): Docker Desktop for Mac (Apple Silicon)

2. เปิดไฟล์ `.dmg` ที่ดาวน์โหลดมา
3. ลาก Docker icon ไปยังโฟลเดอร์ Applications
4. เปิด Docker จากโฟลเดอร์ Applications
5. คลิก "Open" ถ้ามีการเตือนความปลอดภัย

**ขั้นตอนที่ 2: การอนุญาตสิทธิ์**

1. Docker จะขออนุญาตสิทธิ์ privileged access
2. ใส่รหัสผ่าน macOS และคลิก "OK"
3. รอจน Docker เริ่มต้นเสร็จ (ดูที่ Menu bar)

**ขั้นตอนที่ 3: ตรวจสอบการติดตั้ง**

เปิด Terminal และรันคำสั่ง:

```bash
docker --version
docker run hello-world
```

### การตั้งค่าเพิ่มเติมสำหรับ macOS

**ตั้งค่า Resources:**

1. คลิกที่ Docker icon ใน Menu bar
2. เลือก Preferences → Resources
3. ปรับ CPUs, Memory, Swap, และ Disk ตามต้องการ
4. คลิก "Apply & Restart"

**File Sharing:**

1. Preferences → Resources → File Sharing
2. เพิ่มโฟลเดอร์ที่ต้องการให้ Docker เข้าถึงได้
3. คลิก "Apply & Restart"

## ติดตั้ง Docker บน Linux

### Ubuntu

**วิธีที่ 1: ติดตั้งจาก Official Repository (แนะนำ)**

```bash
# อัปเดตระบบ
sudo apt-get update

# ติดตั้ง packages ที่จำเป็น
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# เพิ่ม Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# เพิ่ม repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ติดตั้ง Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# ตรวจสอบการติดตั้ง
sudo docker run hello-world
```

**วิธีที่ 2: ติดตั้งด้วย Convenience Script**

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### CentOS / RHEL

```bash
# ติดตั้ง yum-utils
sudo yum install -y yum-utils

# เพิ่ม repository
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# ติดตั้ง Docker Engine
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# เริ่มต้น Docker
sudo systemctl start docker
sudo systemctl enable docker

# ตรวจสอบการติดตั้ง
sudo docker run hello-world
```

### Debian

```bash
# อัปเดตระบบ
sudo apt-get update

# ติดตั้ง packages ที่จำเป็น
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# เพิ่ม Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# เพิ่ม repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ติดตั้ง Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# ตรวจสอบการติดตั้ง
sudo docker run hello-world
```

### การตั้งค่าเพิ่มเติมสำหรับ Linux

**ให้ user ปกติใช้ Docker โดยไม่ต้อง sudo:**

```bash
# สร้าง docker group (ถ้ายังไม่มี)
sudo groupadd docker

# เพิ่ม user ปัจจุบันเข้า docker group
sudo usermod -aG docker $USER

# ล็อกเอ้าท์และเข้าระบบใหม่ หรือรันคำสั่ง
newgrp docker

# ทดสอบ
docker run hello-world
```

**ตั้งค่าให้ Docker เริ่มต้นอัตโนมัติ:**

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## Docker Desktop vs Docker Engine

### Docker Desktop

**คุณสมบัติ:**
- GUI สำหรับจัดการ Containers, Images, Volumes
- รวม Docker Engine, Docker CLI, Docker Compose
- แนะนำสำหรับ Development
- มีทั้งแบบฟรีและแบบเสียเงิน

**เหมาะกับ:**
- นักพัฒนาที่ต้องการ GUI
- การพัฒนาบน Windows/macOS
- ผู้ที่เริ่มต้นเรียนรู้ Docker

### Docker Engine

**คุณสมบัติ:**
- Command-line เท่านั้น
- เบาและเร็วกว่า
- แนะนำสำหรับ Production servers
- ฟรีและ Open Source

**เหมาะกับ:**
- Production servers
- CI/CD pipelines
- Automated deployments
- ผู้ที่ชำนาญ Command-line

## การตรวจสอบและยืนยันการติดตั้ง

### ตรวจสอบเวอร์ชัน

```bash
# Docker version
docker --version
# Output: Docker version 24.0.x, build xxxxx

# Docker Compose version
docker compose version
# Output: Docker Compose version v2.x.x

# ข้อมูลโดยละเอียด
docker version
```

### ตรวจสอบข้อมูลระบบ

```bash
docker info
```

**ข้อมูลที่ควรตรวจสอบ:**
- Server version
- Storage Driver
- Cgroup Driver
- Cgroup Version
- Plugins (Volume, Network, Log)
- Kernel Version
- Operating System
- Architecture

### ทดสอบการทำงาน

**Test 1: Hello World**
```bash
docker run hello-world
```

**Test 2: รัน Ubuntu container**
```bash
docker run -it ubuntu bash
# พิมพ์คำสั่งใน container
ls
exit
```

**Test 3: รัน Web Server**
```bash
docker run -d -p 8080:80 nginx
# เปิดเบราว์เซอร์ไปที่ http://localhost:8080
# หยุด container
docker ps
docker stop <container_id>
```

## การแก้ปัญหาที่พบบ่อย

### Windows

**ปัญหา: "WSL 2 installation is incomplete"**

แก้ไข:
```powershell
# ติดตั้ง WSL 2 kernel update
# ดาวน์โหลดจาก https://aka.ms/wsl2kernel
```

**ปัญหา: "Hardware assisted virtualization"**

แก้ไข:
- เข้า BIOS และเปิดใช้งาน Virtualization Technology (VT-x/AMD-V)

### macOS

**ปัญหา: "Docker Desktop starting" นานเกินไป**

แก้ไข:
```bash
# รีเซ็ต Docker Desktop
# Troubleshoot → Reset to factory defaults
```

### Linux

**ปัญหา: "permission denied" เมื่อรัน docker**

แก้ไข:
```bash
# เพิ่ม user เข้า docker group
sudo usermod -aG docker $USER
newgrp docker
```

**ปัญหา: "Cannot connect to the Docker daemon"**

แก้ไข:
```bash
# เช็คสถานะ Docker service
sudo systemctl status docker

# เริ่มต้น Docker service
sudo systemctl start docker
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ติดตั้งและตรวจสอบ

1. ติดตั้ง Docker บนเครื่องของคุณ
2. รันคำสั่งเหล่านี้และบันทึกผลลัพธ์:
```bash
docker --version
docker info
docker run hello-world
```

### แบบฝึกหัดที่ 2: ทดสอบ Container พื้นฐาน

รัน containers ต่อไปนี้:

```bash
# 1. Ubuntu
docker run -it ubuntu bash

# 2. Alpine Linux (เบามาก)
docker run -it alpine sh

# 3. Python
docker run -it python:3.11 python

# 4. Node.js
docker run -it node:18 node
```

### แบบฝึกหัดที่ 3: การจัดการ Container

```bash
# รัน nginx ในโหมด detached
docker run -d --name my-nginx -p 8080:80 nginx

# ดูสถานะ
docker ps

# เช็ค logs
docker logs my-nginx

# หยุด container
docker stop my-nginx

# ลบ container
docker rm my-nginx
```

## สรุป

ในบทนี้เราได้เรียนรู้:

- ✅ ข้อกำหนดของระบบสำหรับแต่ละ OS
- ✅ วิธีติดตั้ง Docker บน Windows, macOS, และ Linux
- ✅ ความแตกต่างระหว่าง Docker Desktop และ Docker Engine
- ✅ การตรวจสอบและยืนยันการติดตั้ง
- ✅ การแก้ปัญหาที่พบบ่อย

## ถัดไป

ตอนนี้คุณติดตั้ง Docker สำเร็จแล้ว ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ Docker Images และ Containers อย่างละเอียด

➡️ [บทที่ 3: Docker Images และ Containers](../03-images-containers/README.md)

⬅️ [บทที่ 1: รู้จักกับ Docker](../01-introduction/README.md)
