# 部署手册（NAS 内网 Docker 环境）

## 前置条件
- 已安装 Docker 与 Docker Compose
- NAS 已配置专用网络 `corp_net`，或具备创建网络权限
- 预创建目录：
  - `/mnt/nas/fishes/html` 用于站点与模型文件（只读）
  - `/mnt/nas/fishes/logs` 用于 Nginx 日志（读写）
  - `/mnt/nas/fishes/config/env-config.js` 运行时配置（只读）

## 运行时配置
- 将 `config/env-config.js.example` 复制到 NAS：
```
cp config/env-config.js.example /mnt/nas/fishes/config/env-config.js
```
- 根据环境修改其中的 `BACKEND_URL`、`GOOGLE_CLIENT_ID`、`FIREBASE_CONFIG`

## 网络准备
- 如 NAS 已有 `corp_net`：跳过
- 如需创建：
```
docker network create corp_net
```

## 构建与启动
- 在项目根目录执行：
```
docker compose build
docker compose up -d
```
- 访问：`http://<NAS主机>:8080`

## 反向代理与 TLS
- 推荐由 NAS 入口代理统一处理 80/443 与 TLS 证书
- 将代理的 80/443 映射到容器 `8080`，内部容器保持非 root 监听 8080

## 权限与 UID/GID
- 确保 `/mnt/nas/fishes/logs` 的所有者与权限允许容器非 root 用户写入
- 如需调整：
```
sudo chown -R <uid>:<gid> /mnt/nas/fishes/logs
sudo chmod -R 0755 /mnt/nas/fishes/logs
```

## 资源限制
- Compose 文件已配置 `deploy.resources`，如 NAS 非 Swarm，可在 NAS 面板或运行参数设置 CPU/内存限制

## 升级与回滚
- 升级：
```
docker compose pull
docker compose up -d --no-deps fishes-web
```
- 回滚：保留历史镜像或使用标签策略，`docker compose up -d` 指定旧标签

## 飞牛OS（FeiNiu OS）图形界面部署指南
- 准备共享目录（通过飞牛OS文件管理器）：在某个共享文件夹下创建 `fishes/html`、`fishes/logs`、`fishes/config` 三个子目录，并将 `config/env-config.js.example` 复制为 `fishes/config/env-config.js` 后按需修改。
- 打开飞牛OS的 Docker/Compose 图形界面，新建项目：
  - 方式一：上传仓库中的 `docker-compose.yml`
  - 方式二：复制 `docker-compose.yml` 内容并在 GUI 编辑器中粘贴
  - 若仅在 NAS 本地构建且不从仓库拉取镜像，确保 Compose 中保留 `build:` 并移除 `image:` 字段（仓库已移除），避免 GUI 执行“Pulling”
- 网络设置：
  - 若系统已存在自定义网络 `corp_net`，选择该网络或保留 Compose 中的 `external: true`
  - 若 GUI 不支持引用 external 网络，可在系统 Docker/网络页面先创建 `corp_net`；或删除 Compose 文件中的 `networks` 段以使用默认网络
- 卷映射（在 GUI 中将容器内路径映射到共享文件夹）：
  - 将共享目录 `fishes/html` 映射到容器路径 `/usr/share/nginx/html`（设置为只读）
  - 将共享目录 `fishes/logs` 映射到容器路径 `/var/log/nginx`（读写）
  - 将 `fishes/config/env-config.js` 映射到容器路径 `/usr/share/nginx/html/env-config.js`（只读）
- 端口映射：
  - 将容器端口 `8080` 映射到 NAS 端口（如 `8080:8080`）；如使用飞牛OS反向代理或站点入口，保持容器监听 `8080`，在代理层配置域名与 TLS 并转发到该端口
- 资源限制：在项目的“高级设置/资源限制”中设置 vCPU 与内存（GUI 支持时）；若未启用 Swarm，Compose 中的 `deploy.resources` 可能不生效
- 启动与验证：
  - 启动项目后访问 `http://<NAS_IP>:8080` 检查首页与模型加载
  - 访问 `http://<NAS_IP>:8080/health` 查看返回 200 状态
  - 在 GUI 的日志界面查看容器与 Nginx 文件日志（位于共享目录 `fishes/logs`）
- 权限处理：
  - 若 `fishes/logs` 写入失败，在飞牛OS的共享文件夹权限界面为该目录赋予写权限；或使用命令行将其权限调整为可写（确保仍遵循最小权限原则）
- 升级与回滚（GUI）：
  - 在项目界面执行“重新部署/更新镜像”，完成后检查健康状态
  - 如需回滚，使用上一版本镜像标签或恢复上一次成功配置后重新部署
  - 如使用本地构建方案，重新部署会基于最新源码重新构建；基础镜像（`nginxinc/nginx-unprivileged:alpine`）会在 NAS 上自动拉取

### Compose（默认网络版）备选片段
当飞牛OS GUI不支持 external 网络时，使用下列片段（移除 `networks` 段）：
```
services:
  fishes-web:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: fishes-web
    ports:
      - "8080:8080"
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
```

### GitHub 远程构建（私有仓库）
- 将 NAS 的 SSH 公钥添加为 GitHub 仓库的只读 Deploy Key 或绑定到你的账号
- 在 Compose 中使用 SSH 上下文并开启 SSH 转发：
  - `build.context: git@github.com:ccjiao/fishes-main.git#main`
  - `build.ssh: ["default"]`
- 要求启用 BuildKit；如 GUI 未启用，在 Docker 设置中开启或设置环境变量 `DOCKER_BUILDKIT=1`
- 测试 SSH：在 NAS 终端运行 `ssh -T git@github.com` 应显示成功握手
- 注意：不要在 Compose 中嵌入访问令牌；如必须使用 HTTPS+PAT，请改为在 NAS 上预先配置凭据并避免将令牌写入 Compose 文件

### GitHub 公共仓库构建（HTTPS）
- 公共仓库无需凭据，直接使用：
  - `build.context: https://github.com/ccjiao/fishes-main.git#main`
  - `build.dockerfile: docker/Dockerfile`
- 仍需启用 BuildKit 以支持远程上下文；确保 NAS 能访问 `github.com:443`
- 若 GUI 报告无法解析远程上下文：
  - 检查是否启用 BuildKit
  - 确认 DNS 与出站策略未拦截 GitHub

### GitHub 远程构建（Tarball 模式，避开 Git/HTTP2）
- 当遇到 `curl 16 Error in the HTTP2 framing layer` 或 `fatal: expected flush after ref listing` 时，改用 tarball：
  - `build.context: https://github.com/ccjiao/fishes-main/archive/refs/heads/main.tar.gz`
  - `build.dockerfile: docker/Dockerfile`
- 该模式直接下载压缩包，避免在构建器内使用 Git 协议，常用于企业网络中间设备导致的 HTTP/2 问题
