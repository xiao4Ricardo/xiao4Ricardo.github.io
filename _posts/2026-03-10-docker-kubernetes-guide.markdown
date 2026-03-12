---
layout:     post
title:      "Docker 与 Kubernetes 工程实战"
subtitle:   "From Container Basics to Production Orchestration"
date:       2026-03-10 10:00:00
author:     "Tony L."
header-img: "img/post-bg-kuaidi.jpg"
tags:
    - Docker
    - Kubernetes
    - DevOps
    - Software Engineering
---

## 为什么需要容器化？

"在我电脑上能跑啊"——每个开发者都听过（或说过）这句话。容器技术从根本上解决了环境一致性问题。

**容器 vs 虚拟机：**

```
┌───────────────────────┐    ┌───────────────────────┐
│    Virtual Machine     │    │      Container        │
├───────────────────────┤    ├───────────────────────┤
│  App A  │  App B      │    │  App A  │  App B      │
│  Bins   │  Bins       │    │  Bins   │  Bins       │
│  Guest  │  Guest      │    ├───────────────────────┤
│  OS     │  OS         │    │    Container Runtime  │
├───────────────────────┤    ├───────────────────────┤
│     Hypervisor        │    │      Host OS          │
├───────────────────────┤    ├───────────────────────┤
│     Host OS           │    │      Hardware         │
├───────────────────────┤    └───────────────────────┘
│     Hardware          │
└───────────────────────┘
```

容器共享主机内核，启动快（秒级 vs 分钟级）、资源开销小、镜像更轻量。

## Docker 实战

### 高效 Dockerfile

以 Spring Boot 应用为例：

```dockerfile
# ====== Build Stage ======
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app

# 先复制依赖文件（利用 Docker 层缓存）
COPY build.gradle settings.gradle gradlew ./
COPY gradle/ gradle/
RUN ./gradlew dependencies --no-daemon

# 再复制源代码并构建
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

# ====== Runtime Stage ======
FROM eclipse-temurin:17-jre
WORKDIR /app

# 非 root 用户运行
RUN groupadd -r appuser && useradd -r -g appuser appuser

COPY --from=build /app/build/libs/*.jar app.jar

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

USER appuser
EXPOSE 8080

ENTRYPOINT ["java", "-XX:+UseContainerSupport", \
            "-XX:MaxRAMPercentage=75.0", \
            "-jar", "app.jar"]
```

关键技巧：
1. **多阶段构建**：构建阶段用 JDK，运行阶段只需 JRE，镜像更小
2. **层缓存优化**：先复制依赖文件，后复制源代码——依赖不变时跳过下载
3. **非 root 用户**：安全最佳实践
4. **JVM 容器感知**：`UseContainerSupport` 让 JVM 正确识别容器资源限制

### Vue 前端 Dockerfile

```dockerfile
# Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Serve with Nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

### Docker Compose 全栈开发

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: crm
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-sql:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      retries: 5

  backend:
    build: ./swiftcrew-backend
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: crm
      DB_USER: admin
      DB_PASSWORD: secret
      JWT_SECRET: dev-secret-key
    depends_on:
      postgres:
        condition: service_healthy

  frontend:
    build: ./swiftcrew-frontend
    ports:
      - "5173:80"
    depends_on:
      - backend

volumes:
  pgdata:
```

一条命令启动全栈：`docker compose up -d`

## Kubernetes 核心概念

当你的应用需要在多台机器上运行时，Docker Compose 就不够用了。Kubernetes（K8s）是容器编排的事实标准。

### 核心资源

```
Pod         → 最小部署单元（一个或多个容器）
Deployment  → 管理 Pod 的副本数和滚动更新
Service     → 为 Pod 提供稳定的网络端点
Ingress     → 外部流量的入口（HTTP 路由）
ConfigMap   → 配置数据
Secret      → 敏感数据（密码、证书）
PVC         → 持久化存储
```

### Deployment 示例

```yaml
# k8s/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crm-backend
  labels:
    app: crm-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: crm-backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: crm-backend
    spec:
      containers:
        - name: backend
          image: registry/crm-backend:v1.2.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: backend-config
            - secretRef:
                name: backend-secrets
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: crm-backend-svc
spec:
  selector:
    app: crm-backend
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: crm-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [crm.example.com]
      secretName: crm-tls
  rules:
    - host: crm.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: crm-backend-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: crm-frontend-svc
                port:
                  number: 80
```

## 生产环境关键实践

### 1. 资源限制

永远设置 requests 和 limits。不设置 limits 的 Pod 可能会耗尽节点资源：

```yaml
resources:
  requests:    # 调度依据
    memory: "512Mi"
    cpu: "250m"
  limits:      # 硬性上限
    memory: "1Gi"
    cpu: "500m"
```

### 2. 健康检查

三种探针缺一不可：

```yaml
startupProbe:     # 启动检测（慢启动应用）
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

readinessProbe:   # 就绪检测（是否接收流量）
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 10

livenessProbe:    # 存活检测（是否需要重启）
  httpGet:
    path: /actuator/health
    port: 8080
  periodSeconds: 30
```

### 3. HPA（水平自动扩缩）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: crm-backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: crm-backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 4. 优雅关闭

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```

## 总结

Docker 解决了"环境一致性"问题，Kubernetes 解决了"规模化运行"问题。对于中小团队，Docker Compose 足够应对开发和小规模部署。当你的应用需要高可用、自动扩缩和多节点部署时，再引入 Kubernetes。记住：工具服务于需求，不要为了用 K8s 而用 K8s。

## References

1. Docker Documentation. https://docs.docker.com
2. Kubernetes Documentation. https://kubernetes.io/docs
3. Luksa, M. (2022). "Kubernetes in Action", 2nd Edition. Manning.
