# 新电脑跑前端项目（要求 Node 18+）环境检查清单

> 适用场景：新装机器，要在本机跑一个要求 **Node 18+** 的前端项目。
> 说明：下面所有 `sudo`/安装命令我都**只展示、不执行**，你看着核对后自己跑。命令基于你当前的系统探测结果（Ubuntu 24.04 LTS，x86_64）给出，其他发行版思路一致，命令略有差异。

---

## 0. 先看现在的机器状态（已探测）

| 项目 | 当前状态 | 结论 |
|------|----------|------|
| 系统 | Ubuntu 24.04.4 LTS (x86_64) | ✅ 主流发行版，教程多 |
| Node.js | **v22.22.2** | ✅ 已满足 Node 18+ |
| npm | 10.9.7 | ✅ 已随 Node 一起装好 |
| npx | 10.9.7 | ✅ 可用 |
| git | 2.43.0 | ✅ 已安装 |
| yarn | 1.22.22（经典版 v1） | ⚠️ 有但不一定需要 |
| pnpm | 未安装 | ❌ 若项目用 pnpm 则需补 |
| bun | 未安装 | ❌ 若项目用 bun 则需补 |
| curl | 8.5.0 | ✅ 已安装（装 nvm 等要用） |

**一句话结论**：这台机器最关键的 Node 已经到位（22 > 18），不是"从零开始"。真正要补的通常是**包管理器匹配（pnpm/bun）**、**原生编译工具链（build-essential）**和**项目读取确认**。下面给完整清单，你按需取用。

---

## 1. 环境检查清单（Checklist）

逐项打勾，先确认再装：

- [ ] **操作系统确认**：`cat /etc/os-release`，确认是 64 位（`uname -m` 应为 `x86_64` 或 `aarch64`）
- [ ] **Node 版本 ≥ 18**：`node -v`、 `node --version`
- [ ] **npm 可用**：`npm -v`
- [ ] **包管理器与项目匹配**：项目用哪个？（看 `package.json` 的 `packageManager` 字段或 lockfile 类型）
  - [ ] `package-lock.json` → npm
  - [ ] `yarn.lock` → yarn
  - [ ] `pnpm-lock.yaml` → pnpm
  - [ ] `bun.lockb` → bun
- [ ] **git 可用**：`git --version`
- [ ] **网络 / 镜像**：能访问 npm 源（国内建议配淘宝镜像）
- [ ] **原生编译工具链**：编译某些依赖（sass、esbuild、cypress 等）需要 `build-essential` + `python3`
- [ ] **浏览器**（如需 e2e 测试 / 预览）：Chrome/Chromium
- [ ] **编辑器 + 插件**：VS Code + ESLint/Prettier 等（可选）
- [ ] **项目引擎约束**：`cat package.json` 看 `engines.node`，确认不要求更高版本

---

## 2. 逐步操作（命令只展示，不执行）

### 步骤 A：确认 Node（你已经满足，可跳过安装）

```bash
# 看当前版本
node -v
npm -v

# 如果当前 < 18，推荐用版本管理器，而不是系统 apt 装死版本
# 方案 1：nvm（最常用）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# 重新开终端后：
nvm install 18      # 或 nvm install --lts
nvm use 18
nvm alias default 18

# 方案 2：fnm（更快，Rust 写）
curl -fsSL https://fnm.vercel.app/install | bash

# 方案 3：Ubuntu 直接用 NodeSource（不推荐新手，版本切换不灵活）
# curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
# sudo apt-get install -y nodejs
```

> 提示：用 nvm/fnm 可以避免 `sudo npm -g` 带来的权限坑，强烈推荐。

### 步骤 B：补齐项目需要的包管理器

```bash
# 若项目用 pnpm（没有就用 corepack 启用，无需 sudo）
# 新 Node(16.13+) 自带 corepack：
corepack enable
corepack prepare pnpm@latest --activate
pnpm -v

# 或者全局装：
# sudo npm i -g pnpm

# 若项目用 yarn（你已有 v1，但项目可能要 v3+ berry）：
# corepack enable && corepack prepare yarn@stable --activate
# 或： sudo npm i -g yarn

# 若项目用 bun：
# curl -fsSL https://bun.sh/install | bash
# bun -v
```

> 经验：先看项目的 lockfile 再决定装哪个，别全装。多数现代项目用 pnpm 或 npm。

### 步骤 C：原生编译工具链（装一次，省以后踩坑）

```bash
sudo apt-get update
sudo apt-get install -y build-essential python3 make g++ git
# build-essential 包含 gcc/g++/make，编译 node-gyp 类依赖必备
```

### 步骤 D：配置 npm（国内加速，可选但推荐）

```bash
# 查看当前源
npm config get registry
# 切到淘宝镜像（国内）
npm config set registry https://registry.npmmirror.com/
# 或用 nrm 管理： sudo npm i -g nrm && nrm use npmmirror
```

### 步骤 E：git 基础配置（首次使用要设）

```bash
git config --global user.name "你的名字"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

---

## 3. 拿到项目后怎么跑（通用流程）

```bash
# 1. 克隆（如果是私有仓库，先配 SSH key 或 HTTPS 凭据）
git clone <repo-url> && cd <project>

# 2. 看一眼引擎和脚本约束，别上来就 install
cat package.json

# 3. 用与项目匹配的包管理器装依赖
npm install        # 若用 npm
# pnpm install     # 若用 pnpm
# yarn             # 若用 yarn

# 4. 跑起来（具体看 package.json 的 scripts）
npm run dev        # 或 npm start / pnpm dev

# 5. 构建校验（CI 通常跑这个）
npm run build
```

---

## 4. 常见坑（提前避）

1. **Node 版本太高导致老项目报错**：某些老库（如旧版 node-sass、webpack4）不兼容 Node 22。若装依赖报错，用 `nvm use 18` 切回 18 LTS。
2. **权限错误 `EACCES`**：不要 `sudo npm -g`，改用 nvm/fnm 让全局包装在用户目录。
3. **pnpm 装了但命令找不到**：没 `corepack enable`，或装完没开新终端让 PATH 生效。
4. **`npm install` 卡住/超时**：八成是网络，先配镜像（步骤 D）。
5. **缺少原生编译工具**：报 `gyp ERR` / `make: not found` → 装 `build-essential`（步骤 C）。
6. **engines 警告变报错**：`package.json` 里 `engines.node` 若写死 `"18.x"` 而你是 22，npm 只是警告；但若有 `engine-strict=true` 就会失败，用 nvm 切版本最稳。

---

## 5. 给你这台机器的精简结论

基于探测，你的环境**几乎就绪**：

1. ✅ Node 22 / npm 10 已满足 18+ —— **不用重装 Node**。
2. ⚠️ 先看项目 `package.json` 和 lockfile，确认要不要补 **pnpm**（目前缺）。
3. 🔧 建议跑一次工具链安装，防原生依赖编译失败：
   ```bash
   sudo apt-get update && sudo apt-get install -y build-essential python3
   ```
4. 🌐 如果在国内，配一下镜像：
   ```bash
   npm config set registry https://registry.npmmirror.com/
   ```
5. 🚀 然后直接 `git clone` → `install` → `run dev` 即可。

需要的话，把项目的 `package.json`（或 lockfile 类型）发我，我能告诉你**具体该用哪个包管理器、要不要切 Node 版本**。
