# Ubuntu 22.04 开发环境从零配置指南

> 适用场景：新入职拿到 Ubuntu 22.04 开发机，团队技术栈为 **Go 后端 + React 前端**。
> 说明：下面每一步我都只给出命令+讲解+踩坑提醒。**涉及 `sudo`、安装、写系统配置的命令我不会自动执行，需要你确认后我再帮你跑。**

按你团队的技术栈，真正需要的工具是：

| 类别 | 工具 | 用途 |
|------|------|------|
| 代码托管 | Git + SSH | 拉代码、提交 |
| 后端 | Go (gvm 管理) | 后端开发 |
| 前端 | Node.js (nvm 管理) + Yarn/npm | React 开发 |
| 数据/中间件 | MySQL 或 PostgreSQL、Redis | 后端依赖 |
| 容器 | Docker | 本地起依赖、构建镜像 |
| 反代/部署 | Nginx | 本地联调、反向代理 |
| 可选 | 任意 AI 编程助手（Claude Code / Codex / CodeBuddy） | 提效 |

Python / Miniconda 你团队是 Go + React，一般不用，除非个别脚本需要，先跳过。

下面按「从零到可开发」的顺序逐章来。

---

## 阶段一：开发语言

### 1. Git（第一步，必须先做）

**我会执行的命令（确认后运行）：**

```bash
sudo apt update
sudo apt install git -y
```

**基础配置**（把 `你的用户名` / `你的邮箱` 换成你公司 Git 账号的真实信息，这个不需要 sudo，你确认后我直接跑）：

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

> ⚠️ **坑：提交信息规范。** 团队一般要求 Conventional Commits，例如 `feat(user): 新增登录接口`、`fix(order): 修复金额计算`。入职第一天就按规范来，别等被 review 打回再改。具体 type 列表（feat/fix/docs/style/refactor/test/chore）见手册，跟团队对齐用中文还是英文。

