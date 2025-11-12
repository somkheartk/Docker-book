# บทที่ 8: ตัวอย่างการใช้งานจริง

## ภาพรวม

ในบทนี้เราจะดูตัวอย่างการใช้งาน Docker ในสถานการณ์จริงที่คุณอาจพบในการทำงาน

## 1. Web Application Stack (Node.js + MongoDB + Redis)

### โครงสร้าง Project

```
my-web-app/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── nginx/
│   └── nginx.conf
└── docker-compose.yml
```

### Backend Dockerfile

```dockerfile
# backend/Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup package*.json ./
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Frontend Dockerfile

```dockerfile
# frontend/Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost || exit 1
CMD ["nginx", "-g", "daemon off;"]
```

### Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:6
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongo-data:/data/db
      - ./mongo-init.js:/docker-entrypoint-initdb.d/init.js:ro
    networks:
      - backend
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  backend:
    build: ./backend
    environment:
      NODE_ENV: production
      PORT: 3000
      MONGODB_URI: mongodb://admin:${MONGO_PASSWORD}@mongodb:27017/myapp?authSource=admin
      REDIS_URL: redis://redis:6379
      SESSION_SECRET: ${SESSION_SECRET}
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
      - frontend
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  frontend:
    build: ./frontend
    depends_on:
      - backend
    networks:
      - frontend
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - frontend
      - backend
    networks:
      - frontend
    restart: unless-stopped

networks:
  frontend:
  backend:
    internal: true

volumes:
  mongo-data:
  redis-data:
```

### Nginx Configuration

```nginx
# nginx/nginx.conf
upstream backend {
    least_conn;
    server backend:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name example.com;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Frontend
    location / {
        proxy_pass http://frontend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # API
    location /api/ {
        proxy_pass http://backend/;
        proxy_http_version 1.1;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Environment Variables

```bash
# .env
MONGO_PASSWORD=supersecret123
SESSION_SECRET=your-secret-key-here
```

### การรันและจัดการ

```bash
# Development
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# ดูสถานะ
docker compose ps

# ดู logs
docker compose logs -f backend

# Scale service
docker compose up -d --scale backend=4

# Update service
docker compose up -d --no-deps --build backend

# Backup database
docker compose exec mongodb mongodump --out /data/backup

# Stop all
docker compose down
```

## 2. Microservices Architecture

### โครงสร้าง Project

```
microservices-app/
├── services/
│   ├── user-service/
│   ├── order-service/
│   ├── product-service/
│   └── notification-service/
├── gateway/
├── docker-compose.yml
└── docker-compose.prod.yml
```

### API Gateway

```dockerfile
# gateway/Dockerfile
FROM node:18-alpine
RUN addgroup -S gateway && adduser -S gateway -G gateway
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --chown=gateway:gateway . .
USER gateway
EXPOSE 8080
CMD ["node", "gateway.js"]
```

```javascript
// gateway/gateway.js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

// Service discovery
const services = {
  user: process.env.USER_SERVICE_URL,
  order: process.env.ORDER_SERVICE_URL,
  product: process.env.PRODUCT_SERVICE_URL
};

// Routes
app.use('/api/users', createProxyMiddleware({
  target: services.user,
  changeOrigin: true,
  pathRewrite: { '^/api/users': '' }
}));

app.use('/api/orders', createProxyMiddleware({
  target: services.order,
  changeOrigin: true,
  pathRewrite: { '^/api/orders': '' }
}));

app.use('/api/products', createProxyMiddleware({
  target: services.product,
  changeOrigin: true,
  pathRewrite: { '^/api/products': '' }
}));

app.listen(8080, () => {
  console.log('API Gateway running on port 8080');
});
```

### Docker Compose for Microservices

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Databases
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend

  redis:
    image: redis:alpine
    networks:
      - backend

  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    ports:
      - "15672:15672"  # Management UI
    networks:
      - backend

  # Services
  user-service:
    build: ./services/user-service
    environment:
      DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@postgres/users
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
    networks:
      - backend
    deploy:
      replicas: 2

  order-service:
    build: ./services/order-service
    environment:
      DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@postgres/orders
      RABBITMQ_URL: amqp://admin:${RABBITMQ_PASSWORD}@rabbitmq
      USER_SERVICE_URL: http://user-service:3000
      PRODUCT_SERVICE_URL: http://product-service:3000
    depends_on:
      - postgres
      - rabbitmq
      - user-service
    networks:
      - backend
    deploy:
      replicas: 2

  product-service:
    build: ./services/product-service
    environment:
      DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@postgres/products
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
    networks:
      - backend
    deploy:
      replicas: 2

  notification-service:
    build: ./services/notification-service
    environment:
      RABBITMQ_URL: amqp://admin:${RABBITMQ_PASSWORD}@rabbitmq
      SMTP_HOST: ${SMTP_HOST}
      SMTP_PORT: ${SMTP_PORT}
    depends_on:
      - rabbitmq
    networks:
      - backend

  # API Gateway
  gateway:
    build: ./gateway
    environment:
      USER_SERVICE_URL: http://user-service:3000
      ORDER_SERVICE_URL: http://order-service:3000
      PRODUCT_SERVICE_URL: http://product-service:3000
    ports:
      - "8080:8080"
    depends_on:
      - user-service
      - order-service
      - product-service
    networks:
      - backend
      - frontend

networks:
  frontend:
  backend:
    internal: true

volumes:
  postgres-data:
```

## 3. CI/CD Pipeline with Docker

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

test:
  stage: test
  script:
    - docker run $IMAGE_TAG npm test

deploy_staging:
  stage: deploy
  script:
    - docker pull $IMAGE_TAG
    - docker-compose -f docker-compose.staging.yml up -d
  only:
    - develop
  environment:
    name: staging

