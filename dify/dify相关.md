# Dify 本地部署指南

本文介绍如何通过 Docker Compose 在 Windows 或 Linux/Ubuntu 环境中部署 Dify 社区版。

## 1. 环境要求

Dify 官方最低配置：

- CPU：2 核及以上
- 内存：4 GiB 及以上
- Docker Compose：2.24.0 及以上

个人体验或测试环境建议使用 4 核 CPU、8 GB 内存；生产环境应根据并发量和知识库规模增加资源。

需要提前安装：

- Git
- Docker
- Docker Compose

Windows 用户推荐安装 Docker Desktop，并启用 WSL2 后端。

## 2. Windows 本机部署

启动 Docker Desktop，然后打开 PowerShell，执行：

```powershell
git clone https://github.com/langgenius/dify.git
cd dify\docker

Copy-Item .env.example .env
docker compose up -d
```

首次启动需要下载多个镜像，耗时取决于网络速度。

检查容器状态：

```powershell
docker compose ps
```

查看实时日志：

```powershell
docker compose logs -f
```

所有主要服务启动完成后，在浏览器打开：

```text
http://localhost/install
```

按照页面提示创建管理员账号。

## 3. Linux / Ubuntu 部署

确认 Docker 服务已启动，然后执行：

```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker

cp .env.example .env
docker compose up -d
```

检查服务状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f
```

在浏览器中访问：

```text
http://服务器IP/install
```

如果无法访问，请检查服务器防火墙、安全组以及 80 端口是否开放。

## 4. 配置环境变量

Dify 的主要配置文件为：

```text
dify/docker/.env
```

默认情况下，从 `.env.example` 复制生成即可启动。需要自定义时，可在启动前修改 `.env`。

常见配置包括：

- 对外访问地址
- 数据库账号和密码
- Redis 密码
- 文件存储方式
- 向量数据库类型
- 日志级别
- CORS 设置

修改配置后重新应用：

```bash
docker compose up -d
```

不要将包含密码或密钥的 `.env` 文件上传到公开代码仓库。

## 5. 配置大模型

Dify 是大模型应用开发平台，本身不附带大模型。部署完成后，需要配置模型服务。

登录 Dify 后进入：

```text
设置 → 模型供应商
```

可以根据需要配置 OpenAI、DeepSeek、通义千问或其他兼容服务，并填写对应的 API Key。

### 使用本机 Ollama

如果 Ollama 运行在 Windows 或 macOS 宿主机上，Dify 容器不能通过 `localhost` 访问它。模型服务地址通常应填写：

```text
http://host.docker.internal:11434
```

原因是容器里的 `localhost` 指向容器自身，而不是宿主机。

Linux 环境需要确保容器能够访问 Ollama 所监听的宿主机地址，并确认 Ollama 没有只监听 `127.0.0.1`。

## 6. 常用维护命令

以下命令需要在 `dify/docker` 目录执行。

启动服务：

```bash
docker compose up -d
```

停止并移除容器，但保留持久化数据：

```bash
docker compose down
```

删除dify容器及其持久化数据：

```bash
只移除这次 Dify：
cd C:\Users\yirui\dify\docker
docker compose down --volumes --remove-orphans

确认目录后删除 Dify 文件和数据：
$DifyPath = (Resolve-Path -LiteralPath 'C:\Users\yirui\dify').Path

if ($DifyPath -ne 'C:\Users\yirui\dify') {
    throw "目录不匹配，停止删除"
}

cd C:\Users\yirui
Remove-Item -LiteralPath $DifyPath -Recurse -Force

可选：仅删除本次 Dify 专用镜像：

docker image rm `
  langgenius/dify-web:1.16.0 `
  langgenius/dify-api:1.16.0 `
  langgenius/dify-agent-backend:1.16.0 `
  langgenius/dify-agent-local-sandbox:1.16.0 `
  langgenius/dify-plugin-daemon:0.6.3-local `
  langgenius/dify-sandbox:0.2.15
  
  
这不会卸载 Docker Desktop，也不会主动删除其他项目的 PostgreSQL、Redis、Nginx 或 Weaviate 镜像。
```

重启服务：

```bash
docker compose restart
```

查看容器状态：

```bash
docker compose ps
```

查看所有服务日志：

```bash
docker compose logs -f
```

查看特定服务日志，例如 API：

```bash
docker compose logs -f api
```

## 7. 数据与备份

默认持久化数据主要保存在：

```text
dify/docker/volumes
```

建议定期备份：

- `docker/.env`
- `docker/envs` 中自行创建或修改的配置
- `docker/volumes`

停止服务后再进行完整备份，可降低数据不一致风险：

```bash
docker compose down
```

备份完成后重新启动：

```bash
docker compose up -d
```

> 注意：不要随意删除 `volumes` 目录，也不要执行 `docker compose down -v`，否则可能删除数据库及其他持久化数据。

## 8. 更新 Dify

生产环境建议固定到官方发布版本，不要长期直接使用持续变化的 `main` 分支。

升级前应：

1. 备份 `.env`、自定义配置和 `volumes`。
2. 阅读目标版本的升级说明。
3. 获取目标版本代码。
4. 在 `dify/docker` 目录重新启动服务。

更新镜像并启动：

```bash
docker compose pull
docker compose up -d
```

升级后检查：

```bash
docker compose ps
docker compose logs --tail=200
```

## 9. 生产环境建议

如果需要通过公网或局域网长期提供服务，建议完成以下配置：

- 固定 Dify 版本
- 使用独立域名
- 启用 HTTPS
- 更换数据库、Redis等默认密码
- 只开放必要的 80/443 端口
- 不要直接暴露 PostgreSQL、Redis 和向量数据库端口
- 定期备份数据并测试恢复流程
- 为 Docker 设置合理的磁盘和内存资源
- 配置日志、磁盘空间和容器状态监控

## 10. 常见问题

### 镜像下载失败

检查 Docker 是否能够正常访问镜像仓库。国内网络环境可能需要为 Docker Desktop 或 Docker Engine 配置可靠的代理或镜像加速服务。

### 80 端口被占用

先检查是否已有 Web 服务占用 80 端口，再根据当前版本 `.env.example` 中的 Nginx 端口配置修改映射端口。

### 页面可以打开，但模型调用失败

重点检查：

- API Key 是否正确
- 模型名称是否存在
- Dify 容器是否能访问模型 API
- 自建模型服务是否只监听了 `127.0.0.1`
- Ollama 地址是否错误填写为 `localhost`

### 服务启动异常

执行：

```bash
docker compose ps
docker compose logs --tail=300
```

根据处于 `unhealthy`、`restarting` 或 `exited` 状态的服务继续排查。

## 11. 官方资料

- [Dify Docker Compose 部署文档](https://docs.dify.ai/en/self-host/deploy/quick-start/docker-compose)
- [Dify GitHub 仓库](https://github.com/langgenius/dify)
- [Docker 部署配置说明](https://github.com/langgenius/dify/blob/main/docker/README.md)
- [Dify 官方版本发布页](https://github.com/langgenius/dify/releases)