**生成 SSH 密钥（⚠️ 重点坑）：**

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
# 一路回车用默认路径 ~/.ssh/id_ed25519 即可
cat ~/.ssh/id_ed25519.pub
```

> ⚠️ **坑（手册明确修正）：`ssh-keygen` 不加 `-t` 时默认生成的是 RSA，不是 Ed25519。** 强烈建议显式写 `-t ed25519`（更安全、性能更好）。生成后把 `cat` 出来的公钥内容粘到 GitHub/GitLab 的 SSH Keys 设置里。

---

### 2. Go 后端环境

团队是 Go 后端，用 **gvm** 管理多版本 Go。

**步骤一：装引导版 Go（可选，仅用于编译 gvm 自身）**

```bash
sudo apt update
sudo apt install golang-go -y
```

> 这只是引导用途，装完之后真正的开发版本由 gvm 管。

**步骤二：安装 gvm**

```bash
sudo apt install curl git mercurial make binutils bison gcc build-essential -y
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)
source ~/.bashrc
```

> 国内网络若 GitHub 拉不动，可换 Gitee 镜像：`bash < <(curl -s -S -L https://gitee.com/moovweb/gvm/raw/master/binscripts/gvm-installer)`。
> 装完用 `gvm version` 验证。

**步骤三：安装并指定 Go 版本**

```bash
gvm listall              # 看可装版本
gvm install go1.21.5     # 版本号以团队要求为准，先问同事用哪个
gvm use go1.21.5 --default
```

> ⚠️ **坑（手册重点）：`go version` 一定要显示 gvm 管理的版本，而不是 apt 旧版。** 如果还显示旧版，说明 gvm 的 PATH 没生效——检查 `~/.bashrc` 里 gvm 那段，重新 `source ~/.bashrc` 或重开终端。

**Go 环境变量配置（⚠️ 重点坑）**

```bash
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOSUMDB=sum.golang.google.cn
# 私有仓库（公司内网代码）走 GOPRIVATE，不走代理：
go env -w GOPRIVATE=github.com/your-company/*
```

并在 `~/.bashrc` 追加：

```bash
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
```

> ⚠️ **坑（手册重点警告）：内网环境绝对不要把 `GOPROXY` 设成 `off`！** 那样会**完全无法下载任何模块**。内网应该指向公司私有代理，例如 `go env -w GOPROXY=https://goproxy.your-company.com,direct`。如果你连的是公司内网且公司有自己的 Go 代理，用公司的那个地址。

---

### 3. Node.js & npm（React 前端）

推荐用 **nvm** 管理多版本。

**我会执行的命令（确认后运行）：**

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts
```

**⚠️ 关键坑（手册重点）：装完必须设默认版本，否则新开的终端 `node` 找不到命令！**

```bash
nvm alias default lts/*
nvm use default
```

验证（新开一个终端也要能用）：

```bash
node -v
npm -v
```

> ⚠️ **坑：很多新人装完 nvm 后，第二天开新终端 `node -v` 报 command not found，就是因为忘了 `nvm alias default`。** 一定确认这一步做了。

国内镜像（可选，但推荐）：

```bash
npm config set registry https://registry.npmmirror.com
```

---

### 4. Yarn（React 项目常用）

React 项目很多用 Yarn。装法二选一：

**方法一（推荐，最简单）：**

```bash
npm install -g yarn
```

**方法二（apt，注意坑）：**

> ⚠️ **坑（手册修正）：新版 Ubuntu 已弃用 `apt-key`，必须用 `signed-by` + keyrings 方式，否则后续 apt 会报错。**

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/yarn.gpg
echo "deb [signed-by=/etc/apt/keyrings/yarn.gpg] https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update
sudo apt install yarn -y
```

验证：`yarn -v`。镜像：`yarn config set registry https://registry.npmmirror.com`。

---

## 阶段二：数据库与中间件

> 这一层按你团队实际用的来装，**先问清楚后端用 MySQL 还是 PostgreSQL、要不要 Redis**，别全装。下面都列出来，按需挑。

### 5. MySQL（若团队用 MySQL）

```bash
sudo apt update
sudo apt install mysql-server -y
sudo systemctl start mysql
sudo systemctl status mysql
sudo mysql_secure_installation   # 按提示设 root 密码、安全选项
```

> ⚠️ **坑（手册重点）：Ubuntu 下 MySQL 默认 root 用 `auth_socket` 插件认证，初始无需密码，** 直接用系统 root 身份进：
> ```bash
> sudo mysql
> ```
> 如果你想像平时那样 `mysql -u root -p` 用密码登录，进去后执行：
> ```sql
> ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '你的密码';
> FLUSH PRIVILEGES;
> ```
> 日常开发建议另建一个普通用户并授权，别一直用 root。

**开机自启：**

```bash
sudo systemctl enable mysql
```

### 6. PostgreSQL（若团队用 PostgreSQL）

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl start postgresql
```

> ⚠️ 默认超级用户 `postgres` 用 peer 认证，需切换系统用户登录：`sudo -u postgres psql`。改密码/建库：
> ```sql
> ALTER USER postgres WITH PASSWORD 'password';
> CREATE USER username WITH PASSWORD 'password';
> CREATE DATABASE dbname OWNER username;
> ```
> 自启：`sudo systemctl enable postgresql`。

### 7. Redis（缓存/会话常用）

```bash
sudo apt update
sudo apt install redis-server -y
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

验证：`redis-cli ping` 返回 `PONG` 即成功。

> 需要密码的话改 `/etc/redis/redis.conf`，取消 `requirepass` 注释并设置，重启服务。

---

## 阶段三：容器

### 8. Docker

**我会执行的命令（确认后运行）：**

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

> 国内慢可换镜像：`curl -fsSL https://get.daocloud.io/docker | bash -s docker --mirror Aliyun`。

**当前用户加入 docker 组（免 sudo，⚠️ 注意坑）：**

```bash
sudo usermod -aG docker $USER
# 重新登录终端后生效（重要！不重登还是得加 sudo）
```

启动并自启：

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

验证：

```bash
docker --version
docker run hello-world
```

> Docker 24+ 已内置 `compose` 子命令；若没有就 `sudo apt install docker-compose-plugin -y`，用 `docker compose version` 验证。
> 国内拉镜像慢可配 `/etc/docker/daemon.json` 的 `registry-mirrors`。

---

## 阶段四：Web 服务器

### 9. Nginx（本地联调反代 / 部署）

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

验证：`nginx -v` 与 `sudo nginx -t`（测试配置语法）。

> React 前端 + Go 后端联调时，常见用法是 Nginx 把 `/api` 反代到后端 `127.0.0.1:8080`，静态资源指到前端 build 目录。配置在 `/etc/nginx/sites-available/`。

---

## 阶段五：AI 编程助手（可选）

Node 已装的前提下，任选其一（你正在用的就是其中之一，这里仅作安装说明）：

- **Claude Code**：`npm install -g @anthropic-ai/claude-code`
- **Codex**：`npm install -g @openai/codex`
- **CodeBuddy**：`npm install -g @tencent-ai/codebuddy-code`

> 安装后若命令找不到，把 npm 全局 bin 加进 PATH：`echo 'export PATH=$(npm config get prefix)/bin:$PATH' >> ~/.bashrc && source ~/.bashrc`。

---

## 整体验证清单（全部装完后逐项核对）

- [ ] **Git**：`user.name`/`user.email` 已配，SSH 公钥已加到 GitHub/GitLab
- [ ] **Go**：`go version` 显示 gvm 版本（非 apt 旧版）；`GOPROXY` 在内网指向正确（**非 `off`**）
- [ ] **Node**：**新开终端** `node -v` 也可用（已 `nvm alias default`）
- [ ] **npm / yarn**：registry 已切国内镜像
- [ ] **数据库 / Redis**：服务已启动、可登录、已 `systemctl enable` 自启
- [ ] **Docker**：`docker run hello-world` 成功，当前用户在 docker 组（重登过）
- [ ] **Nginx**：`sudo nginx -t` 无误

---

## 给新人的额外提醒

1. **先问同事两件事**：Go 用哪个具体版本（如 `go1.21.5`）、公司内网有没有自己的 Go/npm/镜像代理地址。这直接决定 GOPROXY 怎么设，是最容易踩的坑。
2. **别一次性全装**：先装 Git → Go → Node/Yarn → 你后端实际用的数据库 + Redis → Docker → Nginx。确认每步能用再下一步。
3. **`source ~/.bashrc` 或重开终端**：每次改了 shell 配置（gvm / nvm / PATH），新终端才生效，很多「命令找不到」其实只是没重开终端。
4. **敏感信息别硬编码**：SSH 私钥、数据库密码、API Key 都只放本地/环境变量，不要写进脚本或提交到仓库。

---

### 下一步

你确认后，我会按上面的顺序**逐步**帮你执行（先从 Git + Go 开始，避免一次性大批量 `sudo` 安装）。请告诉我：

1. 后端用 **MySQL 还是 PostgreSQL**？Redis 要装吗？
2. 公司内网有没有 **Go 代理 / npm 镜像地址**？（没有我就先用 `goproxy.cn` / `registry.npmmirror.com`）
3. Go 的**目标版本号**是多少？（不知道就先问同事，或用 `gvm listall` 看最新稳定版）

确认后我就开始跑第一批命令。
