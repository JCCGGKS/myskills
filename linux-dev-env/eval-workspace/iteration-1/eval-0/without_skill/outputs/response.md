# Ubuntu 22.04 开发环境从零配置指南（Go 后端 + React 前端）

适用场景：你刚入职，拿到一台干净的 Ubuntu 22.04 开发机，团队技术栈是 Go 后端 + React 前端。下面按"先做哪些、后做哪些"的顺序给出每一步，并标注常见的坑。**注意：本文只给出命令，不会替你执行；涉及 `sudo` 的步骤请自己确认后再跑。**

---

## 0. 开工前的几点提醒（先读）

- **能用包管理器/官方脚本装的就别手动下载 tarball**，但 Go 和 Node 这两种，**官方二进制包比 apt 里的版本新、坑少**，强烈建议用官方方式装，不要用 `apt install golang`（版本太老，Ubuntu 22.04 自带的是 Go 1.18，团队大概率用 1.21+）。
- **不要直接 `sudo` 满天飞**。需要 root 的只有：系统更新、装 Docker、改 `/usr/local`、加用户组。
- **先问团队要三样东西再动手**：① 内部 `.gitconfig` / 邮箱规范（提交邮箱要用公司邮箱，别用私人邮箱）；② 是否走公司私有 registry（Go 的 `GOPROXY`、npm 的 `.npmrc`）；③ 项目 README / 内部 wiki 里的环境要求（Go 版本、Node 版本、数据库版本往往有硬性要求）。
- **把终端窗口分组**：比如一个跑后端、一个跑前端、一个看日志、一个跑数据库。后面会讲 `tmux`。
- **所有命令默认你在自己的家目录或非系统目录操作**，除非注明需要 `sudo`。

---

## 1. 系统更新与基础工具

```bash
# 1. 更新系统（第一次可能要等一会儿）
sudo apt update && sudo apt upgrade -y

# 2. 装一套最基础但几乎必用的工具
sudo apt install -y \
  build-essential \     # gcc/g++/make，编译一些 Go/Node 原生依赖要用
  curl wget git \       # 下载、版本控制
  ca-certificates \     # 避免 HTTPS 证书问题
  unzip zip \           # 解压各种包
  software-properties-common \  # 后面加 PPA 要用
  gnupg lsb-release \   # 加 Docker 源要用
  vim nano \            # 总得有个编辑器兜底
  htop tree jq \        # 看进程、看目录树、处理 JSON，调试神器
  net-tools             # netstat 等，查端口占用时用
```

**坑：**
- 公司机器可能走代理。`apt` 不走你的终端 `http_proxy` 环境变量，需要在 `/etc/apt/apt.conf.d/` 下加代理配置，或直接问 IT。先试 `curl -I https://go.dev`，不通就是网络环境问题，先解决这个再往下。
- 如果是**云桌面 / 无 GUI 服务器**，跳过所有 GUI 应用（VS Code 桌面版、Chrome）相关步骤，改用 `code-server` 或纯终端（见第 6 节）。

---

## 2. Git 配置（第一天就要做对）

```bash
# 设置身份——邮箱用公司邮箱！用私人邮箱提交会被团队 Code Review 吐槽
git config --global user.name "你的中文名/英文名"
git config --global user.email "you@company.com"

# 默认分支名改成 main（和 GitHub/GitLab 对齐，避免本地 master 推上去冲突）
git config --global init.defaultBranch main

# 解决跨平台换行符问题（团队有人用 Windows 时特别重要）
git config --global core.autocrlf input

# 让 git 记住凭据一段时间，不用每次输密码（用 HTTPS 克隆时）
git config --global credential.helper cache

# 让 git 区分大小写重命名，避免踩坑
git config --global core.ignorecase false

# 推荐：彩色输出 + 更友好的 diff
git config --global color.ui auto
```

