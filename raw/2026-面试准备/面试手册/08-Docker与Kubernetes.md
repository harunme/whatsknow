# Docker 与 Kubernetes 面试手册

## Summary

本手册面向具备容器化实战经验的高级 AI 应用工程师，系统梳理 Docker 与 Kubernetes 从基础使用到高级编排的完整知识点与高频面试题。Docker 通过 Linux Namespace 与 Cgroup 实现进程级隔离，相比虚拟机无需完整 Guest OS，启动快（毫秒级）、体积小（MB 级 vs GB 级），但共享宿主机内核导致隔离性弱于 VM。K8s 则是 CNCF 毕业的容器编排标准，提供自愈、扩缩容、负载均衡、服务发现等企业级能力。两者关系：Docker 是容器运行时，K8s 是编排层，可对接 containerd/cri-o 等多种运行时。掌握 Dockerfile 优化、Multi-stage Build、网络模型、存储卷、安全加固，以及 K8s 的 Pod/Deployment/Service/Ingress 核心对象，是通过容器方向面试的关键。本手册还涵盖 CI/CD 流水线设计、镜像安全扫描与生产排错思路，帮助候选人在实战层面展现深度。

---

## 零、基础知识速查

### 0.1 Docker 安装与基本命令

```bash
# macOS / Windows 安装
# 官网下载 Docker Desktop（包含 Docker Engine + Docker CLI + Docker Compose + K8s）

# Linux 安装
curl -fsSL https://get.docker.com | sh
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER   # 当前用户免 sudo

# 基本命令
docker version                  # 查看版本
docker info                     # 查看 Docker 状态（registry、容器数等）

# 镜像操作
docker images                   # 查看本地镜像
docker pull python:3.11-slim    # 拉取镜像
docker build -t myapp:1.0 .     # 构建镜像（当前目录的 Dockerfile）
docker tag myapp:1.0 registry.example.com/myapp:1.0   # 打标签
docker push registry.example.com/myapp:1.0             # 推送镜像
docker rmi myapp:1.0            # 删除本地镜像
docker image prune -a           # 清理所有未使用镜像

# 容器操作
docker ps                       # 运行中的容器
docker ps -a                    # 所有容器（含已停止）
docker run -d --name myapp -p 8080:8000 myapp:1.0
# -d 后台运行，--name 命名，-p 宿主机端口:容器端口

docker stop myapp               # 停止
docker start myapp              # 启动
docker restart myapp             # 重启
docker rm myapp                  # 删除（容器）
docker rm -f myapp               # 强制删除（运行中）

# 进入容器
docker exec -it myapp /bin/bash   # 进入运行中的容器
docker cp myapp:/app/logs ./logs  # 从容器复制文件到宿主机

# 日志
docker logs -f --tail 100 myapp   # 查看日志（实时跟踪最后100行）

# 查看资源占用
docker stats                      # 实时资源占用
docker top myapp                  # 容器内进程
docker inspect myapp               # 容器详细信息（IP、挂载、环境变量）
```

### 0.2 Dockerfile 基础

```dockerfile
# 最简 Dockerfile 示例（Python 应用）
FROM python:3.11-slim

WORKDIR /app

# 依赖文件先复制（利用缓存）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 再复制代码
COPY . .

# 环境变量
ENV PORT=8000
EXPOSE ${PORT}

# 运行用户（非 root，更安全）
RUN useradd -m appuser
USER appuser

# 启动命令
CMD ["python", "main.py"]
```

```bash
# .dockerignore（避免把不需要的文件打入镜像）
__pycache__/
*.pyc
.git/
.env
node_modules/
*.log
```

### 0.3 Docker Compose 基础

```bash
# 安装（Docker Desktop 自带）
docker-compose --version

# 启动 / 停止
docker-compose up -d              # 后台启动所有服务
docker-compose down               # 停止并删除容器
docker-compose down -v           # 同时删除数据卷
docker-compose ps                 # 查看服务状态
docker-compose logs -f web       # 查看 web 服务日志
docker-compose exec web python manage.py migrate  # 在 web 容器执行命令
```

