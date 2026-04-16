# 六、工程化（Docker / CI/CD / PM2 / K8s） · 答案

> 本章共 20 题，覆盖 Docker（Q1-Q5）、CI/CD（Q6-Q9）、PM2（Q10-Q12）、Kubernetes（Q13-Q18）、综合（Q19-Q20）

---

## 面试官快速索引

| 题号 | 核心考察点 | 难度 | 追问深度 |
|------|-----------|------|---------|
| Q1 | Docker 核心概念、镜像分层 | ⭐⭐ | 可深问 Union FS |
| Q2 | Dockerfile COPY vs ADD、多阶段构建 | ⭐⭐ | 可深问 build cache |
| Q3 | Docker 容器化改造、配置注入 | ⭐⭐⭐ | 可深问 ConfigMap/Secret |
| Q4 | Docker Compose vs Swarm | ⭐⭐ | 可深问 服务发现 |
| Q5 | Docker 网络模式 bridge/host/overlay | ⭐⭐ | 可深问 DNS 服务发现 |
| Q6 | CI/CD 流水线设计 | ⭐⭐ | 可深问 GitOps |
| Q7 | Git Flow vs TBD | ⭐⭐ | 可深问 feature flag |
| Q8 | 蓝绿/金丝雀/滚动更新 | ⭐⭐⭐ | 可深问 Argo Rollouts |
| Q9 | SonarQube/ESLint/测试覆盖率 | ⭐⭐ | 可深问 quality gate |
| Q10 | PM2 cluster 原理 | ⭐⭐ | 可深问负载均衡 |
| Q11 | PM2 日志/监控/graceful reload | ⭐⭐ | 可深问 startup script |
| Q12 | PM2 vs Docker/K8s | ⭐⭐ | 可深问 container 优势 |
| Q13 | K8s 核心组件 Pod/Service/Deployment/Ingress | ⭐⭐⭐ | 可深问 ReplicaSet vs Deployment |
| Q14 | K8s Service 类型 ClusterIP/NodePort/LoadBalancer | ⭐⭐⭐ | 可深问 Headless Service |
| Q15 | K8s 资源配额 ResourceQuota/LimitRange | ⭐⭐⭐ | 可深问 quota 规划 |
| Q16 | K8s HPA 自动扩缩容 | ⭐⭐⭐ | 可深问 VPA vs HPA |
| Q17 | K8s ConfigMap 和 Secret | ⭐⭐ | 可深问加密存储 |
| Q18 | K8s PV/PVC/动态 provisioning | ⭐⭐⭐ | 可深问 StorageClass |
| Q19 | Git rebase vs merge | ⭐⭐ | 可深问交互式 rebase |
| Q20 | 高可用部署架构 | ⭐⭐⭐ | 可深问多活架构 |

---

## 6.1 Docker

### Q1：Docker 的核心概念（镜像、容器、仓库）是什么？镜像的分层结构有什么好处？

**考察点**：Docker 核心概念、Union FS、分层缓存

**答案要点**：
- **核心概念**：
  - **镜像（Image）**：只读模板，包含运行应用所需的文件系统、依赖、环境变量、启动命令。类似于面向对象的"类"
  - **容器（Container）**：镜像的运行实例，是镜像的可写层。类似于面向对象的"实例"
  - **仓库（Registry）**：存储和分发镜像的服务，如 Docker Hub、阿里云 ACR、Harbor
- **镜像分层结构**：
  - 基于 **UnionFS（联合文件系统）**，镜像由多个只读层叠加而成
  - 每个 Dockerfile 指令创建一层：`FROM`/`RUN`/`COPY`/`ADD`
  - 容器在镜像顶层添加一层**可写层**（容器层）
  - 所有容器共享底层只读层，节省磁盘空间
- **分层的好处**：
  1. **复用**：相同基础镜像的容器共享底层 layers，节省存储
  2. **构建缓存**：如果某层未变化，构建时可复用缓存，大幅加速构建
  3. **版本管理**：每层对应一个 Dockerfile 指令，可追溯、可回滚

**可能的追问**：
- 什么是 UnionFS 的 COW（Copy-on-Write）机制？
- `docker commit` 和 `docker export` 有什么区别？
- 镜像的层数过多会影响什么？（启动速度、存储效率）

---

### Q2：Dockerfile 中 COPY 和 ADD 的区别？多阶段构建（multi-stage build）解决什么问题？

**考察点**：Dockerfile 指令、构建优化