**坑：**
- **优先用 SSH 克隆**，不要每次输密码。生成 SSH key 见第 5 节。
- 如果公司用 **GitLab 自签名证书**，`git clone` 会报证书错误。不要图省事 `GIT_SSL_NO_VERIFY=1`，正确做法是把公司 CA 证书加到系统信任（`sudo cp ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`）。
- 提交邮箱和 GitLab/GitHub 账号主邮箱不一致时，贡献图不会统计到你头上，CI 也可能拒绝。入职第一天就确认好。

---

## 3. 安装 Go（官方二进制方式）

```bash
# 去 https://go.dev/dl/ 看最新稳定版，下面是示例版本号，按实际替换
# 用官方 tarball，不要 apt
cd /tmp
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz

# 清掉旧版本（如果有）再装
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz

# 把 go 加进 PATH（写入你的 shell 配置文件）
# 如果用默认 bash：
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc

# 重新加载
source ~/.bashrc

# 验证
go version        # 应输出 go1.22.5
go env GOPATH
```

**Go 的代理与私有库配置（关键！）：**

```bash
# 国内/公司网络通常要设 GOPROXY，否则 go mod download 慢到怀疑人生
# 公共代理（按公司要求选，可能是公司自建）：
go env -w GOPROXY=https://proxy.golang.org,direct
# 如果是国内环境，常用：
# go env -w GOPROXY=https://goproxy.cn,direct

# 私有仓库不走代理（把公司 git 域名加进去，否则拉不到私有模块）
go env -w GOPRIVATE=git.company.com,github.com/company-private

# 打开 Go module 校验（默认就是 on，确认一下）
go env -w GOFLAGS=-mod=mod
```

**坑：**
- **不要混用 apt 的 go 和官方的 go**：如果之前 `apt install golang` 过，先 `sudo apt remove golang-go` 避免 `PATH` 冲突。
- `GOPATH/bin` 必须加进 `PATH`，否则你 `go install` 装的命令行工具（如 `air`、`golangci-lint`、`mockgen`）敲不出来。
- 公司私有模块一定要配 `GOPRIVATE`，否则 `go get` 会因为要校验 sumdb 而失败。
- Apple/Intel 之外的架构（如 ARM 云机）要下载 `arm64` 包，别下错。

---

## 4. 安装 Node.js + 包管理器（React 前端用）

React 生态推荐 **Node 18 或 20 LTS**（团队若指定版本务必对齐，比如 Next.js 14 要 Node 18.17+）。**强烈建议用 `nvm` 管理 Node 版本**，而不是 apt 或官方 tarball——因为不同项目可能要不同 Node 版本，nvm 切换最干净。

```bash
# 装 nvm（版本号去 https://github.com/nvm-sh/nvm 看最新）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# 重新加载 shell
source ~/.bashrc

# 装并切换到 LTS
nvm install --lts     # 最新 LTS，比如 20.x
nvm use --lts
nvm alias default 'lts/*'   # 设默认，新终端自动用

# 验证
node -v
npm -v
```

**可选但推荐：换用 pnpm 或 yarn**

```bash
# pnpm 更快、磁盘占用小，很多新项目标配
npm install -g pnpm

# 或 yarn（老项目常见）
# npm install -g yarn
```

**npm 镜像（公司/国内网络）：**

```bash
# 如果公司有自己的 npm registry，先要 .npmrc 配置（问团队）
# 公共镜像（国内）：
npm config set registry https://registry.npmmirror.com
# 公司私有 scope：
# npm config set @company:registry https://npm.company.com/
```

**坑：**
- **别用 `sudo npm install -g`**。用 nvm 装的 Node，全局包装在用户目录，不需要 sudo；用 sudo 反而会因权限错乱导致后面装包失败。
- `nvm` 只在交互式 shell 生效，CI/脚本里要显式 `source ~/.nvm/nvm.sh && nvm use`。
- 前端项目 `node_modules` 巨大，注意**家目录磁盘配额**。如果 `/home` 空间小，考虑把项目放数据盘，或定期 `pnpm store prune`。
- 遇到 `node-sass` / 原生模块编译失败，多半是缺 `build-essential`（第 1 节）或 Node 版本不匹配。

