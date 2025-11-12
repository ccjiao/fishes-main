# 故障排查指南

## 无法访问站点
- 检查容器是否运行：`docker compose ps`
- 检查健康检查：`docker inspect --format='{{.State.Health.Status}}' fishes-web`
- 查看日志：`docker compose logs fishes-web`

## 模型文件加载失败
- 确认 `fish_doodle_classifier.onnx` 存在于站点根目录并可访问
- 检查 NAS 映射是否为只读且路径正确

## 后端接口错误
- 验证 `env-config.js` 中的 `BACKEND_URL`
- 确认后端地址可达，使用 curl 测试 `BACKEND_URL/api/fish`
- 检查浏览器控制台与网络请求详情

## 权限写入失败
- Nginx 日志不可写：调整 NAS 目录 `logs` 的所有者与权限
- 验证容器用户非 root，避免 80/443 绑定导致权限问题

## 资源限制未生效
- 在非 Swarm 环境下 `deploy.resources` 可能被忽略
- 使用 NAS 面板设置或以运行参数限制 CPU/内存

## 外部依赖连通性
- Cloud Run 后端、Firebase 与 Google OAuth 均需外网访问
- NAS 环境需开放到对应服务的出站访问与 DNS 解析