```yaml
# docker-compose.yml 示例（FastAPI + Redis）
version: "3.8"

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      redis:
        condition: service_healthy  # 等待 redis 就绪
    restart: unless-stopped
    deploy:
      replicas: 2                    # 启动 2 个副本

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

volumes:
  redis_data:                       # 持久化存储
```

### 0.4 常见使用场景

```bash
# 场景1：快速测试 MySQL
docker run --rm -it \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=soc_db \
  -p 3306:3306 \
  mysql:8.0

# 场景2：运行一次性任务
docker run --rm \
  -v $(pwd)/data:/data \
  python:3.11-slim \
  python process.py

# 场景3：本地开发挂载代码（热更新）
docker run -d --name dev \
  -p 8000:8000 \
  -v $(pwd):/app \
  -w /app \
  python:3.11-slim \
  sh -c "pip install -r requirements.txt && uvicorn main:app --reload"
```

### 0.5 Docker vs 虚拟机基础对比

| 维度 | Docker 容器 | 虚拟机 |
|------|------------|--------|
| 启动速度 | 秒级（毫秒~几秒）| 分钟级 |
| 镜像大小 | MB 级（Alpine < 10MB）| GB 级（完整 OS）|
| 资源占用 | 轻量，共享宿主机内核 | 重，全套虚拟化开销 |
| 隔离性 | 进程级（Namespace/Cgroup）| 硬件级（Hypervisor）|
| 数量限制 | 可运行数十上百个容器 | 通常几个虚拟机 |

### 0.6 常见面试基础问题

**Q: 容器和镜像的区别？**
- **镜像（Image）**：静态模板，只读的 blueprints（如 `python:3.11-slim`）
- **容器（Container）**：镜像的运行时实例，可读写，可启动/停止/删除
- 类比：镜像 = 类，容器 = 对象实例

**Q: `docker run` 和 `docker start` 的区别？**
- `docker run`：创建新容器并启动（`create` + `start`）
- `docker start`：启动已存在的停止状态的容器
- `docker restart` = `stop` + `start`

**Q: 容器内进程 root 权限和宿主机 root 一样吗？**
- 不一样。容器内的 root 是宿主机 user namespace 映射后的虚拟 UID
- 容器 root 的 capabilities 受限（如不能挂载文件系统）
- 但在某些配置下（`--privileged`）权限会非常大，生产禁用

---

## 1. 容器 vs 虚拟机 — 核心区别

### 架构差异

| 维度 | 容器 | 虚拟机 |
|------|------|--------|
| 隔离层级 | 进程级（Linux Namespace/Cgroup） | 硬件级（Hypervisor） |
| 启动时间 | 毫秒~秒级 | 分钟级 |
| 镜像体积 | MB 级（ Alpine 可 < 10MB） | GB 级（完整 OS） |
| 共享内核 | 是（安全边界依赖 namespace） | 否（独立内核） |
| 资源开销 | 低（无虚拟化损耗） | 高（vCPU/内存分配给 VM） |

### Docker 架构三大组件

```
Docker Client（CLI） ──REST API──> Docker Daemon（dockerd）
                                        │
                                  containerd（shim runtime）
                                        │
                              runc（真正的容器进程）

Docker Registry（Harbor/Docker Hub）：镜像存储与分发
```

- **dockerd**：守护进程，负责镜像构建、容器生命周期管理
- **containerd**：CNCF 毕业项目，专注容器运行与管理，K8s 默认用它作为 CRI
- **runc**：轻量 CLI 工具，根据 OCI spec 创建和运行容器

### 面试高频追问

**问：为什么 K8s 1.24 后不再内置 Dockershim？**
答：K8s 早期为了直接调用 Docker API 写了 Dockershim 兼容层。1.24 将其移除，原因是 Docker 不兼容 CRI 协议，维护成本高。现在 K8s 通过 containerd 或 cri-dockerd 对接 Docker。

---

## 2. Dockerfile 核心指令

### 关键指令对比