---

## 5. SSH Key（拉代码、连服务器要用）

```bash
# 生成 ed25519 密钥（比 RSA 更现代安全）
ssh-keygen -t ed25519 -C "you@company.com"
# 一路回车，默认存到 ~/.ssh/id_ed25519

# 启动 ssh-agent 并把私钥加进去（避免每次都要输密码）
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 把公钥打印出来，复制到 GitLab/GitHub 的 SSH Keys 设置页
cat ~/.ssh/id_ed25519.pub
```

**坑：**
- 如果公司有 **跳板机 / Bastion**，可能需要配置 `~/.ssh/config` 的 `ProxyJump`，问运维要模板。
- 多账号（个人 GitHub + 公司 GitLab）要在 `~/.ssh/config` 里用 `Host` 区分，否则会串。
- 私钥权限必须是 `600`，否则 ssh 拒绝使用：`chmod 600 ~/.ssh/id_ed25519`。

---

## 6. 编辑器 / IDE

**选项 A：VS Code（桌面环境推荐）**
```bash
# 用微软官方源装，能自动更新
sudo apt install -y wget gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
echo "deb [arch=amd64,arm64 signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update && sudo apt install -y code
```
推荐插件：`Go`（官方）、`ESLint`、`Prettier`、`React`、`Docker`、`GitLens`、`Error Lens`。

**选项 B：无 GUI 服务器 → code-server 或 Neovim**
```bash
# code-server：浏览器里跑 VS Code
curl -fsSL https://code-server.dev/install.sh | sh
# Neovim（轻量、可远程）：
sudo apt install -y neovim
```
如果你用 Neovim 做 Go + TS，插件配置（LSP、补全）门槛较高，建议先用 code-server 过渡。

**坑：**
- VS Code 远程连开发机时，Go 的 `gopls`、TS 的 `tsserver` 要在**远端**装，不是本地。第一次打开项目会提示"install all"。
- 别在远程机器上把 `gopls`/`eslint` 装在系统目录，让编辑器自己用 `PATH` 找到即可。

---

## 7. Docker（后端本地跑依赖服务几乎必装）

团队大概率用 Docker 跑数据库、Redis、本地服务编排。装 Docker Engine（不是 Desktop，那是 GUI）：

```bash
# 卸载旧版本
sudo apt remove -y docker docker-engine docker.io containerd runc

# 加 Docker 官方源
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 把当前用户加进 docker 组，免 sudo 跑 docker（重要！）
sudo usermod -aG docker $USER
# 退出重登录后生效；临时生效：
newgrp docker

# 验证
docker run hello-world
docker compose version
```

**坑：**
- **加完 docker 组必须重新登录**（注销或 `newgrp docker`），否则还是 `permission denied`。这是新人最常卡的一步。
- 公司内网镜像拉不动？问 IT 要**公司镜像加速器地址**，配到 `/etc/docker/daemon.json` 的 `registry-mirrors`。
- 磁盘：Docker 镜像/卷默认在 `/var/lib/docker`，如果系统盘小，要改 `data-root` 到数据盘。
- 不要图省事把 docker 跑在 `sudo` 下长期用，加用户组是正解。

---

## 8. 本地数据库与中间件（两种思路）

**思路 A（推荐新手）：用 Docker 跑，不污染宿主机**
```bash
# PostgreSQL 示例
docker run -d --name postgres -e POSTGRES_PASSWORD=devpass \
  -p 5432:5432 postgres:16

# Redis 示例
docker run -d --name redis -p 6379:6379 redis:7

# MySQL 示例（如团队用）
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=devpass \
  -p 3306:3306 mysql:8
```
连接串：`postgres://postgres:devpass@localhost:5432/postgres`。