**答案要点**：
- **COPY vs ADD**：
  | 特性 | COPY | ADD |
  |------|------|-----|
  | 源路径 | 本地文件/目录 | 本地文件/目录 + **远程 URL** + **解压 tar** |
  | 透明性 | 明确（只复制） | 行为复杂（URL/解压） |
  | 推荐场景 | ✅ 优先使用 | 仅用于自动解压 tar/zip/gzip 文件 |
- **多阶段构建**：
  - 解决：构建产物中包含编译工具、源码等冗余内容，导致镜像体积过大
  - 原理：一个 Dockerfile 中有多个 `FROM` 阶段，每个阶段构建不同的东西
  - 只将最终需要的文件复制到最终镜像中
  ```dockerfile
  # 第一阶段：构建
  FROM node:18 AS builder
  WORKDIR /app
  COPY . .
  RUN npm run build   # 产生 dist/ 产物

  # 第二阶段：运行（只复制构建产物）
  FROM nginx:alpine
  COPY --from=builder /app/dist /usr/share/nginx/html
  ```
- **效果**：最终镜像不包含 Node.js 编译环境，体积从 1GB 降到 20MB

**可能的追问**：
- `docker build` 的构建缓存如何失效？如何强制重建某层？
- `RUN`, `CMD`, `ENTRYPOINT` 的区别？Shell form vs Exec form？
- .dockerignore 文件的作用是什么？它和 .gitignore 有什么区别？

---

### Q3：你主导了 SOC 平台的 Docker 容器化改造，从物理机部署迁移到容器化部署，遇到了哪些问题？如何处理配置管理和密钥注入？

**考察点**：容器化迁移经验、配置注入实践

**答案要点**：
- **容器化迁移中的问题**：
  1. **环境差异**：本地开发/测试/生产环境不一致，容器化后需要统一 base image
  2. **启动顺序**：物理机上服务启动顺序可人工控制，容器化后需要考虑服务依赖（depends_on 不足：只保证启动顺序不保证健康）
  3. **日志收集**：物理机日志在文件中，容器日志在 stdout/stderr，需要对接 ELK/EFK
  4. **健康检查**：容器漂移/重启，需要 `HEALTHCHECK` 配合服务自检
- **配置管理方案**：
  1. **环境变量注入**：`docker run -e KEY=VALUE`，K8s 中用 ConfigMap/Secret
  2. **配置中心**：Nacos/Apollo 管理配置，容器启动时从配置中心拉取
  3. **配置文件挂载**：docker-compose 中用 `volumes` 挂载配置目录
- **密钥注入方案**：
  1. **Docker Secret**（Swarm 模式）：`echo "password" | docker secret create db_password -`
  2. **K8s Secret**：Base64 编码存储，`kubectl create secret` 或 YAML 声明
  3. **外部密钥管理**：HashiCorp Vault / AWS Secrets Manager，容器启动时从 Vault 获取
  4. **不要写在镜像里**：密钥不打包进镜像，运行时通过环境变量或挂载

**可能的追问**：
- Docker 的 `HEALTHCHECK` 和 K8s 的 readinessProbe/livenessProbe 有什么区别？
- 容器漂移（drifting）是什么？容器如何保持 IP 固定？
- 容器化后如何做日志的集中收集？（filebeat/fluentd 边车部署）

---

### Q4：Docker Compose 和 Docker Swarm 的区别？在 agentic-soc-platform 中，docker-compose 的服务编排结构是怎样的？

**考察点**：Compose vs Swarm、编排能力

**答案要点**：
- **Docker Compose vs Docker Swarm**：
  | 维度 | Docker Compose | Docker Swarm |
  |------|---------------|-------------|
  | 定位 | 单主机本地开发/测试 | 多主机生产集群编排 |
  | 节点数 | 单节点 | 多节点（Manager + Worker） |
  | 扩缩容 | 手动 `docker-compose up --scale` | `docker service scale` |
  | 负载均衡 | 无（需配合 Nginx） | 内置 VIP 负载均衡 |
  | 滚动更新 | ❌ | ✅（`--update-delay`） |
  | 服务发现 | 无 | 内置 DNS 服务发现 |
  | 适用场景 | 开发环境、CI/CD 测试 | 小规模生产部署 |
- **agentic-soc-platform 的 docker-compose 结构**：
  ```yaml
  services:
    soc-api:       # Koa API 网关
      build: ./api
      depends_on: [db, redis, nacos]
    soc-worker:    # FastAPI 后台处理
      build: ./worker
      scale: 3     # 本地开发时用 scale 扩展
    db:
      image: mysql:8.0
    redis:
      image: redis:7-alpine
    kafka:
      image: confluentinc/cp-kafka
    nacos:
      image: nacos/nacos-server
  ```
