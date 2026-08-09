---
name: logsearch
description: 排查和分析应用、服务、容器与基础设施日志，用于定位报错、还原时间线、判断影响范围，并给出下一步排查建议或修复方向。适用于 Codex 需要处理故障排查、日志检索、大文件日志分析、多来源时间关联、错误模式搜索、request_id 或 trace_id 追踪、正常与异常时段对比、以及基于本地日志文件、stdout/stderr、kubectl/docker/systemd 日志或归档日志包进行问题分析的场景。
---

# 日志排查

遵循“先缩小范围，再搜索，再关联，再下结论 / narrow first, search second, correlate third, conclude last”。不要从头通读大日志。

## 快速开始 / Quick Start

每次排查先确认：
- 现象是什么
- 时间范围
- 服务、机器、容器或环境
- 关键标识：`request_id`、`trace_id`、`user_id`、`order_id`
- 预期行为和实际行为

信息不完整时，先假设一个最小排查范围，并在结论里说明假设。

## 工作原则 / Working Rules

- 先按时间、服务、日志级别、链路 ID、错误关键词缩小范围
- 先用 `rg` / `rg --files`，再用 `sed -n`、`tail`、`head`、`awk` 看上下文
- 保留关键证据：首次时间、最后时间、频次、上下游关联
- 敏感信息只做必要摘录，并脱敏处理

## 排查流程 / Workflow

### 1. 定义边界 / Define Scope

明确问题、时间、位置、影响范围，把描述转换成可搜索条件。

### 2. 找日志源 / Find Sources

优先看离故障组件最近、覆盖目标时间窗口、包含链路标识的日志。常见来源：
- 应用日志
- 网关或 ingress 日志
- worker / consumer 日志
- 容器或系统服务日志

### 3. 先宽搜，再收窄 / Broad Then Narrow

先搜索宽泛信号：
- `error`
- `exception`
- `timeout`
- `failed`
- `panic`
- `oom`
- HTTP `5xx` / `4xx`

再用时间戳、`request_id` / `trace_id`、业务主键、pod / host / node、精确异常名继续收窄。

### 4. 还原时间线 / Reconstruct Timeline

整理出触发事件、第一个异常信号、后续传播、恢复或持续失败。

多来源日志要注意时区、时间格式、精度和时钟偏差；无法完全对齐时说明不确定性。

### 5. 输出结论 / Summarize Findings

结论压成以下结构：
- `范围`：看了哪些日志、哪个时间段
- `发现`：关键异常证据
- `判断`：最可能的原因，或按概率排序的假设
- `影响`：影响范围
- `下一步`：继续验证或修复的动作

证据不足时，必须说明假设、置信度和缺失证据。

## References

- `references/logsearch.md`：需要查项目日志地图、日志目录、logger 到文件名映射时读取
- `references/cases.md`：需要按现象快速选日志源、关键词和常见判断时读取