```dockerfile
# 基础镜像选择：优先用 Alpine 或 distroless
FROM python:3.11-slim AS builder

# 安装依赖（每条 RUN 合并层，减少镜像体积）
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# 复制源代码（优先复制依赖文件，再复制代码，利用缓存）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# 暴露端口
EXPOSE 8080

# CMD vs ENTRYPOINT 核心区别：
# CMD：提供默认执行命令，可被 docker run 参数覆盖（覆盖行为）
# ENTRYPOINT：固定入口，参数追加在后面（拼接行为）

# 推荐写法：ENTRYPOINT + CMD 配合
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0"]

# 等价于：python app.py --host 0.0.0.0
```

### CMD vs ENTRYPOINT 实操区别

```bash
# Dockerfile: CMD ["echo", "default"]
# 运行时：docker run image hello
# 输出：hello（CMD 被覆盖）

# Dockerfile: ENTRYPOINT ["echo"]
# 运行时：docker run image hello
# 输出：hello（参数追加到 ENTRYPOINT 后面）
```

面试核心回答：CMD 定义默认参数，可被命令行覆盖；ENTRYPOINT 定义不可变的入口命令，CLI 参数会作为参数拼到后面。两者都可以写 JSON 数组格式（推荐），shell 格式会绕过 exec 模式。

---

## 3. 镜像优化 — 多阶段构建

### Multi-stage Build 实战

```dockerfile
# Stage 1: 构建阶段
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -ldflags="-s -w" -o myapp

# Stage 2: 运行阶段（极简镜像）
FROM alpine:3.19
WORKDIR /root/
COPY --from=builder /app/myapp .
# 只复制二进制，不带源码、构建工具
EXPOSE 8080
CMD ["./myapp"]
```

### 其他优化手段

```dockerfile
# .dockerignore 示例（排除不需要的文件）
.git
node_modules
*.log
tests/
docs/
.env
```

关键优化原则：
- 合并 RUN 指令减少层数：`RUN apt install a && apt install b` 优于分两行
- 清理缓存：每条 apt/yum/pip 命令后清理缓存目录
- 利用构建缓存：将依赖文件（requirements.txt）单独复制并优先构建
- 使用轻量基础镜像：alpine、distroless、scratch
- 合并指令减少层：`COPY a b c /dir/` 优于三个 COPY

---

## 4. Docker 网络

### 四种网络模式

```bash
# bridge（默认）：容器连接到 docker0 网桥，NAT 通信
docker run --network bridge nginx

# host：容器直接使用宿主机网络栈（无隔离，高性能）
docker run --network host nginx

# none：禁用所有网络，只有 loopback
docker run --network none busybox

# overlay：跨主机容器通信（Docker Swarm 场景）
docker network create --driver overlay my-overlay
```

### 容器间通信

同 bridge 网络下，Docker 内置 DNS 会解析容器名到容器 IP：

```bash
# 容器 web 访问数据库
# 只需用服务名作为 hostname：mysql://db:3306
docker run --network mynet --name web nginx
docker run --network mynet --name db mysql
# web 容器内可直接用 db 作为主机名访问
```

### Docker Compose 网络

```yaml
version: "3.8"
services:
  web:
    build: .
    ports:
      - "8080:8080"
    networks:
      - frontend
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:15
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

networks:
  frontend:
  backend:   # 隔离网络，web 无法直接访问 db
```

---

## 5. 数据持久化

### 三种挂载方式对比

| 维度 | Volume | Bind Mount | tmpfs Mount |
|------|--------|------------|-------------|
| 存储位置 | `/var/lib/docker/volumes` | 宿主机任意路径 | 内存（tmpfs） |
| 生命周期 | 独立于容器 | 依赖宿主机路径 | 随容器消失 |
| 适用场景 | 数据库、日志持久化 | 配置文件、源码热更新 | 敏感数据（临时） |
| 容器迁移 | 可直接挂载新容器 | 需指定相同宿主机路径 | 不适用 |

### 实战用法