- **K8s 演进路径**：docker-compose 用于本地开发，生产环境迁移到 K8s（docker-compose → Kompose 转换工具辅助）

**可能的追问**：
- `depends_on` 能保证服务就绪吗？它和 `healthcheck` 有什么关系？
- Docker Swarm 的 Service 如何做滚动更新？
- 什么是 `docker stack deploy`？和 `docker-compose up` 的区别？

---

### Q5：Docker 网络模式（bridge/host/none/overlay）区别？Docker 如何实现容器间通信？

**考察点**：Docker 网络模型、容器通信

**答案要点**：
- **四种网络模式**：
  - **`bridge`（默认）**：容器连接到 `docker0` 网桥，各容器通过 veth pair 连接网桥，通过 NAT 访问外部网络
  - **`host`**：容器直接使用宿主机网络命名空间，无网络隔离，性能好但端口冲突风险
  - **`none`**：容器有独立的网络栈，但不配置任何网络接口（完全隔离）
  - **`overlay`**：跨主机容器网络，用于 Docker Swarm 集群，VXLAN 隧道封装
- **容器间通信**：
  - **同一主机**：通过 bridge 网桥直接通信（同一网段）
  - **不同主机**：通过 overlay 网络（Swarm）或自定义 bridge + 路由
  - **服务发现**：Docker 内置 DNS，`container_name` 自动解析为容器 IP（自定义 bridge 网络内）
- **K8s 网络对比**：K8s 的 CNI（Calico/Flannel）替代了 Docker 原生网络，提供更强大的网络策略（NetworkPolicy）

**可能的追问**：
- Docker bridge 网络和 host 网络的 performance 差距有多大？（bridge 有 NAT 开销）
- 容器如何访问外部网络？（NAT + SNAT）
- Docker 的 `--link` 已经被废弃，什么机制替代了它？

---

## 6.2 CI/CD

### Q6：你搭建了 Jenkins CI/CD 流水线，描述一下你们的流水线流程（代码提交→构建→测试→部署）。

**考察点**：CI/CD 流水线设计、自动化流程

**答案要点**：
- **典型流水线阶段**：
  ```
  代码提交 → Git Hook 触发 Webhook → Jenkins Master 调度 → Agent 执行
  ├── 1. Checkout：拉取代码（git clone）
  ├── 2. 依赖安装：npm install / pip install（缓存加速）
  ├── 3. Lint & 格式化：ESLint / Prettier / Black
  ├── 4. 单元测试：Jest / Pytest（覆盖率报告）
  ├── 5. 构建：Webpack / Vite build / Docker build
  ├── 6. SonarQube 扫描：代码质量检查 + 安全扫描
  ├── 7. 安全扫描：Trivy / Snyk 镜像漏洞扫描
  ├── 8. 推送镜像：docker push 到 Harbor
  ├── 9. 部署到测试环境：kubectl apply / helm upgrade
  ├── 10. 集成测试：自动化 E2E 测试（Playwright/Cypress）
  └── 11. 人工审批（可选）→ 部署到生产环境
  ```
- **Jenkinsfile 声明式流水线**：
  ```groovy
  pipeline {
    agent { label 'docker-node' }
    environment {
      REGISTRY = 'harbor.soc.com'
      IMAGE = "${REGISTRY}/soc-api:${GIT_COMMIT[0..7]}"
    }
    stages {
      stage('Build & Test') {
        steps {
          sh 'docker build -t $IMAGE .'
          sh 'docker run $IMAGE pytest tests/'
        }
      }
      stage('Security Scan') {
        steps {
          sh 'trivy image --severity HIGH,CRITICAL $IMAGE'
        }
      }
      stage('Deploy') {
        when { branch 'main' }
        steps {
          sh 'helm upgrade soc-api ./chart --set image.tag=$GIT_COMMIT'
        }
      }
    }
    post {
      always { cleanWs() }
    }
  }
  ```

**可能的追问**：
- Jenkins Master 和 Agent（slave）的关系是什么？什么情况下需要分布式构建？
- Jenkins 的凭证（credentials）如何安全管理？
- 什么是 GitOps？ArgoCD 如何实现 GitOps 流水线？

---

### Q7：Git Flow 和 Trunk Based Development 的区别？你们团队采用哪种分支策略？

**考察点**：分支策略、团队协作模式

