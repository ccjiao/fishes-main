## 概述
- 目标：将本仓库的前端静态站点以容器方式在公司内网 NAS 上稳定部署，满足非 root 运行、网络与存储规范、健康监控、日志与资源控制、安全与文档要求。
- 仓库特征（调研）：本仓库不含后端代码与构建脚本，乃纯静态前端。依赖外部后端与云服务：
  - 后端基址切换：`src/js/fish-utils.js` 27–36 行（本地 `localhost:8080` 或 Cloud Run）
  - 模型文件：站点根加载 `fish_doodle_classifier.onnx`（`src/js/app.js` 533–541 行）
  - Firebase 初始化与持久化：`src/js/firebase-init.js` 11–20、16–28 行
  - 登录与认证端点：`src/js/login.js` 170–181、196–203、224–231、248–268 行
- 部署方式：使用非 root 的 Nginx 镜像提供静态站点；NAS 侧提供持久化卷与网络；通过反向代理统一对内/对外端口与 TLS。

## 交付物
1. `docker/Dockerfile`
2. `docker/nginx.conf`
3. `docker-compose.yml`
4. `config/env-config.js.example`（运行时外置配置示例）
5. 文档：`docs/deploy.md`、`docs/ops.md`、`docs/troubleshooting.md`

## 实施方案
### Dockerfile（非 root，健康检查）
- 基于 `nginxinc/nginx-unprivileged:alpine`（默认非 root，监听 8080）。
- 复制站点静态资源到 `/usr/share/nginx/html`。
- 注入 `nginx.conf` 到 `/etc/nginx/nginx.conf`。
- `HEALTHCHECK` 通过 `wget` 检查 `index.html` 并验证模型文件存在。

示例：
```
FROM nginxinc/nginx-unprivileged:alpine
COPY . /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/nginx.conf
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD wget -q --spider http://localhost:8080/index.html && test -f /usr/share/nginx/html/fish_doodle_classifier.onnx || exit 1
EXPOSE 8080
```

### Nginx 配置（监听 8080，日志与健康）
- 站点根：`/usr/share/nginx/html`；默认页 `index.html`；前端路由兼容 `try_files`。
- 访问日志与错误日志输出到 `/var/log/nginx`（由卷映射到 NAS）。
- 提供健康检查路径 `/health` 返回 200。
- 可选暴露 `stub_status` 到 `/nginx_status` 供监控采集。

示例：
```
worker_processes auto;
events { worker_connections 1024; }
http {
    include /etc/nginx/mime.types;
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;
    server {
        listen 8080;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;
        location / { try_files $uri $uri/ /index.html; }
        location = /health { return 200; }
        location /nginx_status { stub_status; }
        types { application/octet-stream onnx; }
    }
}
```

### docker-compose.yml（网络、卷、健康、日志、资源）
- 连接到公司内网专用网络 `corp_net`（external）。如 NAS 已创建该网络，直接引用；否则提供网络创建说明。
- 卷：
  - 将站点文件与模型映射为只读（NAS 指定目录）。
  - 将 Nginx 日志目录映射为读写（NAS 指定目录）。
  - 外置运行时配置 `env-config.js`（避免镜像含敏感信息）。
- 健康检查与日志驱动按规范设置；资源限制使用 `deploy.resources`（如非 Swarm，将在文档中提供替代方案）。

示例：
```
version: "3.8"
services:
  fishes-web:
    build:
      context: .
      dockerfile: docker/Dockerfile
    image: fishes-web:latest
    container_name: fishes-web
    ports:
      - "8080:8080"
    networks:
      corp_net:
        aliases:
          - fishes-web
    volumes:
      - /mnt/nas/fishes/html:/usr/share/nginx/html:ro
      - /mnt/nas/fishes/logs:/var/log/nginx
      - /mnt/nas/fishes/config/env-config.js:/usr/share/nginx/html/env-config.js:ro
    healthcheck:
      test: ["CMD-SHELL","wget -q --spider http://localhost:8080/health || exit 1"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: "512M"
        reservations:
          cpus: "0.25"
          memory: "128M"
networks:
  corp_net:
    external: true
```

