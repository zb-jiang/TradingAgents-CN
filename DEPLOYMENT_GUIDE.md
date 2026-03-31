# TradingAgents-CN 完整部署指南

> **项目地址**: https://github.com/hsliuping/TradingAgents-CN  
> **版本**: v1.0.0-preview  
> **更新日期**: 2026-03-28

---

## 📋 目录

- [系统要求](#系统要求)
- [部署方式](#部署方式)
- [安装步骤](#安装步骤)
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
| Git | 最新版本 | 代码克隆需要 |
| Pandoc | 2.14+ | PDF/Word报告导出需要（可选） |

> **注意**：Pandoc 是可选依赖，仅在使用**报告导出功能**（下载PDF/Word格式的分析报告）时才需要安装。

---

## 部署方式

本项目采用**本地源码部署**方式，适用于开发者、定制需求和学习研究场景。

**部署架构**：
- 后端：Python FastAPI 服务
- 前端：Vue 3 开发服务器
- 数据库：MongoDB
- 缓存：Redis

---

## 安装步骤

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

### 7. 安装 Pandoc（可选，用于报告导出）

如果需要使用**PDF/Word 报告导出功能**，需要安装 Pandoc。

**Windows 安装 Pandoc**

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

**Ubuntu 安装 Pandoc**

```bash
# 更新软件包列表
sudo apt update

# 安装 Pandoc
sudo apt install pandoc -y

# 验证安装
pandoc --version
```

> **提示**：Pandoc 安装后，重启后端服务即可使用 PDF/Word 导出功能。

### 8. 初始化系统数据（首次部署必需）

```powershell
# 在项目根目录
# 激活虚拟环境
conda activate tradingagents  # 或 tradingagents\Scripts\activate

# 创建默认管理员用户和基础数据
python scripts/quick_login_fix.py
```

> **说明**：此脚本会创建默认管理员账号（admin / admin123）和基础系统配置。

### 9. 启动后端服务

```powershell
# 在项目根目录
# 激活虚拟环境
conda activate tradingagents  # 或 tradingagents\Scripts\activate

# 启动后端服务
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 10. 启动前端服务

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

### 11. 访问应用

- **前端界面**: http://localhost:3000
- **后端API**: http://localhost:8000
- **API文档**: http://localhost:8000/docs

**首次登录**：
- 用户名：`admin`
- 密码：`admin123`

> ⚠️ **安全提示**：登录成功后请立即修改默认密码！

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

### 1. MongoDB连接失败

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

### 2. Redis连接失败

**问题**: 无法连接到Redis

**解决方案**:
```powershell
# 检查Redis服务状态
docker ps | findstr redis

# 测试Redis连接
docker exec -it tradingagents-redis redis-cli -a tradingagents123 ping
```

### 3. 前端无法访问后端API

**问题**: 前端显示网络错误

**解决方案**:
```powershell
# 检查后端服务状态
curl http://localhost:8000/api/health

# 检查CORS配置
# 确保 .env 中 CORS_ORIGINS 包含前端地址
```

### 4. API密钥无效

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

### 5. Python依赖安装失败

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

### 6. 前端构建失败

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

### 7. Windows下ChromaDB问题

**问题**: ChromaDB相关错误

**解决方案**:
```powershell
# 安装Visual C++ Redistributable
# 下载地址: https://aka.ms/vs/17/release/vc_redist.x64.exe

# 或参考详细解决方案
# docs/troubleshooting/windows10-chromadb-fix.md
```

### 8. PDF/Word报告导出失败

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