**答案要点**：
- **Git Flow**：
  - 分支结构：`main`（生产）+ `develop`（开发）+ `feature/*`（功能）+ `release/*`（发布）+ `hotfix/*`（热修复）
  - 优点：版本发布节奏清晰，热修复路径明确
  - 缺点：分支合并冲突多、发布周期长（需要维护多个长分支）
- **Trunk Based Development（TBD）**：
  - 所有开发者在 `main`（trunk）上开发，短生命周期 feature flag
  - 频繁合并（每天至少一次），避免长期分支
  - 通过 feature flag 隐藏未完成功能
  - 优点：减少合并冲突，CI/CD 友好，快速反馈
  - 缺点：需要成熟的 feature flag 基础设施
- **SOC 平台团队策略**：TBD + hotfix 分支
  - 主线开发：所有功能合并到 `main`
  - Feature flag：`FEATURE_AI_ANALYSIS=true/false` 控制功能开关
  - 热修复：`hotfix/*` 分支直接合并到 `main` 和生产对应 tag

**可能的追问**：
- feature flag 和分支开发的各自优缺点？两者可以结合吗？
- Git Flow 中 release 分支的作用是什么？
- 如何避免 Trunk Based Development 中频繁合入破坏 CI 构建？

---

### Q8：蓝绿部署、金丝雀发布、滚动更新的区别？在 SOC 平台中你们用哪种部署策略？

**考察点**：部署策略、发布风险控制

**答案要点**：
- **三种部署策略对比**：
  | 策略 | 原理 | 优点 | 缺点 |
  |------|------|------|------|
  | **蓝绿部署** | 两套环境（蓝/绿），切流方式更新 | 回滚极快（切 DNS/负载均衡），零停机 | 双倍资源成本 |
  | **金丝雀发布** | 新版本先给小部分用户（5%），逐步扩大 | 风险可控，可实时监控 | 实现复杂，需流量管理 |
  | **滚动更新** | K8s 默认策略，逐步替换旧 Pod | 资源利用率高，无需双倍资源 | 回滚较慢，旧新共存时版本混乱 |
- **SOC 平台部署策略**：
  - **开发/测试环境**：滚动更新，快速验证
  - **生产环境**：**金丝雀发布**为主
    - 第一批 5% 流量（按用户 ID hash）
    - 监控告警错误率/响应时间
    - 稳定后扩到 50% → 全量
    - K8s 中通过 **Argo Rollouts** 或 **Istio VirtualService** 实现金丝雀流量分配

**可能的追问**：
- 蓝绿部署的流量切换如何实现？（DNS 切换 / 负载均衡权重 / SLB 切换）
- Argo Rollouts 的 AnalysisTemplate 和 MetricProvider 是什么？
- 金丝雀发布中，如何用 Prometheus 指标做自动回滚？

---

### Q9：CI/CD 流水线中如何做代码质量卡点？SonarQube、ESLint、单元测试覆盖率各自的作用？

**考察点**：代码质量门禁、自动化检查

**答案要点**：
- **三道质量卡点**：
  1. **ESLint / Prettier**：代码风格检查（缩进/分号/变量声明）+ 潜在 bug 检测（`no-unused-vars`/`no-eval`）
     - 目的：代码风格统一，降低维护成本
  2. **单元测试覆盖率**：
     - 覆盖率指标：`--coverage` 生成报告（行覆盖率、分支覆盖率）
     - 卡点：`branch-coverage < 70%` 禁止合入
     - 注意：覆盖率不是万能的，覆盖率高不等于测试质量高
  3. **SonarQube**：静态代码分析（代码味道/重复代码/安全漏洞/代码复杂度）
     - 扫描语言：Java/JS/Python/TS 等
     - Quality Gate：所有新代码必须满足 Quality Gate（0 Bug / 0 漏洞 / 覆盖率达标）
- **自动化门禁实现**：
  - SonarQube Webhook 回调，PR 评论自动发布分析结果
  - GitLab Merge Request 合并卡点：MR 必须通过 CI/CD 流水线 + SonarQube Quality Gate

**可能的追问**：
- SonarQube 的代码味道（Code Smell）和 Bug/漏洞有什么区别？
- 如何在 SonarQube 中配置 Python 代码的覆盖率报告？
- 单元测试和集成测试的覆盖率如何分别统计？

---

## 6.3 PM2

### Q10：PM2 的 cluster 模式原理是什么？和 Node.js 原生 cluster 模块有什么区别？

**考察点**：PM2 cluster、进程管理、负载均衡