```bash
# Volume（推荐，Docker 管理）
docker volume create my-data
docker run -v my-data:/var/lib/postgresql/data postgres:15

# Bind Mount（开发时热更新源码）
docker run -v $(pwd)/src:/app/src nginx

# tmpfs（密码、token 等敏感数据）
docker run --tmpfs /run/secrets nginx
```

### 数据容器（已不推荐）

数据容器模式（`docker run --volumes-from`）在 Docker Compose 出现后基本被 volume 替代，不再是主流实践。

---

## 6. Docker Compose

### 完整示例

```yaml
version: "3.8"
services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    profiles: ["dev", "prod"]   # 只在指定 profile 时启动
    restart: unless-stopped     # 重启策略
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    environment:
      - DB_HOST=db
      - REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-net

  db:
    image: postgres:15-alpine
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - app-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - app-net

volumes:
  pg-data:
  redis-data:

networks:
  app-net:
    driver: bridge
```

重启策略：`no`（永不重启）、`always`（总是重启）、`on-failure[:n]`（失败时重启，最多 n 次）、`unless-stopped`（手动停止后不重启）。

---

## 7. 容器安全

### 用户权限控制

```dockerfile
# Dockerfile 中创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 运行时强制指定用户
docker run --user 1000:1000 nginx
```

### 安全内核特性

```bash
# Seccomp（系统调用过滤）：Docker 默认已禁用 ~44 个危险系统调用
docker run --security-opt seccomp=profile.json nginx

# AppArmor（程序轮廓限制）：限制进程能访问的文件/网络资源
docker run --security-opt apparmor=my-profile nginx

# Linux Capabilities（细粒度特权）：
# 容器默认移除所有 CAP_SYS_ADMIN（防止 mount 操作）
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

### 镜像签名与安全实践

```bash
# 使用 DOCKER_CONTENT_TRUST 环境变量强制签名验证
export DOCKER_CONTENT_TRUST=1

# 镜像扫描（参见第 11 节）
# 定期更新基础镜像，使用官方认证镜像
```

安全最佳实践：最小权限用户、只读根文件系统（`--read-only`）、禁止特权模式（`--privileged`）、使用非 root 用户运行应用。

---

## 8. Kubernetes 核心对象

### K8s vs Docker Compose

| 维度 | Docker Compose | Kubernetes |
|------|---------------|-------------|
| 适用规模 | 单主机开发/测试 | 多节点生产集群 |
| 自愈能力 | 无（容器挂了不会自动重启） | 有（Pod 挂了自动重建） |
| 扩缩容 | 手动 | 自动 HPA |
| 服务发现 | 容器名 DNS | ClusterIP/NodePort/LoadBalancer |
| 滚动更新 | 健康检查需自己实现 | 内置 RollingUpdate 策略 |

### 核心对象详解

```yaml
# Pod：最小调度单元，一个 Pod 可包含多个容器（共享 network/pid/ipc namespace）
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
    - name: app
      image: myapp:v1.2.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: db.host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: db.password
---
# Deployment：管理 Pod 副本数，提供滚动更新和回滚
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:v1.2.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
---
# Service：固定访问入口，自动负载均衡
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80        # Service 端口
      targetPort: 8080  # Pod 端口
  type: ClusterIP    # ClusterIP / NodePort / LoadBalancer
---
# ConfigMap：非敏感配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  db.host: "db-service"
  log.level: "info"
---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### 探针类型

- **readinessProbe**：就绪探针，流量才进入 Pod
- **livenessProbe**：存活探针，失败则重启 Pod
- **startupProbe**：启动探针，容器启动完成前禁用其他探针（适合慢启动应用）

---

## 9. K8s Ingress 与 TLS

### Ingress 基础配置

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-svc
                port:
                  number: 80
