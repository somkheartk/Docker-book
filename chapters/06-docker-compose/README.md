# บทที่ 6: Docker Compose

## ภาพรวม

Docker Compose เป็นเครื่องมือสำหรับการกำหนดและรัน multi-container Docker applications โดยใช้ไฟล์ YAML เพียงไฟล์เดียว

## Docker Compose คืออะไร

**ปัญหา:**
- รัน containers หลายๆ ตัวด้วยคำสั่งยาวๆ
- ต้องจำคำสั่ง docker run ที่ซับซ้อน
- ยากต่อการจัดการ dependencies ระหว่าง containers

**Solution: Docker Compose**
- กำหนดทุกอย่างในไฟล์ `docker-compose.yml`
- รันทุก container พร้อมกันด้วยคำสั่งเดียว
- จัดการ networks และ volumes อัตโนมัติ

### ก่อนใช้ Docker Compose

```bash
# สร้าง network
docker network create myapp

# รัน database
docker run -d \
  --name db \
  --network myapp \
  -e POSTGRES_PASSWORD=secret \
  -v db-data:/var/lib/postgresql/data \
  postgres:15

# รัน backend
docker run -d \
  --name backend \
  --network myapp \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
  -p 3000:3000 \
  backend:latest

# รัน frontend
docker run -d \
  --name frontend \
  --network myapp \
  -e API_URL=http://backend:3000 \
  -p 80:80 \
  frontend:latest
```

### หลังใช้ Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data

  backend:
    image: backend:latest
    environment:
      DATABASE_URL: postgresql://postgres:secret@db:5432/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db

  frontend:
    image: frontend:latest
    environment:
      API_URL: http://backend:3000
    ports:
      - "80:80"
    depends_on:
      - backend

volumes:
  db-data:
```

```bash
# รันทุกอย่างด้วยคำสั่งเดียว
docker compose up -d
```

## การติดตั้ง Docker Compose

### Docker Desktop
- มาพร้อมกับ Docker Desktop แล้ว

### Linux
```bash
# Docker Compose V2 (แนะนำ)
# มาพร้อมกับ Docker Engine แล้ว
docker compose version

# ถ้ายังไม่มี ติดตั้งด้วย
sudo apt-get install docker-compose-plugin
```

## โครงสร้างไฟล์ docker-compose.yml

### ตัวอย่างพื้นฐาน

```yaml
version: '3.8'  # เวอร์ชันของ Compose file format

services:       # กำหนด containers
  service1:
    image: nginx
    ports:
      - "80:80"

  service2:
    build: ./app
    environment:
      - NODE_ENV=production

networks:       # กำหนด networks (optional)
  default:
    driver: bridge

volumes:        # กำหนด volumes (optional)
  data:
```

## Services

### การใช้ Image

```yaml
services:
  web:
    image: nginx:latest          # ใช้ image จาก Docker Hub
    
  db:
    image: postgres:15-alpine    # ระบุ version และ variant
    
  custom:
    image: myregistry.com/myapp:v1.0  # ใช้จาก private registry
```

### การ Build Image

```yaml
services:
  app:
    build: .                     # Build จาก Dockerfile ใน current directory
    
  app2:
    build:
      context: ./app             # Directory ที่มี Dockerfile
      dockerfile: Dockerfile.dev # ระบุ Dockerfile อื่น
      args:                      # Build arguments
        NODE_VERSION: 18
        
  app3:
    build: ./app
    image: myapp:latest          # ตั้งชื่อ image ที่ build ได้
```

### Ports

```yaml
services:
  web:
    ports:
      - "8080:80"                # HOST:CONTAINER
      - "8443:443"
      
  app:
    ports:
      - "3000-3005:3000-3005"    # port range
      - "127.0.0.1:8000:8000"    # bind to specific IP
      
  internal:
    expose:                      # เปิดให้ services อื่นเท่านั้น
      - "3000"
```

### Environment Variables

```yaml
services:
  app:
    environment:                 # แบบ key-value
      NODE_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      
  app2:
    environment:                 # แบบ array
      - NODE_ENV=production
      - DEBUG=false
      
  app3:
    env_file:                    # จากไฟล์
      - .env
      - .env.production
```

### Volumes

```yaml
services:
  db:
    volumes:
      - db-data:/var/lib/postgresql/data    # named volume
      - ./data:/data                         # bind mount
      - ./config.yml:/app/config.yml:ro      # read-only
      
  app:
    volumes:
      - type: volume
        source: app-data
        target: /data
      - type: bind
        source: ./app
        target: /app