**答案要点**：
- **PM2 cluster 模式原理**：
  - 基于 Node.js 原生 `cluster` 模块封装，提供更友好的 CLI 和管理能力
  - 自动启动 N 个 worker 进程（N = CPU 核数，可配置 `instances`）
  - **负载均衡**：Node.js 原生 cluster 在非 Windows 上使用 **round-robin**（轮询），PM2 也使用相同策略
  - **文件描述符共享**：所有 worker 共享同一个 Server Socket（通过 `SO_REUSEPORT`）
- **与原生 cluster 模块区别**：
  | 维度 | 原生 cluster | PM2 cluster |
  |------|------------|------------|
  | 进程管理 | 需手动管理 fork/exit | CLI 一键管理 |
  | 日志 | 无内置 | 内置日志轮转（`logs/` 目录） |
  | 热重载 | 需手动实现 | `pm2 reload` 零停机 |
  | 监控 | 无 | `pm2 monit` 实时监控 |
  | 远程管理 | 无 | `pm2-runtime`、API 接口 |
  | 启动脚本 | 需写 JS 代码 | JSON declarative 配置 |

**可能的追问**：
- PM2 的 `exec_mode: 'cluster'` 和 `fork` 模式有什么区别？
- PM2 如何处理 worker 进程崩溃后的自动重启？
- 什么是 PM2 的 **graceful reload**？它和普通 reload 有什么区别？

---

### Q11：PM2 的日志管理、进程监控、自动重启各自怎么配置？如何实现零停机部署（graceful reload）？

**考察点**：PM2 管理能力、零停机

**答案要点**：
- **配置方式**（`ecosystem.config.js`）：
  ```javascript
  module.exports = {
    apps: [{
      name: 'soc-api',
      script: 'dist/server.js',
      instances: 4,
      exec_mode: 'cluster',
      autorestart: true,        // 崩溃后自动重启
      watch: false,
      max_memory_restart: '1G', // 内存超限自动重启
      env: {
        NODE_ENV: 'production'
      },
      error_file: './logs/err.log',   // 错误日志
      out_file: './logs/out.log',     // 标准输出
      log_date_format: 'YYYY-MM-DD HH:mm:ss',
      merge_logs: true,                // 多 worker 日志合并
    }]
  }
  ```
- **日志管理**：`pm2 logs` 实时查看，`pm2 logrotate` 自动日志轮转（避免磁盘满）
- **进程监控**：
  - `pm2 monit`：终端实时监控
  - `pm2 list`：查看所有进程状态
  - Keymetrics / PM2 Plus：云端监控面板
- **零停机部署（graceful reload）**：
  ```bash
  pm2 reload soc-api  # 先启动新进程，再关闭旧进程，连接无缝切换
  ```
  - 本质：PM2 发送 `SIGINT`（SIGTERM）信号，等待进程优雅退出（graceful shutdown）后，再启动新进程
  - **关键**：应用需要监听 `process.on('SIGTERM')` 执行清理工作，再退出

**可能的追问**：
- PM2 startup script 是做什么的？重启服务器后如何自动恢复 PM2 进程？
- PM2 的 `max_restarts` 和 `restart_delay` 参数有什么用？
- `pm2 stop` 和 `pm2 delete` 有什么区别？

---

### Q12：PM2 和 Docker/K8s 的关系是什么？在什么场景下用 PM2 而不用 K8s？

**考察点**：PM2 定位、容器 vs 进程管理器

**答案要点**：
- **关系**：PM2 和 Docker/K8s 是**互补**的，不是互斥的
  - **PM2**：Node.js 进程管理器，管理 Node.js 进程的生命周期
  - **Docker**：容器化平台，PM2 可以在容器内运行
  - **K8s**：容器编排平台，`pm2-runtime` 可在 K8s Pod 中使用
- **用 PM2 而不是 K8s 的场景**：
  1. **简单单机部署**：单台服务器运行 Node.js 应用，不需要集群
  2. **快速迭代开发**：开发/测试环境，无需完整 K8s 集群开销
  3. **PM2 Plus 监控**：PM2 自带 APM（应用性能监控），开箱即用
  4. **混合语言环境**：同一机器上跑 Node.js + Python + Go，PM2 只管 Node.js
- **K8s 替代 PM2 的场景**：
  - 多副本、弹性扩缩容需求
  - 需要跨主机服务发现和负载均衡
  - 需要完整的 CI/CD 集成（Helm/ArgoCD）
  - 混合微服务架构（Node.js + Java + Python 统一管理）