```

### cert-manager 自动续期

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

安装 cert-manager 后，为 Ingress 添加 `cert-manager.io/cluster-issuer` 注解，证书即可自动申请与续期，无需手动管理。

### Ingress Controller 选型

- **Nginx Ingress Controller**：功能全面，社区活跃，适合大多数场景
- **Traefik**：支持 TCP/UDP，更适合微服务
- **Kong**：API Gateway 能力强，适合复杂路由策略
- 选型依据：性能需求、功能需求、团队熟悉度

---

## 10. CI/CD 流水线

### Jenkins + Docker + K8s 实战

```groovy
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "registry.example.com/myapp"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = credentials('kubeconfig-prod')
    }
    stages {
        stage('Checkout') {
            steps { git branch: 'main', url: 'https://github.com/org/myapp.git' }
        }
        stage('Build & Test') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .'
                sh 'docker run --rm ${DOCKER_IMAGE}:${DOCKER_TAG} pytest'
            }
        }
        stage('Security Scan') {
            steps {
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKER_IMAGE}:${DOCKER_TAG}'
            }
        }
        stage('Push Image') {
            steps {
                sh 'docker push ${DOCKER_IMAGE}:${DOCKER_TAG}'
                sh 'docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest'
                sh 'docker push ${DOCKER_IMAGE}:latest'
            }
        }
        stage('Deploy to K8s') {
            when { branch 'main' }
            steps {
                sh '''
                    sed 's|IMAGE_TAG|${DOCKER_TAG}|g' k8s/deployment.yaml | \
                    kubectl apply -f - --namespace=production
                '''
            }
        }
    }
    post {
        failure {
            emailext subject: 'Pipeline Failed', body: "Build #${env.BUILD_NUMBER} failed"
        }
    }
}
```

### 灰度发布策略

**蓝绿部署**：两套环境（蓝=当前生产，绿=新版本），切换流量时更新 LoadBalancer 指向，新版本有问题可秒级回切。

**金丝雀发布**（Canary）：

```yaml
# 10% 流量切到新版本，逐步提升
apiVersion: v1
kind: Service
metadata:
  name: my-app-canary
spec:
  selector:
    app: my-app
    version: canary   # 标签筛选 canary 版本 Pod
  ports:
    - port: 80
---
# 通过 Ingress 权重分流
# 配合 Argo Rollouts 实现更精细的流量控制（基于 header/cookie/权重）
```

### 回滚机制

```bash
# K8s Deployment 回滚
kubectl rollout undo deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3

# Docker Compose 回滚
docker-compose down && docker-compose -f docker-compose.v1.yml up -d

# 镜像标签回滚：修改 Deployment 镜像版本重新 apply
```

---

## 11. 镜像安全扫描

### Trivy 实战

```bash
# 安装
brew install trivy

# 扫描镜像（推荐集成到 CI）
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest

# 扫描项目漏洞
trivy fs --severity HIGH,CRITICAL ./

# 输出 JSON 报告供 Jenkins 解析
trivy image --format json --output report.json myapp:latest
```

### 高危漏洞处理流程

1. **识别**：Trivy 扫描发现 CVE-2024-XXXX（CVSS 9.8）
2. **评估**：判断是否影响自身基础镜像和应用依赖
3. **修复**：升级基础镜像或应用依赖包版本
4. **验证**：重新扫描确认漏洞消除
5. **上线**：触发新 CI 构建，替换生产镜像

### Clair 集成

Clair 是 CoreOS 开源的镜像扫描器，适合自建 Harbor 仓库场景，Harbor 内置集成 Clair 提供 Web UI 可视化扫描结果。

---

## 12. 排错实战

### 容器起不来

```bash
# 1. 查看容器状态
docker ps -a | grep <name>

# 2. 查看启动日志
docker logs <container_id>

# 3. 交互模式排查
docker run -it --rm image-name /bin/sh

# 4. 常见原因：
#    - 端口被占用（docker logs 报 address already in use）
#    - 健康检查失败（docker inspect 查看 RestartCount）
#    - 权限不足（挂载卷无权限 /proc/sys 等敏感路径）
```

### 网络不通

```bash
# 1. 查看容器网络
docker network inspect bridge

# 2. 容器内网络测试
docker exec <container_id> ping <target>
docker exec <container_id> curl http://<service>:<port>

