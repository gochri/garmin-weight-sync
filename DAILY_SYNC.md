# Daily Sync 使用指南

`daily_sync` 是一个持续运行的 Docker 容器，用于自动每日同步体重数据到 Garmin Connect。

## 前置

### 构建镜像

如果项目更新了代码（如新增 `daily_sync.py`），需要本地构建镜像：

```bash
docker-compose build --no-cache
```

构建完成后重新启动容器即可使用最新代码。

### 删除镜像

删除本地构建的镜像：

```bash
# 删除 daily_sync 相关的镜像
docker-compose down daily_sync
docker image rm lesliehwang/garmin-weight-sync:latest

# 或重新构建时覆盖旧镜像
docker-compose build --no-cache
```

## 快速开始

### 启动容器

```bash
docker-compose up -d daily_sync
```

### 查看状态

```bash
docker-compose ps
```

### 查看日志

```bash
# 实时日志
docker-compose logs -f daily_sync

# 最后 100 行
docker-compose logs -n 100 daily_sync

# 保存日志到文件
docker-compose logs daily_sync > sync_logs.txt
```

## 工作原理

1. **启动延迟**：容器启动后会延迟 5 分钟（可自定义）
2. **首次同步**：延迟后执行第一次同步
3. **周期执行**：之后每 24 小时（可自定义）执行一次同步
4. **错误处理**：同步失败时会在 1 小时后自动重试
5. **自动重启**：配置了 `restart: unless-stopped`，容器意外停止时会自动重启

## 配置参数

### Docker Compose 配置

编辑 `docker-compose.yml` 中的 `daily_sync` 服务的 `command` 字段：

```yaml
command: ["python", "src/daily_sync.py", "--config", "config/users.json", "--sync", "--delay", "300", "--interval", "86400"]
```

### 参数说明

| 参数 | 说明 | 示例 | 默认值 |
|------|------|------|--------|
| `--config` | 配置文件路径 | `config/users.json` | `config/users.json` |
| `--sync` | 启用 Garmin 上传 | （无值，仅作为标志） | 不启用 |
| `--delay` | 启动延迟（秒） | `600` 表示 10 分钟 | `300`（5 分钟） |
| `--interval` | 同步间隔（秒） | `43200` 表示 12 小时 | `86400`（24 小时） |

### 常见配置示例

#### 每 6 小时同步一次

```yaml
command: ["python", "src/daily_sync.py", "--config", "config/users.json", "--sync", "--delay", "300", "--interval", "21600"]
```

#### 每 12 小时同步一次，延迟 10 分钟

```yaml
command: ["python", "src/daily_sync.py", "--config", "config/users.json", "--sync", "--delay", "600", "--interval", "43200"]
```

#### 启动后立即同步

```yaml
command: ["python", "src/daily_sync.py", "--config", "config/users.json", "--sync", "--delay", "0", "--interval", "86400"]
```

## 日志查看

### 实时日志

```bash
docker-compose logs -f daily_sync
```

输出示例：

```
2024-05-12 10:30:45,123 - root - INFO - Daily Sync Runner started with config: config/users.json
2024-05-12 10:30:45,124 - root - INFO - Initial delay: 300s, Sync interval: 86400s
2024-05-12 10:30:45,125 - root - INFO - Sync to Garmin: enabled
2024-05-12 10:30:45,126 - root - INFO - Delaying start for 300 seconds...
2024-05-12 10:35:45,456 - root - INFO - Starting sync with config: config/users.json
2024-05-12 10:35:50,789 - root - INFO - Sync completed at 2024-05-12 10:35:50.789123
2024-05-12 10:35:50,890 - root - INFO - Sync successful
2024-05-12 10:35:50,891 - root - INFO - Next sync scheduled for 2024-05-13 10:35:50.891123
```

### 历史日志

```bash
# 查看最后 50 行
docker-compose logs -n 50 daily_sync

# 查看特定时间段的日志（需要手动保存）
docker-compose logs daily_sync > /tmp/full_logs.txt
grep "2024-05-12 10:" /tmp/full_logs.txt
```

## 管理操作

### 启动容器

```bash
docker-compose up -d daily_sync
```

### 停止容器（暂停）

```bash
docker-compose stop daily_sync
```

### 启动已停止的容器

```bash
docker-compose start daily_sync
```

### 重启容器

```bash
docker-compose restart daily_sync
```

### 删除容器

```bash
docker-compose down daily_sync
```

### 查看容器详细信息

```bash
docker-compose ps daily_sync
```

## 故障排查

### 容器无法启动

**症状**：运行 `docker-compose up -d daily_sync` 后没有响应，或提示 "can't open file '/app/src/daily_sync.py'"

**解决方案**：
1. 查看错误日志：`docker-compose logs daily_sync`
2. 如果提示找不到 `daily_sync.py`，需要本地构建：`docker-compose build --no-cache`
3. 确保配置文件存在：`config/users.json` 是否存在
4. 尝试重建：`docker-compose up -d --build daily_sync`

### 同步始终失败

**症状**：日志显示 "Sync failed"

**解决方案**：
1. 检查用户配置：`cat config/users.json`
2. 验证 token 有效性（检查 token 是否已过期）
3. 重新登录获取新 token：`docker-compose --profile login run --rm login`
4. 检查网络连接和防火墙设置

### 容器频繁重启

**症状**：`docker-compose ps` 显示容器状态不稳定

**解决方案**：
1. 查看详细日志：`docker-compose logs -n 100 daily_sync`
2. 检查是否有异常错误
3. 如果是因为配置问题，修复配置后重新启动

### 日志过多导致磁盘占满

**症状**：Docker 日志占用大量磁盘空间

**解决方案**：
```bash
# 清理 Docker 日志
docker system prune --volumes

# 限制日志大小（编辑 docker-compose.yml）
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

## 与其他同步方式的比较

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **daily_sync 容器** | 无需外部工具，自动重启，完整日志 | 需要 Docker | 长期持续运行 |
| **一次性同步** | 简单快速，手动控制 | 需要每次手动运行 | 测试、临时使用 |
| **Windows 任务计划** | 系统集成，无需 Docker | 需要管理员权限 | Windows 专用 |
| **Linux crontab** | 系统原生，轻量级 | 需要 crontab 知识 | Linux 专用 |

## 性能注意事项

- **CPU**：每次同步占用 100-200M 内存，CPU 占用较低
- **网络**：每次同步需要上下行网络，建议在网络稳定时间段运行
- **磁盘**：FIT 文件和备份数据存储在 `data/` 目录
- **日志**：日志会持续写入，建议定期检查磁盘空间

## 升级和更新

### 本地构建（项目更新）

当项目新增功能或文件时，需要本地构建新镜像：

```bash
docker-compose build --no-cache
docker-compose restart daily_sync
```

### 拉取最新镜像

从 Docker Hub 更新预构建镜像：

```bash
docker-compose pull daily_sync
docker-compose up -d daily_sync
```

### 清理镜像

删除不使用的旧镜像释放磁盘空间：

```bash
# 删除所有未使用的镜像
docker image prune -a

# 删除特定镜像
docker image rm lesliehwang/garmin-weight-sync:latest
```

### 版本回滚

```bash
# 查看镜像历史
docker image history lesliehwang/garmin-weight-sync

# 指定特定版本
# 编辑 docker-compose.yml，修改 image 为特定版本标签
```

## 相关文档

- [Docker 部署指南](DOCKER_SETUP.md) - 完整的 Docker 使用说明
- [项目 README](README.md) - 项目概述
- [使用说明](USAGE.md) - 功能和特性说明