**可能的追问**：
- `pm2-runtime` 在 K8s 中的使用场景是什么？
- PM2 的 cluster 模式和 Docker 的 `--replicas` 有什么区别？
- K8s 中 Node.js 应用的 graceful shutdown 如何实现？

---

## 6.4 Kubernetes

### Q13：K8s 的核心组件（Pod、Service、Deployment、Ingress）各自的作用？

**考察点**：K8s 核心对象、抽象层级

**答案要点**：
- **核心组件**：
  - **Pod**：K8s 最小调度单位，一个或多个容器（共享 network/Linux namespace），共享 IP 和端口空间
  - **Deployment**：声明 Pod 的期望状态（副本数、镜像版本、更新策略），自动管理 ReplicaSet，实现滚动更新和回滚
  - **Service**：为一组 Pod 提供稳定的访问入口（ClusterIP/NodePort/LoadBalancer），通过 label selector 关联 Pod，提供服务发现和负载均衡
  - **Ingress**：HTTP/HTTPS 路由，将外部请求路由到集群内 Service，支持基于域名/路径的路由规则、SSL 终结
- **层级关系**：Deployment → ReplicaSet → Pod → Container
- **对比**：
  - Deployment vs ReplicaSet：Deployment 是更高层抽象，管理 ReplicaSet；直接操作 ReplicaSet 不常见
  - Service vs Ingress：Ingress 处理 HTTP 层（7层），Service 处理 TCP/UDP 层（4层）

**可能的追问**：
- Pod 的网络命名空间共享具体指什么？（network namespace 共享，PID namespace 可选）
- Init Container 和 Sidecar Container 的区别？
- 什么是 Pod 的 `restartPolicy`？为什么 Web 服务通常用 Always？

---

### Q14：K8s 的 Service 类型（ClusterIP、NodePort、LoadBalancer）区别？Ingress 和 Service 的关系？

**考察点**：Service 类型、流量入口

**答案要点**：
- **三种 Service 类型对比**：
  | 类型 | 访问范围 | 外部流量 | 典型场景 |
  |------|---------|---------|---------|
  | **ClusterIP** | 仅集群内部 | ❌ | 集群内部微服务间通信 |
  | **NodePort** | 集群 + 所有节点端口 | ✅（节点:Port） | 开发/测试，固定端口 |
  | **LoadBalancer** | 集群 + LB | ✅（云厂商 LB） | 生产环境，对接云负载均衡器 |
  | **Headless** | 集群内部 | ❌ | 无 LB 时直接获取 Pod IP |
- **Ingress vs Service**：
  - Service 是 4 层（TCP/UDP），Ingress 是 7 层（HTTP/HTTPS）
  - Ingress 依赖 Service 作为后端（通常是指向 Deployment 的 Service）
  - Ingress Controller（如 ingress-nginx/Traefik）负责解析 Ingress 规则
- **SOC 平台典型配置**：
  ```yaml
  # Ingress
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: soc-api
  spec:
    rules:
    - host: api.soc.com
      http:
        paths:
        - path: /
          backend:
            service:
              name: soc-api-service  # → Service → Pods
              port: { number: 80 }
  ```

**可能的追问**：
- Headless Service 在什么场景下使用？（StatefulSet 的稳定网络标识）
- Ingress 和 Gateway API（K8s 1.19+ GA）有什么关系？
- ExternalTrafficPolicy: Local vs Cluster 的区别？（Local 保留客户端 IP，Cluster 不保留）

---

### Q15：你们在迁移 TCE（腾讯云企业版）过程中，K8s 的资源配额（ResourceQuota）和限制（LimitRange）是如何规划的？

**考察点**：K8s 资源管理、Quota 规划

**答案要点**：
- **ResourceQuota（命名空间级别总配额）**：
  ```yaml
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: soc-quota
  spec:
    hard:
      requests.cpu: "20"      # 总 CPU 请求上限
      requests.memory: 40Gi   # 总内存请求上限
      limits.cpu: "40"        # 总 CPU 限制上限
      limits.memory: 80Gi     # 总内存限制上限
      pods: "100"             # Pod 总数上限
      services: "20"          # Service 总数上限
  ```
- **LimitRange（单容器默认限制）**：
  ```yaml
  kind: LimitRange
  metadata:
    name: soc-limits
  spec:
    limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 200m
        memory: 256Mi
      max:
        cpu: "4"
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
  ```
- **规划原则**：
  - 预留 20-30% buffer 应对突发
  - 生产/测试/开发命名空间分离，各有独立 Quota
  - CPU request = 日常负载，CPU limit = 突发上限
  - 内存一般不设上限太大（OOMKill）