# 3. DNS 解析问题
docker exec <container_id> nslookup <service_name>
docker exec <container_id> cat /etc/resolv.conf

# 4. K8s 网络排查
kubectl get pod -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous  # 查看崩溃前日志
kubectl exec -it <pod-name> -- /bin/sh
```

### 存储挂不上

```bash
# 1. 查看 volume 状态
docker volume inspect <volume_name>

# 2. K8s PVC 问题
kubectl get pvc
kubectl describe pvc <pvc-name>

# 3. 常见原因：
#    - StorageClass 不存在
#    - PVC accessMode 不匹配（ReadWriteOnce vs ReadWriteMany）
#    - PV 绑定到了错误的 PVC
```

### 日志收集

```bash
# Docker 日志驱动：json-file（默认）、syslog、fluentd、awslogs
docker run --log-driver=fluentd --log-opt fluentd-address=fluentd:24224 nginx

# K8s 日志：kubectl logs（单 Pod）、kubectl logs --tail=100 -l app=myapp（标签筛选）
# 生产推荐：EFK（Elasticsearch + Fluentd + Kibana）或 Loki + Grafana
```

---

## 面试 Q&A

**Q1：Docker 和 K8s 的关系是什么？K8s 可以不用 Docker 吗？**
A：Docker 是容器运行时，负责创建和管理容器。K8s 是容器编排平台，通过 CRI（Container Runtime Interface）对接容器运行时。1.24+ 可以完全不用 Docker，直接用 containerd 或 cri-o，Docker 镜像（v2 manifest）仍可通过 ctr 直接加载使用。

**Q2：Docker 的 COPY 和 ADD 指令区别？**
A：COPY 是纯复制，推荐优先使用。ADD 多了两个能力：自动解压 tar/gzip/bzip2/xz 文件，以及从 URL 下载文件。如果只需复制，用 COPY 更清晰，且不会引入意外解压行为。

**Q3：K8s 中 Service 的 ClusterIP、NodePort、LoadBalancer 区别？**
A：ClusterIP 是集群内部访问的虚拟 IP（默认）；NodePort 在每个节点上暴露相同端口（30000-32767），外部可通过 `<NodeIP>:<NodePort>` 访问；LoadBalancer 通过云厂商 LB 代理，分配外部 IP（需要云厂商支持，如腾讯云 TCE）。

**Q4：Pod 内多个容器共享什么命名空间？**
A：共享 network、ipc、uts 命名空间；不共享 pid（但可在 YAML 中启用 shareProcessNamespace）；每个容器有独立的 filesystem。

**Q5：Docker Compose 中 depends_on 和 healthcheck 的 condition: service_healthy 是什么关系？**
A：depends_on 只能保证启动顺序，无法保证服务就绪。配合 healthcheck 和 condition: service_healthy，可以真正等待下游服务（如数据库）健康后再启动依赖方，是生产级配置的关键实践。

**Q6：K8s 中如何实现配置热更新？**
A：ConfigMap 变更后，Pod 内挂载的 ConfigMap 不会自动更新（默认通过 symlink 更新有 1 分钟缓存）。推荐方案：使用挂载 subPath 以外的方式，配置变更时触发 Pod 滚动重建；或使用 confd/Reloader 等工具自动触发 Deployment 滚动更新。

**Q7：什么是 OCI（Open Container Initiative）？**
A：OCI 定义了容器格式和运行时的开放标准，包含 OCI Runtime Spec（容器配置和运行时规范）、OCI Image Spec（镜像格式规范）、OCI Distribution Spec（镜像分发规范）。runc 是首个 OCI 认证的运行时实现。

**Q8：生产环境中如何保证 K8s 集群的高可用？**
A：至少 3 个 Master 节点（etcd 奇数节点）；多可用区部署；kube-apiserver 前置 LoadBalancer；Pod 反亲和性分散到不同节点；资源 limits 防止单 Pod 耗尽节点；PodDisruptionBudget 限制单次 disruption 影响的 Pod 数量。

---

Last Updated: 2026-04-09