volumes:
  db-data:                       # กำหนด volume
  app-data:
    driver: local
```

### Networks

```yaml
services:
  web:
    networks:
      - frontend
      
  app:
    networks:
      - frontend
      - backend
      
  db:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true               # ไม่สามารถเข้าถึงจากภายนอก
```

### Depends On

```yaml
services:
  web:
    depends_on:
      - db                       # รอให้ db เริ่มก่อน
      
  app:
    depends_on:
      db:
        condition: service_healthy    # รอจนกว่า db จะ healthy
      redis:
        condition: service_started    # รอแค่ start
        
  db:
    healthcheck:                 # กำหนด health check
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Restart Policy

```yaml
services:
  app:
    restart: no                  # ไม่ restart (default)
    
  web:
    restart: always              # restart เสมอ
    
  worker:
    restart: on-failure          # restart เมื่อ fail
    
  cache:
    restart: unless-stopped      # restart เว้นแต่ถูก stop
```

## Docker Compose Commands

### คำสั่งพื้นฐาน

```bash
# เริ่มต้น services ทั้งหมด
docker compose up

# เริ่มต้นในโหมด detached
docker compose up -d

# Build images ก่อน
docker compose up --build

# เริ่มเฉพาะบาง services
docker compose up web db

# หยุด services
docker compose stop

# หยุดและลบ containers, networks
docker compose down

# ลบพร้อม volumes
docker compose down -v

# ลบพร้อม images
docker compose down --rmi all
```

### การดูสถานะ

```bash
# ดู services ที่กำลังรัน
docker compose ps

# ดู logs
docker compose logs

# ดู logs แบบ follow
docker compose logs -f

# ดู logs เฉพาะ service
docker compose logs -f web

# ดูสถานะ real-time
docker compose top
```

### การจัดการ Services

```bash
# เริ่มต้น service
docker compose start web

# หยุด service
docker compose stop web

# Restart service
docker compose restart web

# Pause/Unpause
docker compose pause web
docker compose unpause web

# รัน command ใน service
docker compose exec web bash
docker compose exec db psql -U postgres

# รัน one-off command
docker compose run web npm test
```

### การจัดการอื่นๆ

```bash
# ดูการตั้งค่า
docker compose config

# ดู images
docker compose images

# Build หรือ rebuild images
docker compose build

# Pull images
docker compose pull

# Push images
docker compose push
```

## ตัวอย่างการใช้งานจริง

### ตัวอย่างที่ 1: WordPress + MySQL

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: mysql:8
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    restart: always

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    restart: always

volumes:
  db_data:
  wordpress_data:
```

```bash
# รัน
docker compose up -d

# เข้าใช้งานที่ http://localhost:8080
```

### ตัวอย่างที่ 2: MERN Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:6
    volumes:
      - mongo-data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secret
    ports:
      - "27017:27017"

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      NODE_ENV: development
      MONGODB_URI: mongodb://admin:secret@mongodb:27017/myapp?authSource=admin
    depends_on:
      - mongodb
    volumes:
      - ./backend:/app
      - /app/node_modules

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      REACT_APP_API_URL: http://localhost:5000
    depends_on:
      - backend
    volumes:
      - ./frontend:/app
      - /app/node_modules

volumes:
  mongo-data:
```

### ตัวอย่างที่ 3: Microservices

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Database
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend

  # Cache
  redis:
    image: redis:alpine
    networks:
      - backend

  # Services
  user-service:
    build: ./services/user
    environment:
      DATABASE_URL: postgresql://postgres:secret@postgres:5432/users
      REDIS_URL: redis://redis:6379
    networks:
      - backend
    depends_on:
      - postgres
      - redis

  order-service:
    build: ./services/order
    environment:
      DATABASE_URL: postgresql://postgres:secret@postgres:5432/orders
      USER_SERVICE_URL: http://user-service:3000
    networks:
      - backend
    depends_on:
      - postgres
      - user-service

  # API Gateway
  api-gateway:
    build: ./gateway
    ports:
      - "8080:8080"
    environment:
      USER_SERVICE_URL: http://user-service:3000
      ORDER_SERVICE_URL: http://order-service:3000
    networks:
      - backend
      - frontend
    depends_on:
      - user-service
      - order-service

  # Frontend
  web:
    build: ./web
    ports:
      - "80:80"
    environment:
      API_URL: http://localhost:8080
    networks:
      - frontend
    depends_on:
      - api-gateway

