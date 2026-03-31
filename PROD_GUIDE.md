# TradingAgents-CN Ubuntu 生产环境部署指南（源码部署）

> **项目地址**: https://github.com/hsliuping/TradingAgents-CN  
> **版本**: v1.0.0-preview  
> **适用环境**: Ubuntu 20.04/22.04/24.04 LTS  
> **更新日期**: 2026-03-30

---

## 📋 目录

- [环境要求](#环境要求)
- [部署架构](#部署架构)
- [详细部署步骤](#详细部署步骤)
- [配置说明](#配置说明)
- [服务管理](#服务管理)
- [数据备份与恢复](#数据备份与恢复)
- [监控与日志](#监控与日志)
- [更新与维护](#更新与维护)
- [故障排查](#故障排查)

---

## 环境要求

### 服务器配置

| 项目 | 最低配置 | 推荐配置 | 说明 |
|------|----------|----------|------|
| CPU | 2核 | 4核+ | 分析任务需要较多计算资源 |
| 内存 | 4GB | 8GB+ | MongoDB 和 Redis 需要内存 |
| 磁盘 | 50GB SSD | 100GB+ SSD | 数据存储和日志 |
| 带宽 | 5Mbps | 10Mbps+ | API 调用和数据同步 |
| 操作系统 | Ubuntu 20.04+ | Ubuntu 22.04 LTS | 64位系统 |

### 必需软件

| 软件 | 版本 | 用途 |
|------|------|------|
| Python | 3.10, 3.11, 3.12 | 后端运行环境 |
| MongoDB | 4.4+ | 数据库 |
| Redis | 6.x+ | 缓存 |
| Git | 最新版 | 代码拉取 |

---

## 部署架构

```
┌─────────────────────────────────────────────────────────────┐
│                        用户访问层                            │
│                   (浏览器/移动设备)                          │
│                   http://47.94.222.6:3000                  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                     Nginx (端口:3000)                       │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  静态文件服务: / → frontend/dist/                      │ │
│  │  API 反向代理: /api → http://localhost:8000/api      │ │
│  └───────────────────────────────────────────────────────┘ │
└──────────────────┬──────────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌──────────────┐     ┌──────────────┐
│   Frontend   │     │   Backend    │
│   (Vue 3)    │     │  (FastAPI)   │
│   静态文件    │     │   端口:8000  │
│              │     │  仅本地访问   │
└──────────────┘     └──────┬───────┘
                            │
        ┌───────────────────┴───────────────┐
        ▼                                   ▼
┌──────────────┐                   ┌──────────────┐
│   MongoDB    │                   │    Redis     │
│   端口:27017 │                   │   端口:6379  │
│  本地安装    │                   │   本地安装   │
└──────────────┘                   └──────────────┘
```

> 💡 **推荐理由**：
> - 无需修改任何前端代码
> - 后端 8000 端口不暴露到外网（仅本地访问）
> - 只开放 3000 端口，更安全
> - 解决 SPA 路由刷新 404 问题

## 详细部署步骤

### 第一步：安装系统依赖

#### 1.1 系统更新

```bash
# 更新系统软件包
sudo apt update && sudo apt upgrade -y

# 安装基础工具
sudo apt install -y curl wget git vim htop net-tools build-essential
```

#### 1.2 安装 Python 3.11

```bash
# 添加 Python PPA
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# 安装 Python 3.11
sudo apt install -y python3.11 python3.11-venv python3.11-dev python3-pip

# 设置 Python 3.11 为默认版本（可选）
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1

# 验证安装
python3 --version
pip3 --version
```

#### 1.3 安装 MongoDB

> 💡 **注意**：Ubuntu 24.04 (Noble) 需要使用 MongoDB 8.0

```bash
# 添加 MongoDB 源（跳过签名验证，适用于网络受限环境）
echo "deb [ arch=amd64,arm64 trusted=yes ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

# 安装 MongoDB
sudo apt update
sudo apt install -y mongodb-org

# 启动 MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod

# 验证安装
mongosh --eval 'db.runCommand({ connectionStatus: 1 })'
```

#### 1.4 安装 Redis

```bash
# 安装 Redis
sudo apt install -y redis-server

# 确保 Redis 只绑定本地（安全考虑）
sudo nano /etc/redis/redis.conf
```

**确认以下配置**：

```bash
# 绑定本地地址（仅本地访问，不暴露外网）
bind 127.0.0.1

# 启用持久化
appendonly yes
```

```bash
# 重启 Redis
sudo systemctl restart redis-server
sudo systemctl enable redis-server

# 验证安装
redis-cli ping
```

#### 1.5 安装 Nginx（推荐）

```bash
# 安装 Nginx
sudo apt install -y nginx

# 验证安装
nginx -v
```

---

### 第二步：部署应用代码

#### 2.1 创建项目目录

```bash
# 在用户 home 目录下创建项目目录
mkdir -p ~/tradingagents
cd ~/tradingagents
```

> 💡 **说明**：应用安装在用户目录 `/home/zbjiang/tradingagents` 下，无需 sudo 权限

#### 2.2 从开发环境复制代码

**前置步骤：在 Windows 开发机构建前端**

在复制文件到服务器之前，需要先在开发机上构建前端。

**构建前端**

```powershell
# 安装依赖（如果还没安装）
npm install

# 清理旧的构建产物（避免残留文件）
Remove-Item -Recurse -Force dist -ErrorAction SilentlyContinue

# 构建生产版本（会自动读取 .env.production 配置）
npm run build

# 构建完成后，会生成 dist/ 目录
# 确认 dist 目录存在
ls dist
```

> 💡 **说明**：
> - `npm run build` 会将前端代码打包到 `dist/` 目录
> - 只需将 `dist/` 目录复制到服务器，无需复制整个 frontend 源码

---

**方式一：使用 SCP 从开发机复制（推荐）**

在开发机（Windows）上执行，**只复制必要文件**（避免复制无用文件如 .git/, node_modules/, __pycache__/ 等）：

```powershell
# 1. 后端核心代码（必须）
scp -r d:\test\TradingAgents-CN\app zbjiang@your-server-ip:~/tradingagents/
scp -r d:\test\TradingAgents-CN\tradingagents zbjiang@your-server-ip:~/tradingagents/
scp -r d:\test\TradingAgents-CN\config zbjiang@your-server-ip:~/tradingagents/
scp -r d:\test\TradingAgents-CN\scripts zbjiang@your-server-ip:~/tradingagents/

# 2. 配置文件（必须）
scp d:\test\TradingAgents-CN\pyproject.toml zbjiang@your-server-ip:~/tradingagents/
scp d:\test\TradingAgents-CN\requirements.txt zbjiang@your-server-ip:~/tradingagents/
scp d:\test\TradingAgents-CN\.env zbjiang@your-server-ip:~/tradingagents/

# 3. 前端构建产物（必须，只需 dist 目录）
scp -r d:\test\TradingAgents-CN\frontend\dist zbjiang@your-server-ip:~/tradingagents/frontend/
```

> 💡 **说明**：
> - 后端是 Python 运行时服务，无法像前端那样 build 成静态文件
> - 只需复制上述必要文件，无需复制 .git/, node_modules/, __pycache__/ 等

#### 2.3 创建虚拟环境并安装依赖

```bash
cd ~/tradingagents

# 创建虚拟环境
python3 -m venv pythonenv

# 激活虚拟环境
source pythonenv/bin/activate

# 升级 pip
pip install --upgrade pip

# 安装依赖
pip install -e .

# 或从 requirements.txt 安装
pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
```

---

### 第三步：配置环境变量

`.env` 文件已从开发环境复制到服务器（见 2.2 步骤）。

**生产环境关键配置检查**（无认证模式，数据库仅本地访问）：

```bash
cd ~/tradingagents
cat .env | grep -E "^(MONGODB_|REDIS_|JWT_SECRET|CSRF_SECRET|HOST|PORT|DEBUG|TRADINGAGENTS_LOG_DIR)"
```

**必须修改的配置项**：

| 配置项 | 说明 | 命令 |
|--------|------|------|
| `JWT_SECRET` | 至少32字符随机字符串 | `JWT=$(openssl rand -base64 32); sed -i "s|JWT_SECRET=.*|JWT_SECRET=$JWT|g" .env` |
| `CSRF_SECRET` | 至少32字符随机字符串 | `CSRF=$(openssl rand -base64 32); sed -i "s|CSRF_SECRET=.*|CSRF_SECRET=$CSRF|g" .env` |
| `LOG_FILE` | 日志文件路径（改为绝对路径） | `sed -i 's|LOG_FILE=logs/|LOG_FILE=/home/zbjiang/tradingagents/logs/|g' .env` |

> 💡 **说明**：
> - `ALLOWED_ORIGINS` 保留 `localhost` 即可，因为前后端在同一服务器
> - 只需执行上述三条 `sed` 命令快速修改

---

### 第四步：创建日志目录

```bash
mkdir -p ~/tradingagents/logs
```

---

### 第五步：创建 systemd 服务

#### 5.1 创建后端服务

```bash
sudo tee /etc/systemd/system/tradingagents-backend.service << 'EOF'
[Unit]
Description=TradingAgents Backend Service
After=network.target mongod.service redis-server.service
Wants=mongod.service redis-server.service

[Service]
Type=simple
User=zbjiang
Group=zbjiang
WorkingDirectory=/home/zbjiang/tradingagents
Environment=PATH=/home/zbjiang/tradingagents/pythonenv/bin
Environment=PYTHONPATH=/home/zbjiang/tradingagents
Environment=PYTHONUNBUFFERED=1
ExecStart=/home/zbjiang/tradingagents/pythonenv/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10
StandardOutput=append:/home/zbjiang/tradingagents/logs/backend.log
StandardError=append:/home/zbjiang/tradingagents/logs/backend-error.log

[Install]
WantedBy=multi-user.target
EOF
```

#### 5.2 启用并启动后端服务

```bash
# 重新加载 systemd
sudo systemctl daemon-reload

# 启用开机自启（仅后端）
sudo systemctl enable tradingagents-backend.service
```

---

### 第五步：配置 Nginx 反向代理

#### 5.3 创建 Nginx 配置

```bash
# 备份默认配置
sudo mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak

# 创建 TradingAgents 配置
sudo tee /etc/nginx/sites-available/tradingagents << 'EOF'
server {
    listen 3000;
    server_name _;

    # 前端静态文件根目录
    root /home/zbjiang/tradingagents/frontend/dist;
    index index.html;

    # Gzip 压缩（提升性能）
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json application/javascript;

    # API 反向代理 - 将 /api 请求转发到后端 8000 端口
    location /api/ {
        proxy_pass http://127.0.0.1:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket 支持（如果需要）
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # SPA 路由支持 - 所有非文件请求都返回 index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
EOF

# 启用配置
sudo ln -sf /etc/nginx/sites-available/tradingagents /etc/nginx/sites-enabled/

# 测试配置是否正确
sudo nginx -t

# 重启 Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

#### 5.4 验证 Nginx 配置

```bash
# 检查 Nginx 状态
sudo systemctl status nginx

# 查看 Nginx 错误日志
sudo tail -f /var/log/nginx/error.log

# 查看 Nginx 访问日志
sudo tail -f /var/log/nginx/access.log
```

---

### 第六步：初始化数据并启动服务

#### 6.1 初始化系统数据

```bash
cd ~/tradingagents
source pythonenv/bin/activate

# 创建管理员账号
python scripts/quick_login_fix.py
```

> 💡 **说明**：`quick_login_fix.py` 会创建默认管理员账号（admin / admin123）和基础系统配置。

#### 6.2 启动服务

```bash
# 启动后端服务
sudo systemctl start tradingagents-backend.service

# Nginx 已经在配置时启动了，无需额外操作

# 查看状态
sudo systemctl status tradingagents-backend.service
sudo systemctl status nginx
```

#### 6.3 同步股票基础数据（通过 Web UI）

服务启动后，需要同步股票基础数据才能正常使用分析功能：

1. **访问系统**
   - 打开浏览器访问 `http://your-server-ip:3000`
   - 使用默认账号登录：admin / admin123

2. **进入数据管理页面**
   - 点击左侧菜单 **"数据管理"** → **"数据同步"**

3. **同步股票列表**
   - 选择数据源（推荐 Tushare）
   - 点击 **"同步股票列表"** 按钮
   - 等待同步完成（约 1-2 分钟）

> 💡 **说明**：首次部署只需同步一次股票基础数据，之后系统会自动维护。

---

## 配置说明

### 目录结构

```
/home/zbjiang/tradingagents/         # 应用主目录
├── app/                              # FastAPI 后端代码
├── tradingagents/                    # 核心分析模块
├── config/                           # 配置文件
├── scripts/                          # 脚本工具
├── frontend/dist/                    # 前端构建产物
├── pythonenv/                        # Python 虚拟环境
├── logs/                             # 日志目录
├── .env                              # 环境变量
├── pyproject.toml                    # Python 项目配置
└── requirements.txt                  # Python 依赖

系统软件安装位置：
├── MongoDB: /var/lib/mongodb/        # 数据目录
│            /var/log/mongodb/        # 日志目录
│            /etc/mongod.conf         # 配置文件
└── Redis:   /var/lib/redis/          # 数据目录
             /var/log/redis/          # 日志目录
             /etc/redis/redis.conf    # 配置文件

systemd 服务配置：
├── /etc/systemd/system/tradingagents-backend.service   # 后端服务
└── /etc/systemd/system/tradingagents-frontend.service  # 前端服务
```

---

## 服务管理

### systemd 常用命令

```bash
# 查看状态
sudo systemctl status tradingagents-backend.service
sudo systemctl status tradingagents-frontend.service

# 查看日志
sudo journalctl -u tradingagents-backend.service -f
sudo journalctl -u tradingagents-frontend.service -f

# 重启服务
sudo systemctl restart tradingagents-backend.service
sudo systemctl restart tradingagents-frontend.service

# 停止服务
sudo systemctl stop tradingagents-backend.service
sudo systemctl stop tradingagents-frontend.service

# 启用/禁用开机自启
sudo systemctl enable tradingagents-backend.service
sudo systemctl disable tradingagents-backend.service
```

### 系统服务管理

```bash
# MongoDB
sudo systemctl start mongod
sudo systemctl stop mongod
sudo systemctl restart mongod
sudo systemctl status mongod

# Redis
sudo systemctl start redis-server
sudo systemctl stop redis-server
sudo systemctl restart redis-server
sudo systemctl status redis-server

# TradingAgents 服务
sudo systemctl start tradingagents-backend.service
sudo systemctl stop tradingagents-backend.service
sudo systemctl restart tradingagents-backend.service
sudo systemctl status tradingagents-backend.service

sudo systemctl start tradingagents-frontend.service
sudo systemctl stop tradingagents-frontend.service
sudo systemctl restart tradingagents-frontend.service
sudo systemctl status tradingagents-frontend.service
```

---

## 数据备份与恢复

### 自动备份脚本

```bash
tee ~/tradingagents/backup.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/home/zbjiang/backups/tradingagents"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

mkdir -p $BACKUP_DIR

# 备份 MongoDB（无认证模式）
echo "[$(date)] 备份 MongoDB..."
mongodump --host localhost --port 27017 \
  --db tradingagents --out $BACKUP_DIR/mongodb_$DATE
tar -czf $BACKUP_DIR/mongodb_$DATE.tar.gz -C $BACKUP_DIR mongodb_$DATE
rm -rf $BACKUP_DIR/mongodb_$DATE

# 备份 Redis（无认证模式）
echo "[$(date)] 备份 Redis..."
redis-cli BGSAVE
sleep 5
cp /var/lib/redis/dump.rdb $BACKUP_DIR/redis_$DATE.rdb

# 备份配置文件
echo "[$(date)] 备份配置文件..."
tar -czf $BACKUP_DIR/config_$DATE.tar.gz -C /home/zbjiang/tradingagents .env

# 清理旧备份
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.rdb" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] 备份完成"
EOF

chmod +x ~/tradingagents/backup.sh
```

### 配置定时备份

```bash
crontab -e

# 每日凌晨 2 点备份
0 2 * * * /home/zbjiang/tradingagents/backup.sh >> /home/zbjiang/tradingagents/logs/backup.log 2>&1
```

---

## 监控与日志

### 查看日志

```bash
# 应用日志（systemd）
sudo journalctl -u tradingagents-backend.service -f
sudo journalctl -u tradingagents-frontend.service -f

# 应用日志（文件）
tail -f ~/tradingagents/logs/backend.log
tail -f ~/tradingagents/logs/backend-error.log
tail -f ~/tradingagents/logs/frontend.log

# MongoDB 日志
sudo tail -f /var/log/mongodb/mongod.log

# Redis 日志
sudo tail -f /var/log/redis/redis-server.log
```

### 系统监控

```bash
# 查看系统资源
htop

# 查看磁盘使用
df -h

# 查看内存使用
free -h

# 查看服务状态
sudo systemctl status tradingagents-backend.service
sudo systemctl status tradingagents-frontend.service
```

---

## 更新与维护

### 更新应用

```bash
cd ~/tradingagents

# 1. 备份数据
./backup.sh

# 2. 停止服务
sudo systemctl stop tradingagents-backend.service
sudo systemctl stop tradingagents-frontend.service

# 3. 从开发环境复制新代码
# 使用 scp 或 git pull

# 4. 更新依赖
source pythonenv/bin/activate
pip install -e .

# 5. 重启服务
sudo systemctl start tradingagents-backend.service
sudo systemctl start tradingagents-frontend.service

# 6. 验证
sudo systemctl status tradingagents-backend.service
curl http://localhost:8000/api/health
```

### 更新前端

```bash
# 从开发环境复制新的构建产物
scp -r d:\test\TradingAgents-CN\frontend\dist zbjiang@your-server-ip:~/tradingagents/frontend/

# 重启前端服务
sudo systemctl restart tradingagents-frontend.service
```

---

## 故障排查

### 常见问题

#### 1. 后端服务无法启动

```bash
# 检查日志
sudo journalctl -u tradingagents-backend.service -f

# 检查端口占用
sudo netstat -tlnp | grep 8000

# 测试手动启动
cd ~/tradingagents
source pythonenv/bin/activate
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

#### 2. MongoDB 连接失败

```bash
# 检查 MongoDB 状态
sudo systemctl status mongod

# 测试连接（无认证模式）
mongosh --eval 'db.runCommand({ connectionStatus: 1 })'

# 检查配置
cat /etc/mongod.conf
```

#### 3. Redis 连接失败

```bash
# 检查 Redis 状态
sudo systemctl status redis-server

# 测试连接
redis-cli ping

# 检查配置（确认只绑定本地）
cat /etc/redis/redis.conf | grep -E "^bind"
```

#### 4. 前端无法访问

```bash
# 检查前端服务状态
sudo systemctl status tradingagents-frontend.service

# 检查前端日志
sudo journalctl -u tradingagents-frontend.service -f

# 检查前端文件是否存在
ls -la ~/tradingagents/frontend/dist/

# 测试端口
curl -I http://localhost:3000
```

#### 5. 端口冲突

```bash
# 查看端口占用
sudo lsof -i :3000
sudo lsof -i :8000
sudo lsof -i :27017
sudo lsof -i :6379
```

---

## 生产环境检查清单

- [ ] 修改了 JWT_SECRET 和 CSRF_SECRET（≥32字符随机字符串）
- [ ] 配置了有效的 AI API 密钥（DeepSeek 或 DashScope）
- [ ] 配置了 Tushare Token
- [ ] 配置了正确的 CORS 地址
- [ ] 启用了防火墙，**只开放 3000 端口**（前端）
- [ ] MongoDB 和 Redis **仅绑定 127.0.0.1**，不暴露到外网
- [ ] 配置了 systemd 服务开机自启
- [ ] 配置了自动备份
- [ ] 配置了日志轮转
- [ ] 测试了数据备份和恢复流程

> 💡 **安全说明**：本部署方案采用"数据库无认证 + 防火墙隔离"策略，通过仅绑定 localhost 和只开放 3000 端口来保障安全，适用于内网或单服务器部署场景

---

## 技术支持

- **GitHub Issues**: https://github.com/hsliuping/TradingAgents-CN/issues
- **官方邮箱**: hsliup@163.com
- **微信公众号**: TradingAgents-CN

---

**祝您部署顺利！** 🚀📈
