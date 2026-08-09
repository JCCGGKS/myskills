# 项目日志地图

## 总览

这个项目的日志可以分成四类：

- Go 应用业务日志
- HTTP 中间件日志
- 数据库 / SQL 日志
- 部署层与系统日志

排查时先确认机器、服务状态和运行目录，再进入具体日志类型，不要一开始就全盘扫目录。

## 测试服务器入口

- `ssh jx`

## 测试环境日志目录速查


| 日志类型 | 目录 / 入口 | 说明 |
| --- | --- | --- |
| Go 应用业务日志 | `/opt/fincore/build/app/bin/log/` | 业务日志文件模式  `{RootDir}/{YYYY-MM}/{YYYY-MM-DD}_{module}.log` 包括 `app`、`http`、`access`、`error`、`sql` 等模块 |
| deployer / 守护日志 | `/opt/fincore/build/app/bin/logs/` | 常见文件：`fincore.out`、`fincore.pid` |
| Nginx 日志 | `/var/log/nginx/` | 如 `business.access.log`、`business.error.log`、`uniapp.error.log` |
| systemd 日志 | `journalctl -u fincore` | 用于启动、退出、重启、panic 相关问题 |

常用确认命令：

```bash
ls -lah /opt/fincore/build/app/bin/logs
ls -lah /opt/fincore/build/app/bin/log
ls -lah /var/log/nginx
sudo journalctl -u fincore -n 200 --no-pager
```

## 排查入口总表

| 日志类别 | 主要来源 | 查看位置 | 优先排查场景 |
| --- | --- | --- | --- |
| 普通业务 / 模块日志 | `go/src/utils/log` | `{RootDir}/{YYYY-MM}/{YYYY-MM-DD}_{module}.log` | 业务流程、模块行为、第三方调用 |
| HTTP 请求明细日志 | `RequestIDMiddleware` | `*_http.log` | 请求入参与响应明细 |
| HTTP 访问汇总日志 | `AccessLogMiddleware` | `*_access.log` | 接口访问结果、耗时、状态码 |
| 请求错误日志 | `ErrorLogMiddleware` | `*_error.log` | 请求链路错误、Gin 错误 |
| gform SQL 日志 | `go/src/utils/gform/sql_logger.go` | `*_sql.log` | SQL 执行、慢 SQL、SQL 错误 |
| GORM 日志 | `query.NewGormLogger(global.App.Log)` | 应用主日志体系 | 查询、慢查询、GORM 错误 |
| 操作日志 / 审计日志 | `OperateLogHandler` | DB 表 | 后台操作审计、行为追踪 |
| deployer / 守护日志 | `fincore-daemon.sh` / systemd | `journalctl`、`/opt/fincore/build/app/bin/logs/` | 启动失败、重启、进程退出、panic 输出 |

结论：

- 业务文件日志、HTTP 中间件日志、SQL 日志都在 Go 应用日志体系内
- 操作日志不是文件日志，而是直接落库
- deployer / 守护日志是部署层运行日志，和业务日志不是一套

## 各类日志说明

### 业务文件日志

相关实现：

- `go/src/utils/log/log.go`
- `go/src/utils/log/context.go`
- `go/src/utils/log/manager.go`

落点规则：

- 根目录来自 `global.App.Config.Log.RootDir`
- 文件格式：`{RootDir}/{YYYY-MM}/{YYYY-MM-DD}_{module}.log`

适合排查：

- 普通业务流程
- 模块级错误
- 第三方调用链
- 定时任务或自定义模块日志

常见模块：

- 默认应用日志：`app`
- 框架相关：`http`、`access`、`error`、`sql`
- 业务模块：`order`、`repayment`、`payment`、`risk`、`bankcard`、`channel_entry`
- 动态模块：如 `scheduler`、第三方支付模块名

#### `log.go` 预置日志器与文件名

`go/src/utils/log/log.go` 里的便捷 logger 本质上都是按模块名写到同一套文件规则下：

- 文件模式：`{RootDir}/{YYYY-MM}/{YYYY-MM-DD}_{module}.log`
- 示例：`{RootDir}/2026-04/2026-04-18_channel_entry.log`

可直接按下表找文件：