deploy_production:
  stage: deploy
  script:
    - docker pull $IMAGE_TAG
    - docker-compose -f docker-compose.prod.yml up -d --no-deps --build app
  only:
    - main
  environment:
    name: production
  when: manual
```

### GitHub Actions

```yaml
# .github/workflows/docker.yml
name: Docker CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest
          ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }}
        cache-from: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/myapp:buildcache
        cache-to: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/myapp:buildcache,mode=max
    
    - name: Run tests
      run: |
        docker run --rm ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest npm test
    
  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to production
      env:
        SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        SERVER_HOST: ${{ secrets.SERVER_HOST }}
      run: |
        echo "$SSH_PRIVATE_KEY" > private_key
        chmod 600 private_key
        ssh -i private_key -o StrictHostKeyChecking=no user@$SERVER_HOST '
          cd /app &&
          docker-compose pull &&
          docker-compose up -d --no-deps --build app
        '
```

## 4. Development Environment

### Full Stack Development Setup

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
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

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    volumes:
      - ./backend:/app
      - /app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"  # Node.js debugger
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://postgres:dev@postgres/myapp_dev
      REDIS_URL: redis://redis:6379
      SMTP_HOST: mailhog
      SMTP_PORT: 1025
    command: npm run dev
    depends_on:
      - postgres
      - redis

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    volumes:
      - ./frontend:/app
      - /app/node_modules
    ports:
      - "8080:8080"
    environment:
      CHOKIDAR_USEPOLLING: "true"
      REACT_APP_API_URL: http://localhost:3000
    command: npm start
    depends_on:
      - backend

  adminer:
    image: adminer
    ports:
      - "8081:8080"
    depends_on:
      - postgres

volumes:
  postgres-dev:
```

### Backend Development Dockerfile

```dockerfile
# backend/Dockerfile.dev
FROM node:18-alpine

WORKDIR /app

# Install nodemon globally
RUN npm install -g nodemon

# Copy package files
COPY package*.json ./
RUN npm install

# Copy source
COPY . .

EXPOSE 3000 9229

CMD ["nodemon", "--inspect=0.0.0.0:9229", "server.js"]
```

### VS Code Debug Configuration

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker: Attach to Node",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}/backend",
      "remoteRoot": "/app",
      "protocol": "inspector",
      "restart": true
    }
  ]
}
```

## 5. Production Deployment

### Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: myregistry.com/myapp:${VERSION}
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
    networks:
      - app-network
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 5s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx-logs:/var/log/nginx
    depends_on:
      - app
    networks:
      - app-network

  prometheus:
    image: prom/prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - monitoring

  grafana:
    image: grafana/grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    networks:
      - monitoring

networks:
  app-network:
  monitoring:

volumes:
  nginx-logs:
  prometheus-data:
  grafana-data:
```

### Deploy Script

```bash
#!/bin/bash
# deploy.sh

set -e

VERSION=$1
ENVIRONMENT=${2:-production}

if [ -z "$VERSION" ]; then
  echo "Usage: ./deploy.sh <version> [environment]"
  exit 1
fi

echo "Deploying version $VERSION to $ENVIRONMENT"

# Pull latest images
docker compose -f docker-compose.prod.yml pull

# Run database migrations
docker compose -f docker-compose.prod.yml run --rm app npm run migrate

# Deploy with zero-downtime
docker compose -f docker-compose.prod.yml up -d --no-deps --scale app=6
sleep 10
docker compose -f docker-compose.prod.yml up -d --no-deps --scale app=3

# Health check
for i in {1..30}; do
  if curl -f http://localhost/health; then
    echo "Deployment successful!"
    exit 0
  fi
  sleep 2
done

echo "Deployment failed - rolling back"
docker compose -f docker-compose.prod.yml rollback
exit 1
```

## 6. Database Backup และ Restore

### Backup Script

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# PostgreSQL
docker compose exec -T postgres pg_dumpall -U postgres | \
  gzip > $BACKUP_DIR/postgres_${DATE}.sql.gz

# MongoDB
docker compose exec -T mongodb mongodump --archive | \
  gzip > $BACKUP_DIR/mongodb_${DATE}.archive.gz

# Redis
docker compose exec -T redis redis-cli --rdb - | \
  gzip > $BACKUP_DIR/redis_${DATE}.rdb.gz

# Cleanup old backups (keep last 7 days)
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

echo "Backup completed: ${DATE}"
```

### Restore Script

```bash
#!/bin/bash
# restore.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: ./restore.sh <backup_file>"
  exit 1
fi

# PostgreSQL
gunzip < $BACKUP_FILE | docker compose exec -T postgres psql -U postgres

echo "Restore completed"
```

### Automated Backup with Cron

```yaml
# docker-compose.backup.yml
version: '3.8'

services:
  backup:
    image: alpine:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./backups:/backups
      - ./backup.sh:/backup.sh
    entrypoint: sh -c "apk add --no-cache docker-cli && crond -f"
    environment:
      CRON_SCHEDULE: "0 2 * * *"  # Daily at 2 AM
```

## สรุป

ในบทนี้เราได้เห็นตัวอย่างการใช้งานจริง:

- ✅ Web Application Stack แบบสมบูรณ์
- ✅ Microservices Architecture
- ✅ CI/CD Pipeline
- ✅ Development Environment
- ✅ Production Deployment
- ✅ Database Backup และ Restore

## ถัดไป

➡️ [ภาคผนวก: คำสั่งที่ใช้บ่อยและ Troubleshooting](../../appendix/README.md)

⬅️ [บทที่ 7: หัวข้อขั้นสูง](../07-advanced/README.md)
