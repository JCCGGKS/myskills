# 在 Ubuntu 上安装 MySQL 和 Redis

> 按 `linux-dev-env` skill 的流程，我先给你完整的命令和要点说明。**下面的 `sudo` / `apt install` / 写系统配置命令我都不会自动执行**，会先展示给你确认。你回复「确认」或「执行」后，我再真正跑这些命令。

---

## 你之前登不进去的原因

你遇到的 `sudo mysql -u root -p` 一直登不进去，是 Ubuntu 上 MySQL 的**经典坑**：

- Ubuntu 下 MySQL 的 `root` 用户默认使用 **`auth_socket`** 插件认证（而非密码认证）。
- 这意味着：只能用 **系统 root 身份** 登录，且**直接忽略密码**。你加了 `-p` 让 MySQL 走密码校验，而 root 账号根本没设密码，于是校验失败、登录被拒。
- 正确做法：**不要带 `-p`**，用 `sudo mysql`（系统 root 身份）免密登进去，再在内部用 `ALTER USER` 给 root 设置密码并切换成密码认证插件，之后才能用 `mysql -u root -p`。

下面分两部分：安装 MySQL、安装 Redis。

---

## 一、安装 MySQL

### 1. 安装软件包

```bash
sudo apt update
sudo apt install mysql-server -y
```

> 影响：`apt update` 刷新软件源；`mysql-server` 会安装服务端 + 客户端，并自动创建 `mysql` 系统用户和初始数据目录。

### 2. 启动服务并设置开机自启

```bash
sudo systemctl start mysql
sudo systemctl enable mysql     # 开机自启，可选但推荐
sudo systemctl status mysql
```

> 确认 `status` 输出里 `Active:` 为 `active (running)` 再继续。

### 3. 安全初始化（可选但推荐）

```bash
sudo mysql_secure_installation
```

> 会交互式地让你设置密码强度策略、root 密码、移除匿名用户、禁止远程 root 登录、删除测试库等。注意：在 `auth_socket` 模式下它设置密码的方式有限，第 4 步的 `ALTER USER` 才是真正把 root 切成密码登录的关键。

### 4. 关键步骤：用 `auth_socket` 免密登入，再设置密码

**先这样登进去（不带 `-p`）：**

```bash
sudo mysql
```

> 用系统 root 身份执行，`auth_socket` 会直接放行，进入 `mysql>` 提示符。这就是你之前一直缺的那一步。

**在 `mysql>` 提示符里执行，把 root 改成密码登录：**

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '你的密码';
FLUSH PRIVILEGES;
exit
```

> 把 `你的密码` 换成你自己的强密码。**切勿把真实密码写进任何文件或聊天记录**——这里只是占位。
> 说明：Ubuntu 新版 MySQL 默认认证插件是 `caching_sha2_password`；如果你的客户端（如很老的驱动）不支持，可以改用 `mysql_native_password`。

### 5. 现在就能用密码登录了

```bash
mysql -u root -p
```

> 输入第 4 步设的密码即可。这就是你想要的 `mysql -u root -p` 登录方式。

### 6.（可选）创建专用开发账号，别用 root 直接连业务

```sql
CREATE USER 'dev'@'localhost' IDENTIFIED BY 'dev密码';
GRANT ALL PRIVILEGES ON *.* TO 'dev'@'localhost';
FLUSH PRIVILEGES;
```

> 日常开发建议用 `dev` 账号，避免拿 root 直接跑应用。

### 常用管理命令

```bash
sudo systemctl start mysql     # 启动
sudo systemctl stop mysql      # 停止
sudo systemctl restart mysql   # 重启
sudo systemctl status mysql    # 查看状态
```

---

## 二、安装 Redis

### 1. 安装软件包

```bash
sudo apt update
sudo apt install redis-server -y
```

> 影响：安装 Redis 服务端，`redis-cli` 客户端会随包一起装上。

### 2. 启动服务并设置开机自启

```bash
sudo systemctl start redis-server
sudo systemctl enable redis-server     # 开机自启，可选但推荐
sudo systemctl status redis-server
```

### 3. 验证安装

```bash
redis-cli ping
# 返回 PONG 表示成功
```

> 如果之后你给 Redis 设置了 `requirepass`，验证要带密码：
> `redis-cli -a 你的密码 ping`

### 4.（可选）基础安全/持久化配置

```bash
sudo nano /etc/redis/redis.conf
```

常用配置项：

```conf
bind 127.0.0.1          # 默认只监听本机；需远程访问再改
protected-mode yes      # 保护模式，建议保持开启
# requirepass your_password   # 取消注释并设置密码，增强安全性
appendonly yes          # 开启 AOF 持久化
```

> 改完配置后生效：
> ```bash
> sudo systemctl restart redis-server
> ```

### 常用管理命令

```bash
sudo systemctl start redis-server    # 启动
sudo systemctl stop redis-server     # 停止
sudo systemctl restart redis-server  # 重启
redis-cli                           # 连接 CLI
```

---

## 三、验证清单（装完后逐条核对）

- [ ] MySQL 服务 `active (running)`
- [ ] `sudo mysql` 能免密登入（auth_socket）
- [ ] 已用 `ALTER USER` 给 root 设密码并切到密码认证
- [ ] `mysql -u root -p` 用新密码能登入
- [ ] Redis 服务 `active (running)`
- [ ] `redis-cli ping` 返回 `PONG`
- [ ] 必要服务已 `systemctl enable` 开机自启

---

## 四、下一步

上面的命令我都**没有实际执行**。请确认以下几点，我就开始执行：

1. 是否就装 **MySQL + Redis** 这两项（不需要 Git / Node / Docker 等其它工具）？
2. MySQL root 密码你打算用哪个（给我一个规则即可，比如「用强密码，我稍后自己填」，我绝不会把真实密码写进文件）？
3. Redis 是否需要设 `requirepass` 密码、是否需开启远程访问？

你回复确认后，我会按「MySQL 安装 → 启动 → 设密码 → Redis 安装 → 启动 → 验证」的顺序执行，并在每步后反馈结果。