**可能的追问**：
- `requests.cpu` 的 `m`（millicores）是什么意思？`0.5` 和 `500m` 等价吗？
- LimitRange 的 `maxLimitRequestRatio` 有什么用？
- GPU 资源如何声明？（`nvidia.com/gpu: 1`，需安装 Device Plugin）

---

### Q16：K8s 的 HPA（水平自动扩缩容）的工作原理？基于什么指标扩缩容？在 SOC 平台的告警高峰期如何应对？

**考察点**：HPA 机制、扩缩容策略

**答案要点**：
- **HPA 工作原理**：
  1. HPA Controller（kube-controller-manager 内）定期查询指标（默认 15s）
  2. 指标来源：`metrics-server`（资源指标）或 Prometheus Adapter（自定义指标）
  3. 计算：desiredReplicas = ceil(sum(currentMetrics) / targetMetric × currentReplicas)
  4. 通过 Deployment/ReplicaSet 调整 Pod 数量
- **扩缩容指标**：
  - **CPU 利用率**：`type: Resource, resource.name: cpu, resource.targetAverageUtilization: 70`
  - **内存利用率**：`type: Resource, resource.name: memory, resource.targetAverageValue: 1Gi`
  - **自定义指标**：Prometheus 查询（HPA adapter），如**告警队列积压数**、**HTTP 请求延迟 P99**
- **SOC 告警高峰期应对**：
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: soc-worker-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: soc-worker
    minReplicas: 2
    maxReplicas: 10
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: alert_queue_depth
        target:
          type: AverageValue
          averageValue: "100"  # 队列积压 > 100 则扩容
  ```
- **KEDA**：更高级的事件驱动扩缩容，可基于 Kafka lag、Redis Stream 积压量扩缩容

**可能的追问**：
- HPA 的 stabilizationWindow 是什么？如何防止频繁抖动？
- 什么是 VPA（Vertical Pod Autoscaler）？它和 HPA 的区别？
- HPA 在缩容到 0 之后，如何保证冷启动延迟？（配置 `scaleDownDelayAfterFailure`）

---

### Q17：K8s 的 ConfigMap 和 Secret 有什么区别？如何在 Pod 中使用环境变量和 volume 挂载？

**考察点**：ConfigMap/Secret 使用方式

**答案要点**：
- **ConfigMap vs Secret**：
  | 维度 | ConfigMap | Secret |
  |------|----------|--------|
  | 用途 | 非敏感配置（JSON/YAML/env 文件） | 敏感数据（密码/API Key/TLS 证书） |
  | 编码 | 明文存储 | Base64 编码（不加密！需配合 EncryptionConfiguration） |
  | 加密 | 无 | 可加密（KMS/Sealed Secrets） |
  | 类型 | 键值对/文件/命令行参数 | Opaque/docker-registry/tls |
- **Pod 中使用方式**：
  ```yaml
  # 环境变量方式
  envFrom:
  - configMapRef:
      name: app-config     # 整个 ConfigMap 注入为环境变量
  - secretRef:
      name: api-keys       # Secret 注入

  env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log.level

  # volume 挂载方式（配置文件）
  volumes:
  - name: config
    configMap:
      name: app-config
      items:
      - key: nginx.conf
        path: nginx.conf
  - name: secrets
    secret:
      secretName: api-keys
  volumeMounts:
  - name: config
    mountPath: /etc/app/config
  - name: secrets
    mountPath: /etc/secrets
    readOnly: true
  ```

**可能的追问**：
- Secret 的 Base64 编码是加密吗？如何真正加密 Secret？
- 什么是 Sealed Secrets？它和普通 Secret 有什么区别？
- ConfigMap 变更后，Pod 内挂载的文件如何更新？（需要重启 Pod，或用 Reloader 工具）

---

### Q18：K8s 的 PV（PersistentVolume）和 PVC（PersistentVolumeClaim）是什么？动态 provisioning 的原理？

**考察点**：K8s 存储抽象、StorageClass

**答案要点**：
- **PV vs PVC**：
  - **PV**：集群层面的存储资源（类似"硬盘"），由管理员创建或动态供给
  - **PVC**：Pod 对存储的请求（类似"挂载请求单"），声明需要的存储大小和访问模式（ReadWriteOnce/ReadOnlyMany/ReadWriteMany）
- **生命周期**：
  - PV 和 PVC 是一对一绑定（ClaimRef）
  - PV 的 Reclaim Policy：`Retain`（保留）/ `Delete`（删除底层存储）/ `Recycle`（删除数据后重用）
- **动态 provisioning**：
  - **StorageClass**：定义存储类型和 provisioner（存储插件）
  - Pod 创建时声明 PVC → StorageClass 选择对应 Provisioner → Provisioner 在云存储/网络存储上动态创建 PV
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: soc-fast-storage
  provisioner: disk.eks.amazonaws.com  # 云厂商 provisioner
  parameters:
    type: gp3
    encrypted: "true"
  reclaimPolicy: Retain
  volumeBindingMode: WaitForFirstConsumer  # 延迟绑定，等 Pod 调度后再创建 PV
  ```