| 代码里的 logger | 模块名 | 对应文件名 |
| --- | --- | --- |
| `log.App()` | `app` | `{YYYY-MM-DD}_app.log` |
| `log.HTTP()` | `http` | `{YYYY-MM-DD}_http.log` |
| `log.SQL()` | `sql` | `{YYYY-MM-DD}_sql.log` |
| `log.Access()` | `access` | `{YYYY-MM-DD}_access.log` |
| `log.ErrorLog()` | `error` | `{YYYY-MM-DD}_error.log` |
| `log.Channel()` | `channel` | `{YYYY-MM-DD}_channel.log` |
| `log.Customer()` | `customer` | `{YYYY-MM-DD}_customer.log` |
| `log.Order()` | `order` | `{YYYY-MM-DD}_order.log` |
| `log.Payment()` | `payment` | `{YYYY-MM-DD}_payment.log` |
| `log.Repayment()` | `repayment` | `{YYYY-MM-DD}_repayment.log` |
| `log.Risk()` | `risk` | `{YYYY-MM-DD}_risk.log` |
| `log.BankCard()` | `bankcard` | `{YYYY-MM-DD}_bankcard.log` |
| `log.Identity()` | `identity` | `{YYYY-MM-DD}_identity.log` |
| `log.Complaint()` | `complaint` | `{YYYY-MM-DD}_complaint.log` |
| `log.System()` | `system` | `{YYYY-MM-DD}_system.log` |
| `log.ChannelEntry()` | `channel_entry` | `{YYYY-MM-DD}_channel_entry.log` |

补充：

- `log.WithModule("xxx")`、`log.RegisterModule("xxx", ...)`、`log.GetModule("xxx")` 这类动态模块同样落到 `{YYYY-MM-DD}_xxx.log`
- 默认 logger 没显式指定模块时，回落到 `app`，即 `{YYYY-MM-DD}_app.log`

### HTTP 中间件日志

#### 请求明细日志

- 中间件：`log.RequestIDMiddleware()`
- 文件：`*_http.log`
- 适合看：请求开始/结束、`X-Request-ID`、path、query、body、response body

#### 访问汇总日志

- 中间件：`log.AccessLogMiddleware()`
- 文件：`*_access.log`
- 适合看：request_id、method、url、ip、status、耗时、请求体/响应体大小

#### 请求错误日志

- 中间件：`log.ErrorLogMiddleware()`
- 文件：`*_error.log`
- 适合看：`gin.Context.Errors`、error type、error meta

### 数据库相关日志

#### gform SQL 日志

- 主实现：`go/src/utils/gform/sql_logger.go`
- 文件：`*_sql.log`
- 适合看：SQL 执行、慢 SQL、SQL 错误

- 当前主流程接入的是 `sql_logger.go`
- 旧版 `go/src/utils/gform/logger.go` 不是主流程排查入口

#### GORM 日志

- 实现：`go/src/query/log.go`
- 落点：应用主日志体系
- 适合看：查询、慢查询、GORM 错误

- SQL 问题先看 `*_sql.log`
- 如果是 query/GORM 链路，也要同时看应用主日志

### 操作日志 / 审计日志

- 来源：`middleware.OperateLogHandler()`

- 落点：不落文件，直接写数据库

- 优先查表：`admin_operate_log`、`user_operate_log`

- 适合排查：操作人、操作对象、IP、地区、操作名、成功/失败、详情、时间

### deployer / 守护日志

关键目录见上面的“测试环境日志目录速查”。

#### `/opt/fincore/build/app/bin/logs`

这是 deployer 和守护脚本明确使用的目录，常见文件：

- `fincore.pid`
- `fincore.out`
- `fincore.out.YYYYMMDD_HHMMSS`

适合排查：

- 启动失败
- 进程退出
- panic 标准输出
- 守护脚本重启问题

#### `/opt/fincore/build/app/bin/log`

当前仓库里没有脚本或 systemd 配置直接引用这个目录。线上如果存在，更可能是：

- 应用自身创建的业务日志目录
- 配置使用了相对路径 `./log`

结论：

- `logs/` 更像部署和守护层日志
- `log/` 更像应用自身文件日志

## 部署后运行日志定位

线上排查优先以实际 systemd 配置和运行目录为准，不要只看仓库里的旧模板路径。

优先查看：

- `/etc/systemd/system/fincore.service`
- `/opt/fincore/build/app/bin/logs/`
- `/var/log/nginx/`

不要优先相信旧模板里的历史路径，例如：

- `/opt/fincore/build/app/logs/...`

## 建议排查顺序

1. 先确认服务状态和 systemd 输出：

```bash
sudo systemctl status fincore
sudo journalctl -u fincore -n 200 --no-pager
```

2. 再看部署层输出和守护日志：

```bash
tail -n 200 /opt/fincore/build/app/bin/logs/fincore.out
cat /opt/fincore/build/app/bin/logs/fincore.pid
```

3. 如果问题发生在请求链路，优先看：

- `*_http.log`
- `*_access.log`
- `*_error.log`

4. 如果是业务流程或模块异常：

- 先确认 `global.App.Config.Log.RootDir`
- 再按 `{YYYY-MM-DD}_{module}.log` 查对应模块

5. 如果是 SQL 或查询问题：

- 先看 `*_sql.log`
- 再看应用主日志里的 GORM / query 输出

6. 如果是后台操作审计问题：

- 不查文件
- 直接查 `admin_operate_log`、`user_operate_log`

7. 如果怀疑还有 Nginx 或另一套应用日志，再补查：

```bash
ls -lah /opt/fincore/build/app/bin/log
ls -lah /opt/fincore/build/app/bin/logs
ls -lah /var/log/nginx
```
