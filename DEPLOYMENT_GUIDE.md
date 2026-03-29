# TradingAgents-CN 完整部署指南

> **项目地址**: https://github.com/hsliuping/TradingAgents-CN  
> **版本**: v1.0.0-preview  
> **更新日期**: 2026-03-28

---

## 📋 目录

- [系统要求](#系统要求)
- [部署方式选择](#部署方式选择)
- [方式一：Docker 部署（远程 Ubuntu 服务器）](#方式一docker-部署远程-ubuntu-服务器)
- [方式二：本地源码部署](#方式二本地源码部署)
- [方式三：绿色版部署（仅Windows）](#方式三绿色版部署仅windows)
- [配置说明](#配置说明)
- [数据同步](#数据同步)
- [常见问题](#常见问题)

---

## 系统要求

### 硬件要求

| 项目 | 最低配置 | 推荐配置 |
|------|----------|----------|
| CPU | 双核 2.0GHz | 四核或更高 |
| 内存 | 4GB RAM | 8GB 或更高 |
| 存储 | 5GB 可用空间 | 20GB 或更高 |
| 网络 | 稳定的互联网连接 | 高速网络 |

### 软件要求

| 软件 | 版本要求 | 说明 |
|------|----------|------|
| 操作系统 | Windows 10/11 (64位), macOS 10.15+, Linux | 推荐 Windows 10/11 |
| Python | 3.10, 3.11, 3.12 | 推荐 3.11 |
| Node.js | 18.x 或更高 | 前端构建需要 |
| Docker | 20.x 或更高 | Docker部署需要 |
| Git | 最新版本 | 代码克隆需要 |
| Pandoc | 2.14+ | PDF/Word报告导出需要 |

> **注意**：Pandoc 是可选依赖，仅在使用**报告导出功能**（下载PDF/Word格式的分析报告）时才需要安装。

---

## 部署方式选择

| 部署方式 | 适用场景 | 难度 | 推荐指数 |
|----------|----------|------|----------|
| 🐳 Docker 部署（远程 Ubuntu） | 生产环境、跨平台、性能要求高 | ⭐⭐ 中等 | ⭐⭐⭐⭐⭐ |
| 💻 本地源码部署 | 开发者、定制需求、学习研究 | ⭐⭐⭐ 较难 | ⭐⭐⭐⭐ |
| 🟢 绿色版部署 | Windows用户、快速体验 | ⭐ 简单 | ⭐⭐⭐⭐ |

---

## 方式一：Docker 部署（远程 Ubuntu 服务器）

> 适用于：Windows 本地开发维护源代码 + 远程 Ubuntu 服务器运行 Docker 服务

### 部署架构说明

```
┌─────────────────────────────┐       ┌─────────────────────────────┐
│     Windows 开发机           │       │    Ubuntu 服务器             │
│                             │       │                             │
│  - 源代码维护                │       │  - Docker Engine            │
│  - Docker CLI (远程连接)     │ ────► │  - 运行容器                  │
│  - 镜像构建                  │       │  - 数据存储                  │
│  - 配置文件编辑              │       │                             │
└─────────────────────────────┘       └─────────────────────────────┘
```

**优势**：
- ✅ 源代码始终在本地，便于开发和版本控制
- ✅ 利用远程服务器性能运行 Docker
- ✅ 本地无需安装 Docker Desktop
- ✅ 支持多台远程服务器部署

---

### 方案选择

| 方案 | 适用场景 | 复杂度 |
|------|----------|--------|
| **方案 A：远程 Docker Context** | 推荐，操作简单，体验一致 | ⭐ 简单 |
| 方案 B：镜像仓库推送 | 需要镜像仓库，适合团队协作 | ⭐⭐ 中等 |
| 方案 C：镜像文件传输 | 无需镜像仓库，单次部署 | ⭐⭐ 中等 |

---

### 方案 A：远程 Docker Context（推荐）

通过 Docker Context 将本地 Docker CLI 连接到远程 Docker daemon，实现本地构建、远程运行。

#### 1. Ubuntu 服务器配置

##### 1.1 安装 Docker

```bash
# SSH 登录到 Ubuntu 服务器
ssh username@your-server-ip

# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装 Docker
curl -fsSL https://get.docker.com | sh

# 将当前用户添加到 docker 组
sudo usermod -aG docker $USER

# 重新登录使权限生效
exit
ssh username@your-server-ip

# 验证安装
docker --version
```

##### 1.2 配置 Docker 远程访问

```bash
# 创建 Docker 配置目录
sudo mkdir -p /etc/systemd/system/docker.service.d

# 创建配置文件
sudo vim /etc/systemd/system/docker.service.d/override.conf
```

配置内容：

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375
```

```bash
# 重新加载配置
sudo systemctl daemon-reload

# 重启 Docker
sudo systemctl restart docker

# 验证监听端口
sudo netstat -tlnp | grep 2375
```

##### 1.3 配置防火墙

```bash
# 开放必要端口
sudo ufw allow 22      # SSH
sudo ufw allow 2375    # Docker Remote API（仅限内网访问，生产环境建议用 TLS）
sudo ufw allow 3000    # 前端
sudo ufw allow 8000    # 后端 API

# 启用防火墙
sudo ufw enable
```

> ⚠️ **安全警告**：`2375` 端口无加密，仅适用于内网环境。生产环境建议配置 TLS，参考 [Docker TLS 文档](https://docs.docker.com/engine/security/protect-access/)。

#### 2. Windows 开发机配置

##### 2.1 安装 Docker CLI（无需 Docker Desktop）

**方法一：使用 Scoop 安装**

```powershell
# 安装 Scoop（如果没有）
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
irm get.scoop.sh | iex

# 安装 Docker CLI
scoop install docker

# 验证安装
docker --version
```

**方法二：手动下载**

```powershell
# 下载 Docker CLI
# 访问 https://github.com/StefanScherer/docker-cli-builder/releases
# 下载 windows-amd64 版本

# 解压后将 docker.exe 放到 PATH 目录
# 例如: C:\Windows\ 或 D:\tools\docker\
```

##### 2.2 创建远程 Docker Context

```powershell
# 创建远程上下文
docker context create remote-server `
  --docker "host=tcp://your-server-ip:2375"

# 切换到远程上下文
docker context use remote-server

# 验证连接
docker info
docker ps
```

##### 2.3 安装 Docker Compose

```powershell
# 使用 Scoop 安装
scoop install docker-compose

# 或手动下载
# 访问 https://github.com/docker/compose/releases
# 下载 windows 版本，重命名为 docker-compose.exe
# 放到 PATH 目录
```

#### 3. 本地开发部署流程

##### 3.1 克隆项目到本地 Windows

```powershell
# 克隆项目
git clone https://github.com/hsliuping/TradingAgents-CN.git
cd TradingAgents-CN
```

##### 3.2 配置环境变量

```powershell
# 复制环境变量模板
copy .env.example .env

# 编辑配置文件
notepad .env
```

**重要配置修改**（Docker 部署使用容器内网络）：

```env
# ===== MongoDB 数据库配置 =====
MONGODB_HOST=mongodb        # Docker 服务名
MONGODB_PORT=27017
MONGODB_USERNAME=admin
MONGODB_PASSWORD=tradingagents123    # 修改为强密码
MONGODB_DATABASE=tradingagents
MONGODB_AUTH_SOURCE=admin

# ===== Redis 缓存配置 =====
REDIS_HOST=redis            # Docker 服务名
REDIS_PORT=6379
REDIS_PASSWORD=tradingagents123      # 修改为强密码
REDIS_DB=0

# ===== JWT/CSRF 安全配置 =====
# 生产环境请重新生成随机字符串
# 生成方式: python -c "import secrets; print(secrets.token_urlsafe(32))"
JWT_SECRET=your-super-secret-jwt-key-at-least-32-chars
CSRF_SECRET=your-csrf-secret-key-change-in-production

# ===== LLM API 密钥 =====
DEEPSEEK_API_KEY=sk-your-deepseek-api-key
DEEPSEEK_ENABLED=true

# ===== 数据源配置 =====
DEFAULT_CHINA_DATA_SOURCE=tushare
TUSHARE_TOKEN=your-tushare-token
TUSHARE_ENABLED=true
```

> 📋 **完整配置说明**：请参考 `.env` 文件中的详细注释

##### 3.3 构建并启动服务

```powershell
# 确保使用远程上下文
docker context use remote-server

# 构建镜像（在远程服务器上构建）
docker-compose build

# 启动服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f
```

##### 3.4 访问应用

在浏览器中访问：

| 服务 | 地址 |
|------|------|
| 前端界面 | http://your-server-ip:3000 |
| 后端 API | http://your-server-ip:8000 |
| API 文档 | http://your-server-ip:8000/docs |

#### 4. 日常开发流程

```powershell
# 1. 切换到远程上下文
docker context use remote-server

# 2. 修改本地代码后，重新构建
docker-compose build backend frontend

# 3. 重启服务
docker-compose up -d

# 4. 查看日志
docker-compose logs -f backend

# 5. 切回默认上下文（可选）
docker context use default
```

#### 5. 管理 Docker Context

```powershell
# 查看所有上下文
docker context ls

# 切换上下文
docker context use remote-server
docker context use default

# 删除上下文
docker context rm remote-server

# 查看当前上下文
docker context show
```

---

### 方案 B：镜像仓库推送

适用于有镜像仓库（Docker Hub、阿里云、私有仓库）的场景。

#### 1. 本地构建镜像

```powershell
# 在项目目录下构建镜像
docker build -f Dockerfile.backend -t your-registry/tradingagents-backend:v1.0 .
docker build -f Dockerfile.frontend -t your-registry/tradingagents-frontend:v1.0 .
```

#### 2. 推送到镜像仓库

```powershell
# 登录镜像仓库
docker login your-registry

# 推送镜像
docker push your-registry/tradingagents-backend:v1.0
docker push your-registry/tradingagents-frontend:v1.0
```

#### 3. 在 Ubuntu 服务器拉取运行

```bash
# 登录镜像仓库
docker login your-registry

# 拉取镜像
docker pull your-registry/tradingagents-backend:v1.0
docker pull your-registry/tradingagents-frontend:v1.0

# 修改 docker-compose.yml 使用远程镜像
# 然后启动
docker compose up -d
```

---

### 方案 C：镜像文件传输

适用于无镜像仓库、单次部署场景。

#### 1. 本地构建并导出镜像

```powershell
# 构建镜像
docker build -f Dockerfile.backend -t tradingagents-backend:v1.0 .
docker build -f Dockerfile.frontend -t tradingagents-frontend:v1.0 .

# 导出为 tar 文件
docker save -o backend.tar tradingagents-backend:v1.0
docker save -o frontend.tar tradingagents-frontend:v1.0
```

#### 2. 传输到 Ubuntu 服务器

```powershell
# 使用 SCP 传输
scp backend.tar frontend.tar username@your-server-ip:~/
```

#### 3. 在 Ubuntu 服务器加载并运行

```bash
# 加载镜像
docker load -i ~/backend.tar
docker load -i ~/frontend.tar

# 启动服务
docker compose up -d
```

---

### 安装 Pandoc（可选，用于报告导出）

如果需要使用**PDF/Word 报告导出功能**，需要在服务器上安装 Pandoc。

#### Windows 安装 Pandoc

**方法一：使用 Chocolatey（推荐）**

```powershell
# 安装 Chocolatey（如果未安装）
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# 安装 Pandoc
choco install pandoc -y

# 验证安装
pandoc --version
```

**方法二：手动安装**

1. 访问 https://github.com/jgm/pandoc/releases
2. 下载 `pandoc-x.x.x-windows-x86_64.msi`
3. 运行安装程序
4. 将安装目录（如 `C:\Program Files\Pandoc`）添加到系统 PATH
5. 验证安装：
   ```powershell
   pandoc --version
   ```

#### Ubuntu 安装 Pandoc

```bash
# 更新软件包列表
sudo apt update

# 安装 Pandoc
sudo apt install pandoc -y

# 验证安装
pandoc --version
```

#### Docker 环境安装 Pandoc

如果使用 Docker 部署，需要在 Dockerfile 中添加 Pandoc 安装：

```dockerfile
# 在 Dockerfile.backend 中添加
RUN apt-get update && apt-get install -y pandoc
```

> **提示**：Pandoc 安装后，重启后端服务即可使用 PDF/Word 导出功能。

---

### Ubuntu 服务器运维命令

```bash
# 查看运行状态
docker compose ps

# 查看日志
docker compose logs -f --tail=100

# 重启服务
docker compose restart

# 停止服务
docker compose down

# 查看资源使用
docker stats

# 进入容器调试
docker exec -it tradingagents-backend bash
```

### 配置 Nginx 反向代理（推荐生产环境）

#### 安装 Nginx

```bash
sudo apt install nginx -y
```

#### 创建配置文件

```bash
sudo vim /etc/nginx/sites-available/tradingagents
```

配置内容：

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /api {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
    }

    location /ws {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
    }
}
```

#### 启用配置

```bash
sudo ln -s /etc/nginx/sites-available/tradingagents /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

#### 配置 HTTPS（可选）

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your-domain.com
```

### 设置开机自启

```bash
sudo systemctl enable docker
```

---

## 方式二：本地源码部署

### 1. 安装 Python 环境

**Windows系统（使用 Miniconda）**：

```powershell
# 下载 Miniconda 安装包
# 访问: https://docs.conda.io/en/latest/miniconda.html

# 安装完成后，打开 Anaconda Prompt 或 PowerShell

# 创建虚拟环境
conda create -n tradingagents python=3.11

# 激活虚拟环境
conda activate tradingagents
```

**或使用 venv**：

```powershell
# 创建虚拟环境
python -m venv tradingagents

# 激活虚拟环境（Windows）
tradingagents\Scripts\activate

# 激活虚拟环境（Linux/Mac）
source tradingagents/bin/activate
```

### 2. 安装 Node.js 环境

```powershell
# 下载 Node.js 安装包
# 访问: https://nodejs.org/

# 验证安装
node --version
npm --version
```

### 3. 安装 MongoDB

**Windows系统**：

```powershell
# 方法1：使用 Chocolatey
choco install mongodb

# 方法2：手动安装
# 1. 访问 https://www.mongodb.com/try/download/community
# 2. 下载 Windows 版本
# 3. 运行安装程序

# 启动 MongoDB 服务
net start MongoDB
```

**或使用 Docker 运行 MongoDB**：

```powershell
docker run -d `
  --name tradingagents-mongodb `
  -p 27017:27017 `
  -e MONGO_INITDB_ROOT_USERNAME=admin `
  -e MONGO_INITDB_ROOT_PASSWORD=tradingagents123 `
  -v tradingagents_mongodb_data:/data/db `
  mongo:4.4
```

### 4. 安装 Redis

**Windows系统**：

```powershell
# 使用 Docker 运行 Redis
docker run -d `
  --name tradingagents-redis `
  -p 6379:6379 `
  redis:7-alpine `
  redis-server --appendonly yes --requirepass tradingagents123
```

**或下载 Windows 版本 Redis**：
1. 访问 https://github.com/microsoftarchive/redis/releases
2. 下载 Redis-x64-xxx.msi
3. 安装并启动服务

### 5. 克隆项目并安装依赖

```powershell
# 克隆项目
git clone https://github.com/hsliuping/TradingAgents-CN.git
cd TradingAgents-CN

# 配置环境变量
copy .env.example .env
notepad .env  # 编辑配置文件
```

**本地部署配置要点**：

```env
# ===== MongoDB 数据库配置（本地无认证模式）=====
MONGODB_HOST=localhost
MONGODB_PORT=27017
MONGODB_USERNAME=       # 本地无认证，留空
MONGODB_PASSWORD=       # 本地无认证，留空
MONGODB_DATABASE=tradingagents
MONGODB_AUTH_SOURCE=

# ===== Redis 缓存配置（本地无认证模式）=====
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=         # 本地无认证，留空
REDIS_DB=0

# ===== LLM API 密钥（必需）=====
DEEPSEEK_API_KEY=sk-your-deepseek-api-key
DEEPSEEK_ENABLED=true

# ===== 数据源配置 =====
DEFAULT_CHINA_DATA_SOURCE=tushare
TUSHARE_TOKEN=your-tushare-token
TUSHARE_ENABLED=true
```

> 📋 **完整配置说明**：请参考 `.env` 文件中的详细注释

```powershell
# 安装后端依赖
pip install --upgrade pip
pip install -e .

# 或使用 requirements.txt
pip install -r requirements.txt
```

### 6. 安装前端依赖

```powershell
# 进入前端目录
cd frontend

# 安装依赖（使用 yarn）
npm install -g yarn
yarn install

# 或使用 npm
npm install
```

### 7. 初始化系统数据（首次部署必需）

```powershell
# 在项目根目录
# 激活虚拟环境
conda activate tradingagents  # 或 tradingagents\Scripts\activate

# 创建默认管理员用户和基础数据
python scripts/quick_login_fix.py
```

> **说明**：此脚本会创建默认管理员账号（admin / admin123）和基础系统配置。

### 8. 启动后端服务

```powershell
# 在项目根目录
# 激活虚拟环境
conda activate tradingagents  # 或 tradingagents\Scripts\activate

# 启动后端服务
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 9. 启动前端服务

**开发模式**：

```powershell
# 打开新的终端窗口
cd frontend

# 启动开发服务器
yarn dev

# 或使用 npm
npm run dev
```

**生产模式**：

```powershell
# 构建生产版本
yarn build

# 使用 nginx 或其他静态服务器托管 dist 目录
```

### 10. 访问应用

- **前端界面**: http://localhost:3000
- **后端API**: http://localhost:8000
- **API文档**: http://localhost:8000/docs

**首次登录**：
- 用户名：`admin`
- 密码：`admin123`

> ⚠️ **安全提示**：登录成功后请立即修改默认密码！

---

## 方式三：绿色版部署（仅Windows）

### 1. 下载绿色版

关注微信公众号 **"TradingAgents-CN"**，获取最新绿色版下载链接。

### 2. 解压并配置

```powershell
# 解压下载的压缩包到任意目录
# 例如: D:\TradingAgents-CN-Portable

# 进入目录
cd D:\TradingAgents-CN-Portable

# 编辑配置文件
notepad .env
```

### 3. 启动服务

```powershell
# 双击运行 start.bat
# 或在命令行中运行
.\start.bat
```

### 4. 访问应用

浏览器自动打开 http://localhost:8501

---

## 配置说明

> **注意**：所有配置项均在 `.env` 文件中设置。请参考项目根目录的 `.env` 文件，其中包含详细的配置说明和示例。

### 快速配置指南

1. **复制配置文件**：`copy .env.example .env`
2. **编辑配置文件**：根据您的环境修改 `.env` 文件中的相关配置
3. **关键配置项**：
   - MongoDB 连接（本地部署使用 localhost）
   - Redis 连接（本地部署使用 localhost）
   - DeepSeek API Key（AI分析功能必需）
   - Tushare Token（A股数据推荐）

### API 密钥获取

| 服务 | 用途 | 获取地址 |
|------|------|----------|
| **DeepSeek** | AI分析（推荐） | https://platform.deepseek.com/ |
| **Tushare** | A股专业数据 | https://tushare.pro/register?reg=tacn |
| **AkShare** | A股免费数据 | 无需密钥，直接使用 |
| **通义千问** | AI分析（备选） | https://dashscope.aliyun.com/ |

### 本地部署配置要点

- **MongoDB**：Windows 本地安装默认无认证，用户名密码留空
- **Redis**：Windows 建议使用 Docker 运行，或留空密码
- **JWT/CSRF**：已预生成随机值，生产环境建议重新生成

---

## 数据同步

### 同步股票基础数据

在使用分析功能前，需要先同步股票基础数据。

#### 通过Web界面同步

1. 登录系统
2. 进入 **数据管理** → **数据同步**
3. 选择数据源（Tushare/AkShare/BaoStock）
4. 点击 **同步股票列表**
5. 等待同步完成

#### 通过API同步

```powershell
# 同步Tushare数据
curl -X POST http://localhost:8000/api/tushare/sync/basics

# 同步AkShare数据
curl -X POST http://localhost:8000/api/akshare/sync/basics

# 同步BaoStock数据
curl -X POST http://localhost:8000/api/baostock/sync/basics
```

### 同步历史数据

```powershell
# 同步指定股票的历史数据
curl -X POST http://localhost:8000/api/stock/sync/historical `
  -H "Content-Type: application/json" `
  -d '{"symbol": "000001", "market_type": "a"}'
```

---

## 常见问题

### 1. Docker启动失败

**问题**: `docker-compose up -d` 失败

**解决方案**:
```powershell
# 检查Docker是否运行
docker info

# 检查端口占用
netstat -an | findstr :8000
netstat -an | findstr :3000
netstat -an | findstr :27017
netstat -an | findstr :6379

# 查看容器日志
docker-compose logs backend
docker-compose logs frontend
```

### 2. MongoDB连接失败

**问题**: 无法连接到MongoDB

**解决方案**:
```powershell
# 检查MongoDB服务状态
docker ps | findstr mongodb

# 重启MongoDB容器
docker restart tradingagents-mongodb

# 检查连接字符串
# 确保用户名密码正确
```

### 3. Redis连接失败

**问题**: 无法连接到Redis

**解决方案**:
```powershell
# 检查Redis服务状态
docker ps | findstr redis

# 测试Redis连接
docker exec -it tradingagents-redis redis-cli -a tradingagents123 ping
```

### 4. 前端无法访问后端API

**问题**: 前端显示网络错误

**解决方案**:
```powershell
# 检查后端服务状态
curl http://localhost:8000/api/health

# 检查CORS配置
# 确保 .env 中 CORS_ORIGINS 包含前端地址
```

### 5. API密钥无效

**问题**: 分析时报API密钥错误

**解决方案**:
```powershell
# 检查API密钥配置
# 确保 .env 文件中密钥正确

# 测试DeepSeek API
curl https://api.deepseek.com/v1/models `
  -H "Authorization: Bearer YOUR_API_KEY"

# 测试通义千问 API
curl https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation `
  -H "Authorization: Bearer YOUR_API_KEY"
```

### 6. Python依赖安装失败

**问题**: `pip install` 失败

**解决方案**:
```powershell
# 使用国内镜像源
pip install -e . -i https://pypi.tuna.tsinghua.edu.cn/simple

# 或使用阿里云镜像
pip install -e . -i https://mirrors.aliyun.com/pypi/simple/

# 单独安装问题包
pip install package_name -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 7. 前端构建失败

**问题**: `yarn build` 失败

**解决方案**:
```powershell
# 清除缓存
yarn cache clean

# 删除node_modules重新安装
Remove-Item -Recurse -Force node_modules
yarn install

# 检查Node.js版本
node --version  # 需要18.x或更高
```

### 8. Windows下ChromaDB问题

**问题**: ChromaDB相关错误

**解决方案**:
```powershell
# 安装Visual C++ Redistributable
# 下载地址: https://aka.ms/vs/17/release/vc_redist.x64.exe

# 或参考详细解决方案
# docs/troubleshooting/windows10-chromadb-fix.md
```

### 9. PDF/Word报告导出失败

**问题**: 下载PDF或Word报告时提示"需要安装 pandoc 工具"

**解决方案**:

**Windows:**
```powershell
# 使用 Chocolatey 安装
choco install pandoc -y

# 或手动安装
# 1. 下载: https://github.com/jgm/pandoc/releases
# 2. 运行安装程序
# 3. 重启后端服务
```

**Ubuntu:**
```bash
sudo apt update
sudo apt install pandoc -y
```

**Docker:**
```dockerfile
# 在 Dockerfile.backend 中添加
RUN apt-get update && apt-get install -y pandoc
```

安装完成后**重启后端服务**即可。

---

## 端口说明

| 服务 | 默认端口 | 说明 |
|------|----------|------|
| 前端（开发） | 5173 | Vite开发服务器 |
| 前端（生产） | 3000 | Nginx静态服务 |
| 后端API | 8000 | FastAPI服务 |
| MongoDB | 27017 | 数据库服务 |
| Redis | 6379 | 缓存服务 |
| Redis Commander | 8081 | Redis管理界面 |
| Mongo Express | 8082 | MongoDB管理界面 |

---

## 目录结构

```
TradingAgents-CN/
├── app/                    # FastAPI后端代码
│   ├── core/              # 核心配置
│   ├── routers/           # API路由
│   ├── services/          # 业务逻辑
│   ├── models/            # 数据模型
│   └── main.py            # 应用入口
├── frontend/              # Vue 3前端代码
│   ├── src/               # 源代码
│   ├── public/            # 静态资源
│   └── package.json       # 依赖配置
├── tradingagents/         # 核心分析模块
├── config/                # 配置文件
├── scripts/               # 脚本工具
├── docs/                  # 文档
├── docker-compose.yml     # Docker编排配置
├── Dockerfile.backend     # 后端Dockerfile
├── Dockerfile.frontend    # 前端Dockerfile
├── .env.example           # 环境变量模板
├── pyproject.toml         # Python项目配置
└── requirements.txt       # Python依赖
```

---

## 技术支持

### 官方渠道

- **GitHub仓库**: https://github.com/hsliuping/TradingAgents-CN
- **官方邮箱**: hsliup@163.com
- **微信公众号**: TradingAgents-CN

### 问题反馈

- **GitHub Issues**: https://github.com/hsliuping/TradingAgents-CN/issues
- **功能建议**: 通过GitHub Discussions提交

### 学习资源

- **使用指南**: docs/usage/
- **API文档**: http://localhost:8000/docs（启动后访问）
- **常见问题**: docs/faq/faq.md

---

## 许可证说明

本项目采用**混合许可证**模式：

- **开源部分**（Apache 2.0）：除 `app/` 和 `frontend/` 外的所有文件
- **专有部分**（需商业授权）：`app/`（FastAPI后端）和 `frontend/`（Vue前端）目录

**个人使用**：可自由使用全部功能  
**商业使用**：请联系获取专有组件授权：hsliup@163.com

---

**祝您使用愉快！** 🚀📈
