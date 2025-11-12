# 运维手册

## 日志
- 容器日志驱动：`json-file`，支持大小与文件数滚动
- Nginx 文件日志：`/mnt/nas/fishes/logs`
- 查看：
```
docker compose logs -f fishes-web
```

## 健康检查与监控
- 健康路径：`/health`（200）
- Docker 健康检查：已配置 `wget` 检测
- Nginx 监控：`/nginx_status`（可被 Prometheus Nginx Exporter 采集）

## 常用操作
- 重启容器：
```
docker compose restart fishes-web
```
- 查看容器资源占用：
```
docker stats fishes-web
```
- 调整运行时配置：更新 NAS 上的 `env-config.js` 后无需重建镜像

## 目录结构
- `/mnt/nas/fishes/html`：静态文件与模型（只读）
- `/mnt/nas/fishes/logs`：访问与错误日志（读写）
- `/mnt/nas/fishes/config/env-config.js`：运行时配置（只读）

