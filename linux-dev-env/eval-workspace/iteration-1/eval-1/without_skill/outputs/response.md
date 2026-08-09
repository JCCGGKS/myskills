# Ubuntu 上安装 MySQL + Redis，并解决 root 登录问题

下面是在 Ubuntu（22.04/24.04 适用）上安装 MySQL 和 Redis 的完整步骤，以及你 `sudo mysql -u root -p` 一直登不进去的原因和解决办法。

---

## 一、先更新系统

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 二、安装 MySQL

```bash
sudo apt install mysql-server -y
```

安装完成后，MySQL 会自动启动并设为开机自启。确认状态：

```bash
sudo systemctl status mysql
```

看到 `active (running)` 就说明服务起来了。

---

## 三、为什么 `sudo mysql -u root -p` 登不进去（关键）

在 Ubuntu 上，通过 apt 安装的 MySQL，**root 用户默认使用 `auth_socket` 插件认证**，而不是密码认证。

这意味着：
- 你**不需要、也不能**用密码登录 root。
- `mysql -u root -p` 会提示输入密码，但你根本没设过密码 → 自然登不进去。
- 正确的用法是 **不带 `-p`**，并且用 `sudo` 以系统 root 身份连：

```bash
sudo mysql
```

只要你能 `sudo`，这样就能直接以 root 身份进入 MySQL 命令行，不需要密码。

### 如果你确实想用密码登录 root（可选）

很多教程用 `mysql -u root -p` 是因为他们给 root 设了密码。如果你想要这种体验，可以改成密码认证：

```bash
sudo mysql
```

进入 MySQL 后执行：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的强密码';
FLUSH PRIVILEGES;
EXIT;
```

之后就可以用 `mysql -u root -p` 输入密码登录了。

### 更推荐的做法：新建一个专用用户

直接用 root 开发/日常并不安全，建议新建一个用户：

```sql
CREATE USER 'dev'@'localhost' IDENTIFIED BY '你的密码';
GRANT ALL PRIVILEGES ON *.* TO 'dev'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

然后 `mysql -u dev -p` 登录即可。

---

## 四、安装 Redis

```bash
sudo apt install redis-server -y
```

启动并确认状态：

```bash
sudo systemctl status redis-server
```

看到 `active (running)` 即可。简单测试：

```bash
redis-cli ping
# 返回 PONG 说明正常
```

### Redis 安全建议（可选但推荐）

默认 Redis 只监听本机 `127.0.0.1` 且无需密码。如果要加密码，编辑配置：

```bash
sudo nano /etc/redis/redis.conf
```

找到 `requirepass` 这一行，取消注释并设置：

```
requirepass 你的强密码
```

保存后重启：

```bash
sudo systemctl restart redis-server
```

之后连接需要：

```bash
redis-cli -a 你的强密码 ping
```

---

## 五、常用运维命令速查

```bash
# 启动 / 停止 / 重启
sudo systemctl start mysql
sudo systemctl stop mysql
sudo systemctl restart mysql

sudo systemctl start redis-server
sudo systemctl restart redis-server

# 开机自启（一般安装时已自动设置）
sudo systemctl enable mysql
sudo systemctl enable redis-server

# 卸载（谨慎）
# sudo apt remove --purge mysql-server redis-server
```

---

## 六、常见问题排查

1. **`mysql -u root -p` 报 `Access denied for user 'root'@'localhost'`**
   → 就是上面说的 `auth_socket` 问题。改用 `sudo mysql`，或按第三步改成密码认证。

2. **`connection refused` / 连不上**
   → 服务没起来，检查 `systemctl status`，看是不是端口被改或没启动。

3. **忘了 root 密码**
   → 停服务后跳过权限表重置，或直接用 `sudo mysql` 进入后再 `ALTER USER` 改密。

---

## 总结

- 装：`sudo apt install mysql-server redis-server -y`
- 进 MySQL：默认用 **`sudo mysql`**（不要带 `-p`）；想用密码就改认证方式或新建用户。
- 进 Redis：`redis-cli ping` 返回 `PONG` 即可。

你之前登不进去，99% 就是被 MySQL 的 `auth_socket` 默认认证坑了——这跟密码对不对没关系，是认证方式的问题。