networks:
  frontend:
  backend:

volumes:
  postgres-data:
```

### ตัวอย่างที่ 4: Development Environment

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
      CHOKIDAR_USEPOLLING: "true"  # สำหรับ hot-reload
    command: npm run dev
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres-dev:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db

volumes:
  postgres-dev:
```

```bash
# รัน
docker compose -f docker-compose.dev.yml up
```

### ตัวอย่างที่ 5: Production Setup

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: myapp:${VERSION:-latest}
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      SECRET_KEY: ${SECRET_KEY}
    networks:
      - app-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - static-files:/usr/share/nginx/html
    depends_on:
      - app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  static-files:
```

## Environment Variables และ .env Files

### ไฟล์ .env

```bash
# .env
# Database
POSTGRES_PASSWORD=secret
POSTGRES_DB=myapp

# Application
NODE_ENV=development
API_PORT=3000
SECRET_KEY=your-secret-key

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
```

### ใช้ใน docker-compose.yml

```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

  app:
    build: .
    ports:
      - "${API_PORT}:3000"
    environment:
      NODE_ENV: ${NODE_ENV}
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgresql://postgres:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
```

### หลายไฟล์ Environment

```bash
# .env.development
NODE_ENV=development
DEBUG=true

# .env.production
NODE_ENV=production
DEBUG=false
```

```bash
# ระบุไฟล์ env
docker compose --env-file .env.production up
```

## Override Files

### docker-compose.override.yml

```yaml
# docker-compose.yml (base)
version: '3.8'
services:
  app:
    image: myapp:latest
    ports:
      - "3000:3000"
```

```yaml
# docker-compose.override.yml (auto-merged)
version: '3.8'
services:
  app:
    volumes:
      - .:/app       # เพิ่ม volume สำหรับ development
    command: npm run dev
```

```bash
# รันอัตโนมัติ merge ทั้งสองไฟล์
docker compose up
```

### ใช้หลายไฟล์

```bash
# รันด้วยไฟล์เฉพาะ
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Best Practices

### 1. ใช้ Version Control

```bash
# เพิ่มใน .gitignore
.env
.env.local
.env.*.local
docker-compose.override.yml
```

### 2. ใช้ Health Checks

```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      
  app:
    depends_on:
      db:
        condition: service_healthy
```

### 3. จำกัด Resources

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 256M
```

### 4. ใช้ Networks อย่างเหมาะสม

```yaml
services:
  frontend:
    networks:
      - frontend-net
      
  backend:
    networks:
      - frontend-net
      - backend-net
      
  db:
    networks:
      - backend-net    # แยกจาก frontend

networks:
  frontend-net:
  backend-net:
    internal: true     # ไม่สามารถเข้าถึงจากภายนอก
```

### 5. Logging Configuration

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Blog Platform

สร้าง blog platform ด้วย Node.js, PostgreSQL, และ Redis

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: blogpass
      POSTGRES_DB: blog
    volumes:
      - blog-db:/var/lib/postgresql/data

  redis:
    image: redis:alpine

  blog-app:
    build: ./blog-app
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://postgres:blogpass@db:5432/blog
      REDIS_URL: redis://redis:6379
      SESSION_SECRET: your-secret-key
    depends_on:
      - db
      - redis

volumes:
  blog-db:
```

### แบบฝึกหัดที่ 2: E-commerce Stack

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ecommerce
      POSTGRES_DB: shop

  product-service:
    build: ./services/product
    environment:
      DB_URL: postgresql://postgres:ecommerce@postgres/shop

  cart-service:
    build: ./services/cart
    environment:
      REDIS_URL: redis://redis:6379

  redis:
    image: redis:alpine

  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - product-service
      - cart-service
```

## สรุป

ในบทนี้เราได้เรียนรู้:

- ✅ Docker Compose คืออะไรและทำไมต้องใช้
- ✅ โครงสร้างไฟล์ docker-compose.yml
- ✅ Services, networks, volumes, environment variables
- ✅ Docker Compose commands
- ✅ ตัวอย่างการใช้งานจริงหลายรูปแบบ
- ✅ Best Practices สำหรับ Docker Compose

## ถัดไป

ในบทต่อไปเราจะเรียนรู้หัวข้อขั้นสูงเช่น Multi-stage builds, Security, และ Optimization

➡️ [บทที่ 7: หัวข้อขั้นสูง](../07-advanced/README.md)

⬅️ [บทที่ 5: Docker Volumes และ Networking](../05-volumes-networking/README.md)