### 非 root 运行
- 镜像默认以非 root 用户运行并监听 8080；NAS 入口代理将 80/443 映射到容器 8080/8443（如需）。
- 若 NAS 对卷有 UID/GID 要求：在文档中给出 `chown` 与目录预创建方法，确保容器内非 root 用户对 `/var/log/nginx` 有写权限。

### 网络与存储
- 网络：加入 `corp_net`，通过 NAS 反向代理对外统一暴露，内部流量走专用网段。
- 存储：
  - 站点内容与模型：只读映射，避免篡改。
  - 日志：读写映射，便于持久化与审计。
  - 运行时配置 `env-config.js`：只读映射，集中管理敏感配置。

### 权限处理（NAS 与容器之间）
- 建议在 NAS 上预创建目录并设置与容器用户匹配的 UID/GID（文档提供具体命令）。
- 如需强制容器侧修复权限：在 `command` 中执行一次 `chown` 后启动 Nginx（文档作为备选方案，不默认启用）。

### 健康检查与监控
- 健康：`/health` 路径直接由 Nginx返回 200；同时 Dockerfile 与 Compose 均配置健康检查。
- 监控：启用 `stub_status`；文档内提供使用 Prometheus Nginx Exporter 采集的示例 Compose 片段。

### 日志与资源限制
- 日志：`access.log` 与 `error.log` 写入 `/var/log/nginx`，滚动策略使用容器日志驱动配置。
- 资源：Compose 标准 `deploy.resources`；若 NAS 不启用 Swarm，文档提供以运行参数或 NAS 面板方式设置 CPU/内存的替代方案。

### 安全与敏感信息
- 容器间加密：
  - 如仅本前端容器：对外服务由 NAS 统一 TLS 终止；容器到外部云服务使用 HTTPS。
  - 如新增后端容器：提供 Traefik/Caddy 作为内部 mTLS 或 TLS 代理的可选方案与 Compose 增量片段。
- 敏感信息管理：
  - 将 `BACKEND_URL`、`GOOGLE_CLIENT_ID`、`FIREBASE_CONFIG` 等外置到 `env-config.js` 并通过卷挂载，镜像不含敏感信息。
  - 如使用 Swarm：提供 `secrets` 管理方案；非 Swarm：通过 NAS 权限与只读映射保护配置文件。

## 文档编写
- `docs/deploy.md`：环境前置、网络与卷准备、构建与启动、反向代理与 TLS、资源限制设置、回滚与升级。
- `docs/ops.md`：日志查看与轮转、健康状态与监控、目录权限调整、常规重启与滚动更新。
- `docs/troubleshooting.md`：网络不可达、权限写入失败、健康检查失败、模型文件缺失、外部依赖（Cloud Run/Firebase/OAuth）连通性问题排查。

## 测试验证
- 测试环境：在隔离网络下以相同 Compose 启动，验证：
  - 站点可访问并能加载 `fish_doodle_classifier.onnx`。
  - 与后端交互成功（`src/js/app.js` 127–133 行上传、`src/js/fish-utils.js` 各接口）。
  - Firebase 持久化与登录流程可用。
  - 健康检查与日志输出正常，监控端点可采集。
- 备份恢复：日志与 `env-config.js` 卷的备份脚本与恢复步骤（文档提供示例）。

## 风险与取舍
- Compose `deploy.resources` 在非 Swarm 环境可能被忽略：文档将给出 NAS 面板或运行参数替代。
- 卷权限取决于 NAS 的导出策略与 UID/GID 映射：优先在 NAS 侧对齐 UID/GID，避免容器侧频繁 `chown`。
- 现有代码中部分配置写死：通过新增 `env-config.js` 外置运行时配置以满足“镜像不含敏感信息”。

## 下一步
- 确认以上方案后：
  - 编写与提交 Dockerfile、Nginx 配置与 Compose 文件。
  - 增加 `env-config.js` 加载顺序（在各 HTML 文件中引入），并将相关配置从源码移至运行时配置。
  - 补充文档与在测试环境完成端到端验证。