- **SOC 平台场景**：MySQL 数据用持久化存储（StorageClass: `managed-premium`），告警 ES 数据用本地 NVMe（Local PV）

**可能的追问**：
- `volumeBindingMode: Immediate` vs `WaitForFirstConsumer` 的区别？
- CSI（Container Storage Interface）是什么？它和 in-tree plugin 有什么区别？
- ReadWriteOnce、ReadOnlyMany、ReadWriteMany 各支持什么类型的存储？

---

### Q19：Git 的 `rebase` 和 `merge` 的区别？什么场景用 rebase？rebase 的风险是什么？

**考察点**：Git 历史管理、分支策略

**答案要点**：
- **merge vs rebase**：
  - **merge**：将两个分支的历史**合并**，生成一个新的合并 commit（merge commit），保留完整历史分支结构
  - **rebase**：将当前分支的提交"移植"到目标分支的顶部，重写历史提交（commit SHA 全部变化）
- **rebase 使用场景**：
  - 合并 `feature` 到 `main` 之前：先 rebase main，保持线性历史
  - 整理本地提交（交互式 rebase：`git rebase -i`）：合并 commit、修改提交信息、删除无用提交
- **merge 使用场景**：
  - 合并长期分支（release/hotfix）
  - 合并到共享分支（避免重写他分支基于的历史）
- **rebase 的风险**：
  1. **重写历史**：push 到远程后 force push，会影响其他协作者
  2. **冲突处理复杂**：每个提交都要解决冲突，比 merge 更繁琐
  3. **无法恢复**：rebase 后旧的提交不可见（除非 reflog）
- **黄金法则**：**不要 rebase 已经 push 到远程的提交**；本地私人分支可自由 rebase

**可能的追问**：
- 什么是 squash merge？它和 rebase 有什么区别？
- `git rebase --onto` 的用法是什么？
- 如何撤销一个 rebase？（`git reflog` + `git reset --hard`）

---

### Q20：如何设计一个高可用的部署架构？（多活、灾备、故障切换）

**考察点**：高可用架构设计、容灾策略

**答案要点**：
- **高可用部署架构层级**：
  1. **前端层**：
     - 多 CDN 节点 + Anycast DNS
     - 浏览器端：多域名静态资源、域名预解析
  2. **负载均衡层**：
     - 双 LB（主备或-active/active）：Nginx / HAProxy / 云 SLB
     - 健康检查：定期探测后端实例，自动摘除故障节点
  3. **应用层**：
     - K8s 多副本 + 跨 AZ（可用区）调度
     - 最小 2 副本，分布在不同 AZ
  4. **数据层**：
     - **MySQL**：主从半同步复制 + 自动选主（RTO < 30s）
     - **Redis**：Cluster 3 主 3 从，跨 AZ 部署
     - **Kafka**：多 Broker + 多副本，ISR（In-Sync Replicas）>= 2
     - **ES**：跨 AZ 分片 + 副本，任意一个 AZ 挂了仍可服务
- **容灾切换策略**：
  - **同城双活**：两个机房同时服务，同城延迟 < 1ms，流量 1:1 分担
  - **异地灾备**：冷备 Warm Standby，故障时 DNS 切换（RTO 分钟级）
  - **RTO / RPO 指标**：SOC 平台要求 RTO < 5min，RPO < 1min（Kafka + MySQL 半同步保证）
- **故障自动处理**：
  - K8s LivenessProbe：容器不健康自动重启
  - HPA：流量突增自动扩容
  - 熔断降级：Sentinel/Hystrix，依赖服务故障时返回降级数据

**可能的追问**：
- 什么是"多活"？同城双活和异地多活有什么区别？
- K8s 的 Pod Disruption Budget（PDB）是什么？它如何防止主动驱逐时服务中断？
- 如何设计跨 AZ 的数据库连接池？（多地址连接、连接时分发到最近 AZ）