**思路 B：宿主机直接装（需要 Systemd 管理的场景）**
```bash
sudo apt install -y postgresql postgresql-contrib redis-server
sudo systemctl enable --now postgresql redis-server
```
**坑：**
- Docker 跑的数据库**数据不持久**，容器删了数据没了。开发无妨，但记得 `-v` 挂卷（`-v $HOME/pgdata:/var/lib/postgresql/data`）。
- Ubuntu 22.04 自带 Postgres 14；团队若要求 16，用 Docker 的 `postgres:16` 最省事。
- 数据库默认只在本地监听，前端联调时若跨容器/跨机，注意 `bind` 地址和防火墙。

---

## 9. 终端体验增强（可选但强烈推荐）

```bash
# 装 zsh + oh-my-zsh，命令行体验提升巨大
sudo apt install -y zsh
chsh -s $(which zsh)   # 注销重登录后默认 zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# tmux：多窗口管理，跑前后端/日志超好用
sudo apt install -y tmux
# 复制 ~/.tmux.conf 模板（问团队要，或自行配置鼠标滚动、分屏）
```

**坑：**
- 切了 zsh 后，**前面所有 `echo 'export ...' >> ~/.bashrc` 要改成写 `~/.zshrc`**。如果已经写进 `.bashrc` 了，手动搬过去，或两处都写。
- `chsh` 要**重新登录**才生效，别以为命令跑完就切了。

---

## 10. 项目克隆与首次运行（收尾验证）

```bash
# 建个统一的工作目录
mkdir -p ~/workspace && cd ~/workspace

# 用 SSH 克隆（别用 HTTPS，省得老输密码）
git clone git@git.company.com:team/backend.git
git clone git@git.company.com:team/frontend.git

# 后端
cd backend
go mod download
go build ./...          # 先确保能编译
# 看 README 跑起来（可能是 make run / air / go run cmd/server）

# 前端
cd ../frontend
pnpm install            # 或 npm install / yarn
pnpm dev                # 或 npm run dev
```

**坑：**
- 后端跑起来连不上数据库？先确认 Docker 里的 PG/Redis 起来了（`docker ps`），连接串对不对。
- `go mod download` 报私有库 403？回头检查 `GOPRIVATE` 和 SSH key。
- 前端 `pnpm install` 报网络错？检查 `.npmrc`/registry 是否配了公司源。
- 端口被占用（`EADDRINUSE`）？`sudo lsof -i :3000` 或 `netstat -tulpn | grep 3000` 查谁占了。

---

## 11. 建议的目录与习惯（长期受益）

- 家目录下建 `~/workspace`（代码）、`~/go`（Go 默认）、`~/Downloads`（安装包临时放）。
- 把常用的 `export`、alias 收进 `~/.bashrc`/`~/.zshrc`，比如：
  ```bash
  alias ll='ls -alh'
  alias k='kubectl'        # 如果后续用 K8s
  export EDITOR=vim
  ```
- **入职第一周把这篇流程里"问团队要"的部分补全**（公司 GOPROXY、npm 私有源、GitLab 邮箱、docker 镜像加速器），这些是 apt/官方脚本替代不了的，只能问人。
- 配置好之后，**给这台机器做个快照或备份**（如果是云主机，让平台打镜像），下次重装省两小时。

---

## 一句话检查清单

- [ ] 系统更新、基础工具装好
- [ ] Git 身份（公司邮箱）、默认分支 main
- [ ] Go 官方版 + GOPROXY + GOPRIVATE 配好
- [ ] nvm + LTS Node + pnpm，npm registry 配好
- [ ] SSH key 加进 GitLab/GitHub
- [ ] 编辑器（VS Code / code-server）装好 + Go/React 插件
- [ ] Docker 装好 + 用户加 docker 组 + 重登录
- [ ] 数据库/Redis（Docker 或宿主机）能起来
- [ ] zsh + tmux（可选）
- [ ] 项目克隆、前后端都能跑通
- [ ] 公司专属配置（代理/镜像/私有源）已补齐

按这个顺序走，半天到一天内能把一台干净 Ubuntu 22.04 变成可开工的 Go + React 开发机。遇到卡住的地方，90% 集中在：网络/代理、PATH、docker 组要重登录、SSH key 没配好这四类，优先往这几个方向排查。
