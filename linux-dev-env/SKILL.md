---
name: linux-dev-env
description: >-
  Step-by-step guide for setting up a Linux (Ubuntu/Debian, systemd) developer
  environment for frontend/backend work. Use this skill whenever the user wants
  to configure, install, or set up dev tools on Linux — including Git, Go,
  Node.js, Yarn, Python, Miniconda, MySQL, PostgreSQL, Redis, Docker, Nginx,
  or Claude Code — or mentions onboarding environment setup, "新电脑环境配置",
  "Linux 装 Go/MySQL/Redis", dev environment handbook, or verifying their setup.
  Even if they only ask about one tool, consult the bundled handbook and help
  them install/configure it step by step, running commands only when they ask.
---

# Linux 开发环境配置

本 skill 以 `references/handbook.md` 为权威参考，指导在 Linux（Ubuntu/Debian + systemd）上搭建前后端开发环境。内容工具中立，不绑定任何特定 AI 编程工具。

## 何时使用

- 用户要在 Linux 上配置 / 安装开发环境，或问某个具体工具怎么装、怎么配。
- 用户做「新电脑 / 入职环境配置」，或要检查环境是否齐全。

## 使用流程

1. **判断范围**：确认用户要装哪些工具（可多选：Git、Go、Node、Yarn、Python、Miniconda、MySQL、PostgreSQL、Redis、Docker、Nginx、Claude Code）。
2. **读取对应章节**：打开 `references/handbook.md`，定位到相关章节（目录见文件顶部）。
3. **给出命令并提示坑点**：按章节输出命令，并提醒手册中已修正的易错点：
   - SSH：用 `ssh-keygen -t ed25519`，默认并非 Ed25519。
   - nvm：装完要 `nvm alias default lts/*`，否则新终端 `node` 不可用。
   - Go：apt 版仅作 gvm 引导；`go version` 应显示 gvm 管理的版本。
   - MySQL：Ubuntu 默认 `auth_socket`，初始 `sudo mysql` 免密。
   - GOPROXY：内网**不要**设 `off`，否则无法下载任何模块。
   - Yarn（apt 安装）：用 `signed-by`，不用已弃用的 `apt-key`。
4. **先确认后执行**：任何 `sudo`、`apt install`、写系统配置的操作，**先展示命令、说明影响，等用户确认后再执行**；不自动批量安装全部工具。
5. **按需适配**：若用户不是 Ubuntu/Debian 或不是 systemd，指出差异（如 CentOS 用 `yum`/`dnf`、无 `systemctl` 时用 `service`）。
6. **验证**：装完后用手册末尾「整体验证清单」逐项核对。

## 安全与边界

- 涉及 SSH 密钥、数据库密码等敏感信息时，提示用户妥善保管，不硬编码到输出。
- 只安装用户需要的工具，不强行全装。

## 参考

- 完整安装 / 配置步骤、每条命令、验证方式、常见问题：见 `references/handbook.md`
