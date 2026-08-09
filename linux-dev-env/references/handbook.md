# 程序员入职环境配置手册 - Linux 版

> 适用于 Ubuntu/Debian 等使用 systemd 的 Linux 系统。所有命令默认在个人开发机上以 `sudo` 执行。

---

## 新手指引

本手册按「从零到可开发」的顺序编排，建议**自上而下**逐章执行：

1. 先装 **Git**（代码托管必备）
2. 再装**开发语言**（Go / Node.js / Python，按项目需要选装）
3. 装**数据库与中间件**（MySQL / PostgreSQL / Redis）
4. 装 **Docker**（容器化开发）
5. 装 **Nginx**（反向代理 / 部署）
6. 装 **Claude Code**（AI 编程助手，可选）
7. 最后用文末「整体验证清单」逐项确认

---

## 目录

- [新手指引](#新手指引)
- [0. WSL 安装（仅 Windows 用户）](#0-wsl-安装仅-windows-用户)
- [阶段一：开发语言](#阶段一开发语言)
  - [1. Git](#1-git)
  - [2. Go](#2-go)
  - [3. Node.js & npm](#3-nodejs--npm)
  - [4. Yarn](#4-yarn)
  - [5. Python & pip](#5-python--pip)
  - [6. Miniconda](#6-miniconda)
- [阶段二：数据库与中间件](#阶段二数据库与中间件)
  - [7. MySQL](#7-mysql)
  - [8. PostgreSQL](#8-postgresql)
  - [9. Redis](#9-redis)
- [阶段三：容器](#阶段三容器)
  - [10. Docker](#10-docker)
- [阶段四：Web 服务器](#阶段四web-服务器)
  - [11. Nginx](#11-nginx)
- [阶段五：AI 编程助手](#阶段五ai-编程助手)
  - [12. Claude Code](#12-claude-code)
  - [13. Codex](#13-codex)
  - [14. CodeBuddy](#14-codebuddy)
- [附录](#附录)
  - [整体验证清单](#整体验证清单)
  - [快速检查脚本](#快速检查脚本)
  - [常见问题](#常见问题)
  - [更新日志](#更新日志)

---

## 0. WSL 安装（仅 Windows 用户）

> 如果你已经在用 Linux / 云主机，跳过本节。本节只针对在 Windows 上需要一套 Linux 开发环境的场景。

以**管理员**身份打开 PowerShell 或 Windows 终端，运行一条命令即可（自动启用功能、装好 Ubuntu 并设为默认）：

```powershell
wsl --install
```

> `wsl --install` 默认将 **WSL2** 设为默认版本，安装的 Ubuntu 也运行在 WSL2 内核上。

安装完成后**重启**，首次启动按提示设置用户名和密码即可进入 Linux 环境，之后的配置全部按本手册的 Linux 步骤进行。

如需确认版本，或老系统手动升级后仍是 WSL1，可运行：

```powershell
wsl --list --verbose          # 查看各发行版 VERSION 列
wsl --set-version Ubuntu 2    # 若为 1，手动切到 WSL2
```

需要指定发行版（如不想用默认 Ubuntu）可先 `wsl --list --online` 查看，`wsl --install -d Debian` 安装指定版本。

### WSL 连接 Windows 代理（梯子）

WSL2 是独立虚拟网络，**localhost 与 Windows 不互通**，所以 Windows 上开的代理（Clash / v2rayN 等）默认连不上。推荐用**镜像网络模式**解决：

在 Windows 用户目录（`C:\Users\<你>\`）新建/编辑 `.wslconfig`：

```ini
[wsl2]
networkingMode=mirrored
```

然后以管理员身份在 PowerShell 执行 `wsl --shutdown` 重启 WSL。开启后 localhost 直接互通，WSL 能直接用 `http://127.0.0.1:<代理端口>`，无需算主机 IP，并自动继承 Windows 网络栈。

> 要求 Windows 11 22H2+。老系统只能手动方式：让代理监听 `0.0.0.0`（允许局域网连接），再在 WSL 里 `export http_proxy="http://$(ip route | awk '/default/ {print $3}'):<端口>"`。

---

# 阶段一：开发语言

## 1. Git

### 安装

```bash
sudo apt update
sudo apt install git -y
```

### 验证安装

```bash
git --version
```

### 基础配置

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

### 提交信息规范

团队统一提交格式，便于阅读历史与自动生成 changelog。推荐 **Conventional Commits**：

```text
<type>(<scope>): <subject>

# 示例
feat(user): 新增用户登录接口
fix(order): 修复订单金额计算错误
docs(readme): 更新部署说明
refactor(auth): 重构鉴权逻辑
```

常用 `type`：

| type | 含义 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(cart): 添加购物车` |
| `fix` | 修复 bug | `fix(login): 修复 token 过期` |
| `docs` | 文档变更 | `docs: 更新 API 文档` |
| `style` | 格式调整（不影响逻辑） | `style: 统一缩进` |
| `refactor` | 重构（非新功能/非修复） | `refactor(util): 抽取公共方法` |
| `test` | 增加/修改测试 | `test(user): 补充登录单测` |
| `chore` | 构建/依赖等杂项 | `chore: 升级依赖版本` |

> `scope` 可省略（如 `fix: 修复空指针`）。提交信息用中文还是英文由团队统一。

### 生成 SSH 密钥

**方式一：推荐（Ed25519 + 注释）**

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
# -t ed25519：显式指定算法（推荐，安全性与性能优于 RSA）
# -C：添加注释标识密钥归属，方便管理多个密钥
# 一路回车使用默认路径 ~/.ssh/id_ed25519 即可

# 查看公钥
cat ~/.ssh/id_ed25519.pub
```

**方式二：RSA（兼容老系统时使用）**

```bash
ssh-keygen -t rsa -b 4096 -C "你的邮箱"

# 查看公钥
cat ~/.ssh/id_rsa.pub
```

**说明：**

- `ssh-keygen` 不加 `-t` 时**默认生成 RSA 密钥**，并非 Ed25519，建议显式指定 `-t ed25519`。
- 将公钥内容复制到 GitHub / GitLab 的 SSH Keys 设置中即可。

---

## 2. Go

> 使用 gvm 管理多版本 Go。gvm 安装器会自带引导用的 Go；你也可以先用 apt 装一个基础版作引导（仅用于编译 gvm 自身）。

### 步骤一：安装引导版 Go（可选）

```bash
sudo apt update
sudo apt install golang-go -y
```

仅用于引导 gvm，后续真正的开发版本由 gvm 管理。装完后可直接进入步骤二。

### 步骤二：安装 gvm（Go 版本管理器）

**方式一：官方脚本安装（推荐）**

```bash
# 安装依赖
sudo apt install curl git mercurial make binutils bison gcc build-essential -y

# 安装 gvm
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)

# 使配置生效
source ~/.bashrc
```

**方式二：国内镜像安装**

```bash
# 安装依赖
sudo apt install curl git mercurial make binutils bison gcc build-essential -y

# 使用镜像安装
bash < <(curl -s -S -L https://gitee.com/moovweb/gvm/raw/master/binscripts/gvm-installer)

# 使配置生效
source ~/.bashrc
```

**方式三：手动安装**

```bash
# 安装依赖
sudo apt install curl git mercurial make binutils bison gcc build-essential -y

# 克隆仓库
git clone https://github.com/moovweb/gvm.git ~/.gvm

# 添加到 shell 配置
echo '[[ -s "$HOME/.gvm/scripts/gvm" ]] && source "$HOME/.gvm/scripts/gvm"' >> ~/.bashrc

# 使配置生效
source ~/.bashrc
```

**验证 gvm 安装：**

```bash
gvm version
```

### 步骤三：使用 gvm 安装指定版本

```bash
# 查看可安装的版本
gvm listall

# 安装指定版本（如 1.21.5）
gvm install go1.21.5

# 设置为默认版本
gvm use go1.21.5 --default
```

### 验证安装

```bash
go version   # 应显示 gvm 管理的版本（如 go1.21.5），而非 apt 旧版
gvm list
```

> 若 `go version` 仍显示 apt 旧版，说明 gvm 的 PATH 未生效。检查 `~/.bashrc` 中 gvm 配置，重新打开终端或执行 `source ~/.bashrc` 后再试。

### gvm 常用命令

```bash
gvm list                      # 列出已安装版本
gvm listall                   # 列出可安装版本
gvm install go1.21.5          # 安装版本
gvm use go1.21.5              # 切换版本
gvm use go1.21.5 --default   # 设置默认版本
gvm uninstall go1.21.5        # 卸载版本
```

### Go 环境变量配置

**查看当前环境：**

```bash
go env
```

**常用环境变量：**

```bash
go env -w GO111MODULE=on                                              # 模块模式（推荐开启）
go env -w GOPROXY=https://goproxy.cn,direct                           # 国内代理
go env -w GOSUMDB=sum.golang.google.cn                               # 国内校验镜像
go env -w GOPRIVATE=github.com/your-company/*                        # 私有仓库（不走代理）
```

> ⚠️ 内网环境**不要**把 `GOPROXY` 设成 `off`，否则将**完全无法下载任何模块**。内网应指向公司私有代理，例如：
> `go env -w GOPROXY=https://goproxy.your-company.com,direct`

**可用镜像源：**

| 镜像源 | 地址 | 说明 |
|--------|------|------|
| 七牛云 | `https://goproxy.cn` | 推荐，国内最稳定 |
| 阿里云 | `https://mirrors.aliyun.com/goproxy/` | 阿里云镜像 |
| 官方 | `https://proxy.golang.org` | 官方代理，国内较慢 |
| 直连 | `direct` | 不使用代理，直连下载 |
| 多源组合 | `https://goproxy.cn,https://proxy.golang.org,direct` | 依次尝试 |

**推荐配置：**

```bash
# 国内环境推荐
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOSUMDB=sum.golang.google.cn

# 多源备份
go env -w GOPROXY=https://goproxy.cn,https://proxy.golang.org,direct
```

**手动配置环境变量（添加到 ~/.bashrc）：**

```bash
# Go 环境变量
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
```

**常用环境变量说明：**

| 变量 | 说明 | 推荐值 |
|------|------|--------|
| `GO111MODULE` | 模块模式 | `on` |
| `GOPROXY` | 模块代理 | `https://goproxy.cn,direct` |
| `GOPRIVATE` | 私有仓库 | `github.com/your-org/*` |
| `GOPATH` | 工作目录 | `~/go` |
| `GOCACHE` | 构建缓存 | `~/.cache/go-build` |
| `GOMODCACHE` | 模块缓存 | `$GOPATH/pkg/mod` |

**验证配置：**

```bash
go env GOPROXY
go env GOPATH
```

---

## 3. Node.js & npm

### 方法一：使用 apt 安装（简单但版本可能较旧）

```bash
sudo apt update
sudo apt install nodejs npm -y
```

### 方法二：使用 nvm 安装（推荐，可管理多版本）

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# 使配置生效
source ~/.bashrc

# 安装 Node.js LTS 版本
nvm install --lts

# 设为默认版本（否则新终端 node 不可用）
nvm alias default lts/*

# 切换到默认版本
nvm use default
```

### 验证安装

```bash
node -v
npm -v
```

### 配置国内镜像（可选）

```bash
npm config set registry https://registry.npmmirror.com
```

---

## 4. Yarn

### 方法一：使用 npm 安装（推荐）

```bash
npm install -g yarn
```

### 方法二：使用 apt 安装

> 注：新版 Debian/Ubuntu 已弃用 `apt-key`，此处改用 `signed-by` + keyrings 方式。

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/yarn.gpg
echo "deb [signed-by=/etc/apt/keyrings/yarn.gpg] https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update
sudo apt install yarn -y
```

### 验证安装

```bash
yarn -v
```

### 配置国内镜像（可选）

```bash
yarn config set registry https://registry.npmmirror.com
```

---

## 5. Python & pip

### 安装

```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv -y
```

### 验证安装

```bash
python3 --version
pip3 --version
```

### 虚拟环境（推荐）

```bash
# 在项目目录创建虚拟环境
python3 -m venv .venv

# 激活
source .venv/bin/activate

# 退出
deactivate
```

### 配置国内镜像（可选）

```bash
pip3 config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

---

# 阶段二：数据库与中间件

## 6. Miniconda

Miniconda 是轻量的 Python 包与环境管理器（Conda 的最小发行版），可隔离不同项目的依赖，适合作为默认 Python 环境方案。

### 安装

```bash
# 下载安装脚本
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# 执行安装（按提示操作，建议默认安装到 ~/miniconda3）
bash Miniconda3-latest-Linux-x86_64.sh

# 使配置生效
source ~/.bashrc
```

### 验证安装

```bash
conda --version
```

### 配置国内镜像（可选）

```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
conda config --set show_channel_urls yes
```

### 常用操作

```bash
conda create -n myenv python=3.11   # 创建环境
conda activate myenv                # 激活环境
conda deactivate                    # 退出环境
conda env list                      # 查看所有环境
conda install numpy                 # 安装包
conda remove -n myenv --all         # 删除环境
```

### 与 pip 的关系

> Conda 环境内也可用 `pip` 安装包，但建议优先 `conda install`；混用时先 conda 后 pip，避免依赖冲突。

---

## 7. MySQL

### 安装

```bash
sudo apt update
sudo apt install mysql-server -y
```

### 启动服务

```bash
sudo systemctl start mysql
sudo systemctl status mysql
```

### 安全配置

```bash
sudo mysql_secure_installation
```

按提示设置 root 密码和各项安全选项。

### 登录 MySQL

> Ubuntu 下 MySQL 默认 root 使用 `auth_socket` 插件认证，初始**无需密码**，直接用系统 root 身份进入：

```bash
sudo mysql
```

如需改用密码登录，进入后执行：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '你的密码';
FLUSH PRIVILEGES;
```

之后即可用 `mysql -u root -p` 登录。

### 常用命令

```bash
sudo systemctl start mysql     # 启动
sudo systemctl stop mysql      # 停止
sudo systemctl restart mysql   # 重启
sudo systemctl status mysql    # 查看状态
```

### 创建用户并授权

```sql
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost';
FLUSH PRIVILEGES;
```

---

## 8. PostgreSQL

### 安装

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

### 启动服务

```bash
sudo systemctl start postgresql
sudo systemctl status postgresql
```

### 登录

> 默认超级用户为 `postgres`，使用 peer 认证，需切换系统用户登录：

```bash
sudo -u postgres psql
```

### 修改密码 / 创建用户与库

```sql
ALTER USER postgres WITH PASSWORD 'password';

CREATE USER username WITH PASSWORD 'password';
CREATE DATABASE dbname OWNER username;
```

### 常用命令

```bash
sudo systemctl start postgresql     # 启动
sudo systemctl stop postgresql      # 停止
sudo systemctl restart postgresql   # 重启
sudo systemctl status postgresql    # 查看状态
```

### 配置远程访问（可选）

```bash
# 监听地址（将 localhost 改为 *）
sudo nano /etc/postgresql/<version>/main/postgresql.conf   # listen_addresses = '*'

# 允许远程连接（在文件末尾追加）
sudo nano /etc/postgresql/<version>/main/pg_hba.conf        # host all all 0.0.0.0/0 scram-sha-256
```

> 修改 `<version>` 为实际版本号（如 16），改完执行 `sudo systemctl restart postgresql`。

---

## 9. Redis

### 安装

```bash
sudo apt update
sudo apt install redis-server -y
```

### 启动服务

```bash
sudo systemctl start redis-server
sudo systemctl status redis-server
```

### 验证安装

```bash
redis-cli ping
# 返回 PONG 表示成功
# 若配置了 requirepass，使用：redis-cli -a 你的密码 ping
```

### 常用命令

```bash
sudo systemctl start redis-server   # 启动
sudo systemctl stop redis-server    # 停止
sudo systemctl restart redis-server # 重启
redis-cli                           # 连接 CLI
```

### 配置文件

```bash
sudo nano /etc/redis/redis.conf
```

常用配置项：

```conf
bind 127.0.0.1          # 绑定地址（如需远程访问可注释或修改）
protected-mode yes      # 保护模式
# requirepass your_password   # 密码（取消注释并设置）
appendonly yes          # 持久化
```

---

# 阶段三：容器

## 10. Docker

### 安装

**方式一：官方脚本（推荐）**

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**方式二：国内镜像**

```bash
curl -fsSL https://get.daocloud.io/docker | bash -s docker --mirror Aliyun
```

### 当前用户加入 docker 组（免 sudo）

```bash
sudo usermod -aG docker $USER
# 重新登录终端后生效
```

### 启动并设置开机自启

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### 验证

```bash
docker --version
docker run hello-world
```

### Docker Compose

Docker 24+ 已内置 `compose` 子命令；旧版单独安装插件：

```bash
sudo apt install docker-compose-plugin -y
docker compose version
```

### 配置国内镜像加速（可选）

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://<your-mirror>.mirror.aliyuncs.com"]
}
EOF
sudo systemctl restart docker
```

---

# 阶段四：Web 服务器

## 11. Nginx

### 安装

```bash
sudo apt update
sudo apt install nginx -y
```

### 启动服务

```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

### 验证安装

```bash
nginx -v          # 查看版本
sudo nginx -t     # 测试配置文件语法
```

### 常用命令

```bash
sudo systemctl start nginx     # 启动
sudo systemctl stop nginx      # 停止
sudo systemctl restart nginx   # 重启
sudo systemctl reload nginx    # 重新加载配置（不中断服务）
sudo systemctl status nginx    # 查看状态
```

### 配置文件

```bash
sudo nano /etc/nginx/nginx.conf                  # 主配置文件
sudo nano /etc/nginx/sites-available/default     # 默认站点
sudo nano /etc/nginx/sites-available/yoursite    # 新建站点
```

### 基础配置示例

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /var/www/html;
        index index.html index.htm;
    }

    # 反向代理示例
    location /api {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

# 阶段五：AI 编程助手

## 12. Claude Code

Claude Code 是 Anthropic 官方推出的 AI 编程助手 CLI 工具。

### 前置要求

- Node.js 18+（见第 3 节）
- npm 或 yarn

### 安装

**方式一：npm（推荐）**

```bash
npm install -g @anthropic-ai/claude-code
```

**方式二：yarn**

```bash
yarn global add @anthropic-ai/claude-code
```

### 验证安装

```bash
claude --version
```

### 登录认证

```bash
claude
```

首次运行会提示登录，按指引完成 Anthropic 账号授权。

### 常用命令

```bash
claude                                    # 启动交互模式
claude /path/to/project                  # 在指定目录启动
claude "帮我分析这个项目的结构"           # 执行单条命令
claude --model claude-sonnet-4-20250514  # 使用指定模型（示例）
claude --help                            # 查看帮助
```

### 配置文件

```bash
~/.claude/
├── config.json    # 全局配置
├── credentials    # 认证信息
└── history        # 命令历史
```

### 项目级配置

```bash
mkdir .claude                       # 在项目根目录创建
```

创建 `.claude/CLAUDE.md` 写入项目特定指令：

```markdown
# 项目说明

这是一个 Go 后端项目，使用 Gin 框架。

## 代码规范

- 遵循 Go 标准命名规范
- 使用 gofmt 格式化代码
- 错误处理要完善
```

### 常见问题

**安装后命令找不到：**

```bash
npm config get prefix                                       # 检查 npm 全局路径
echo 'export PATH=$(npm config get prefix)/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

**登录失败：**

```bash
rm -rf ~/.claude/credentials   # 清除凭证后重新登录
claude
```

---

## 13. Codex

Codex 是 OpenAI 推出的 AI 编程助手 CLI 工具（Codex CLI），可在终端中阅读代码、修改文件、执行命令。

### 前置要求

- Node.js 18+（见第 3 节）
- npm

### 安装

```bash
npm install -g @openai/codex
```

> 国内下载慢可换镜像：`npm install -g @openai/codex --registry https://registry.npmmirror.com`

### 验证安装

```bash
codex --version
```

### 登录认证

```bash
codex login
```

首次运行会打开浏览器，用 OpenAI 账号授权；也可直接设置 API Key：

```bash
export OPENAI_API_KEY=sk-xxxx   # 建议写入 ~/.bashrc
```

### 常用命令

```bash
codex                           # 启动交互模式
codex "审查这个 PR 的代码变更"   # 单条任务
codex --help                    # 查看帮助
```

### 配置文件

```bash
~/.codex/
├── config.toml    # 全局配置
└── auth.json      # 认证信息
```

---

## 14. CodeBuddy

CodeBuddy 是腾讯推出的 AI 编程助手 CLI 工具，支持自然语言驱动开发全流程（代码生成、修改、部署等）。

### 前置要求

- Node.js（见第 3 节）
- npm

### 安装

```bash
npm install -g @tencent-ai/codebuddy-code
```

### 验证安装

```bash
codebuddy --version
```

### 登录认证

```bash
codebuddy
```

首次运行按提示用微信 / 腾讯账号登录（会跳转浏览器授权）。

### 常用命令

```bash
codebuddy                      # 启动交互模式
codebuddy "帮我实现登录接口"    # 单条任务
codebuddy --help               # 查看帮助
```

### 配置文件

登录后的凭证与配置由 CodeBuddy 自动管理（默认位于用户目录下），一般无需手动修改。

---

# 附录

## 整体验证清单

- [ ] **Git**：已配置 `user.name` / `user.email`，SSH 公钥已添加到 GitHub/GitLab
- [ ] **Go**：`go version` 显示预期版本（非 apt 旧版）
- [ ] **Node**：新终端执行 `node -v` 也可用（已 `nvm alias default`）
- [ ] **npm / yarn**：registry 已切到国内镜像
- [ ] **Python**：`python3 --version` 正常，需用虚拟环境时已会 `venv`
- [ ] **Conda**：`conda --version` 正常，会用 `conda create` / `activate` 管理环境
- [ ] **MySQL / PostgreSQL / Redis**：服务已启动且可登录
- [ ] **Docker**：`docker run hello-world` 成功，当前用户在 docker 组
- [ ] **Nginx**：`sudo nginx -t` 配置无误
- [ ] **自启**：必要服务已 `systemctl enable` 开机自启

## 快速检查脚本

创建一个脚本快速检查所有服务状态：

```bash
#!/bin/bash

echo "=== 环境检查 ==="

echo -e "\n📦 Git:"
git --version

echo -e "\n🐹 Go:"
go version

echo -e "\n🐍 Python:"
python3 --version

echo -e "\n🟢 Node.js:"
node -v

echo -e "\n📦 npm:"
npm -v

echo -e "\n🧶 Yarn:"
yarn -v

echo -e "\n🗄️ MySQL:"
sudo systemctl status mysql

echo -e "\n🐘 PostgreSQL:"
sudo systemctl status postgresql

echo -e "\n🔴 Redis:"
sudo systemctl status redis-server
redis-cli ping   # 若设了 requirepass，改用：redis-cli -a 密码 ping

echo -e "\n🐳 Docker:"
docker --version
docker compose version

echo -e "\n🌐 Nginx:"
sudo systemctl status nginx
nginx -v

echo -e "\n🤖 Claude Code:"
claude --version

echo -e "\n🧠 Codex:"
codex --version

echo -e "\n🐝 CodeBuddy:"
codebuddy --version

echo -e "\n✅ 检查完成！"
```

## 常见问题

### 设置服务开机自启

Linux（systemd）下使用 `systemctl enable` 设置服务开机自启：

```bash
sudo systemctl enable mysql
sudo systemctl enable postgresql
sudo systemctl enable redis-server
sudo systemctl enable docker
sudo systemctl enable nginx
```

### 权限问题

如果遇到权限问题，使用 `sudo` 或检查用户组：

```bash
sudo usermod -aG mysql $USER
sudo usermod -aG redis $USER
sudo usermod -aG docker $USER
```

---

## 更新日志

- 2026-03-30：初始版本，包含 Git、Go、Node.js、Yarn、MySQL、Redis、Nginx
- 2026-03-30：新增 Claude Code 安装说明
- 2026-08-09：重构为「入职配置手册」分阶段结构；修正 SSH 默认算法、nvm 默认版本、Go 引导、GOPROXY=off、MySQL auth_socket、Yarn apt-key 等错误；新增 Python、PostgreSQL、Docker 章节及整体验证清单
- 2026-08-09：AI 编程助手章节补充 Codex、CodeBuddy（均以 CLI 形式安装）
