
# 日志排查案例速查

给 `logsearch` skill 用的短版案例库。回答时优先压成 `现象 / 证据 / 判断 / 下一步`，不要展开成长教程。

## 案例索引

| 现象 | 优先日志源 | 首批关键词 | 常见判断 |
| --- | --- | --- | --- |
| 接口返回 `405` | 网关日志、`*_access.log` | `405`、`not allowed`、`method not allowed` | 路由未挂载、方法不匹配、前缀映射错误 |
| 接口返回 `502` | nginx / ingress、`journalctl`、`fincore.out`、应用主日志 | `502`、`bad gateway`、`panic`、`connection refused` | 上游未启动、端口未监听、服务 panic |
| HTTP 成功但业务失败 | `*_http.log`、`*_access.log`、`*_error.log`、业务日志 | `request_id`、`error`、`fail`、业务码 | HTTP 成功不等于业务成功 |
| 请求超时或耗时高 | `*_access.log`、`*_sql.log`、应用主日志 | `timeout`、`deadline exceeded`、`slow sql` | SQL 慢、下游慢、重试放大耗时 |
| 任务 / 消费者异常 | 模块日志、scheduler / worker / consumer 日志 | `panic`、`retry`、`consume failed`、`job failed` | 异步链路问题，不在 HTTP 入口 |
| SQL 报错或数据异常 | `*_sql.log`、应用主日志 | `duplicate entry`、`deadlock`、`rows affected` | 唯一约束冲突、锁冲突、脏数据 |
| 服务启动后退出 | `journalctl`、`fincore.out`、守护日志 | `panic`、`fatal`、`killed`、`address already in use` | 配置错误、依赖不可用、进程崩溃 |

## 405 路由未挂载

- `现象`：新增接口返回 `405`，旧接口正常，业务日志没有落下。
- `优先看`：网关日志、`*_access.log` 的 method、url、status。
- `判断`：优先怀疑路由未注册、method 不匹配、nginx / ingress 前缀映射错误。
- `下一步`：对齐路由注册路径、请求 method、网关转发前缀。
- `关键词`：`405`、`nginx-not allowed`、`method not allowed`、`route`、`prefix`

## 502 上游服务 panic

- `现象`：接口返回 `502`，同时间段进程重启或健康检查异常。
- `优先看`：`journalctl`、`fincore.out`、应用主日志里的 `panic` / `fatal`。
- `证据样例`：`panic: interface conversion: context.Context is *context.valueCtx, not *gin.Context`
- `判断`：`502` 常是网关层症状，根因在上游未启动、端口未监听或服务 panic。
- `下一步`：先确认 upstream，再沿 panic 栈回到具体业务代码。
- `关键词`：`502`、`bad gateway`、`connection refused`、`interface conversion`

## HTTP 成功但业务失败

- `现象`：`status=200`，但响应体业务码失败，前端仍报错。
- `优先看`：`*_access.log` 拿 `request_id`，再查 `*_http.log`、`*_error.log`、业务模块日志。
- `判断`：HTTP 成功只代表请求完成响应，不代表业务动作成功。
- `下一步`：按 `request_id` 串起请求体、响应体、业务错误，分别写 HTTP 结果和业务结果。
- `关键词`：`request_id`、`biz code`、`response body`、`validation failed`

## 请求超时或耗时高

- `现象`：接口超时、耗时高，访问日志里能看到长尾请求。
- `优先看`：`*_access.log` 的耗时，`*_sql.log` 的慢 SQL，再看应用日志里的下游 timeout。
- `判断`：常见根因是慢 SQL、下游调用慢、串行重试或单请求处理数据量过大。
- `下一步`：先锁定最慢层，再决定是查 SQL、查下游，还是查代码路径。
- `关键词`：`timeout`、`context deadline exceeded`、`slow sql`、`retrying`

## 任务 / 消费者异常

- `现象`：任务没跑、回调没处理、消费者卡住，但查不到 HTTP 请求日志。
- `优先看`：scheduler / worker / consumer 模块日志，按业务主键、任务名、topic 搜时间窗。
- `判断`：问题可能根本不在 HTTP 链路，而在任务创建、投递、消费、重试中的某一段。
- `下一步`：还原“创建 -> 投递 -> 消费 -> 重试 / 成功”的时间线。
- `关键词`：`scheduler`、`consumer`、`job failed`、`consume failed`、`retry count`

## SQL 报错或数据异常

- `现象`：保存失败、重复提交、更新失败，SQL 日志里出现唯一键冲突、死锁或 `rows affected = 0`。
- `优先看`：`*_sql.log` 的错误类型、SQL 语句、表名，再回业务日志找调用上下文。
- `判断`：常见根因是重复提交、并发更新、状态条件不命中或历史脏数据。
- `下一步`：按错误类型拆开处理，不要笼统归因为“数据库挂了”。
- `关键词`：`duplicate entry`、`deadlock`、`lock wait timeout`、`affected rows`

## 服务启动后退出

- `现象`：服务刚启动就掉，反复重启，外部表现为连接拒绝、`502` 或无响应。
- `优先看`：`journalctl -u fincore`、`fincore.out`、守护日志。
- `判断`：要先区分“启动即失败”还是“运行后崩溃”；前者多是配置/端口/权限，后者多是 panic 或 OOM。
- `下一步`：部署层日志定位退出时刻，应用日志定位退出原因。
- `关键词`：`fatal`、`killed`、`address already in use`、`oom`、`restart`
