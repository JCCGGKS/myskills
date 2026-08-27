---
name: llm-adapter-integration
description: 将新的 LLM/AI 模型（文生图、图生图、文生视频、图生视频、参考生视频、视频编辑等特定能力）根据接口文档接入 sucaihezi 的 task_svc 适配器层 task_svc/internal/handler/llmange 的工作流。当用户提供接口文档并要求"接入"/集成某个具备上述能力的模型、在 llmange 中新增或合并适配器，或对照厂商 API 调试已有适配器的请求/响应结构时使用。
origin: ECC
---

# LLM 适配器接入（sucaihezi / task_svc）

根据第三方 AI 模型的接口文档，将其接入 `task_svc/internal/handler/llmange` 适配器层的可复用工作流。
覆盖的能力包括：文生图、图生图、文生视频、图生视频、参考生视频、视频编辑等。

## 目录
- [何时启用](#何时启用)
- [硬性约束（编码前必读）](#硬性约束编码前必读)
- [工作流总览与三道确认闸门](#工作流总览与三道确认闸门)
- [阶段一：接口分析与文档撰写](#阶段一接口分析与文档撰写此阶段不写代码)
- [阶段二：技术文档落盘并确认](#阶段二技术文档落盘并确认写代码前必做)
- [阶段三：代码实现](#阶段三代码实现)
- [阶段四：输出结论报告](#阶段四输出结论报告)
- [产物落盘位置汇总](#产物落盘位置汇总)
- [参考示例：MiniMax image-01-live](#参考示例minimax-image-01-live)

## 何时启用

- 用户提供接口文档，要求"接入"/集成某个具备上述能力的模型（图像 / 视频 / 音频 / LLM）。
- 在 `task_svc/internal/handler/llmange/` 下新增或合并适配器。
- 对照厂商 API 调试已有适配器的请求/响应结构。

## 硬性约束（编码前必读）

- 适配器位于 `task_svc/internal/handler/llmange/`（包名 `llmange`）。
- 必须实现 `model.LLMApiPictureInterface`（定义于 `common/model/req_pic_model.go`）：
  - `BuildRequest(item *model.LLMApiPictureInterfaceReq, Req *fasthttp.Request) ([]byte, error)`
  - `ParseResponse(*fasthttp.Response) (*model.TaskResult, error)`
  - `GetTaskQueryUrl(taskId string, Req *fasthttp.Request) error`
- 注册在 `llm_api_interface.go` 的 `AllRegisterLLM()` 中，通过
  `RegisterLLMModelAdapter(modelName, NewXxx_ReqInfo(modelName), url)` 完成。
- **同一个 `modelName` 只能注册一次** —— 第二次注册时 `RegisterLLMModelAdapter` 会打印
  `"模型已经注册过了"`（util.Error）后 return，因此**先注册者生效**。若两种模式（如 t2i + i2i）
  共用一个模型 key，应合并进单个适配器，而非两个文件争抢同一 key。
  - 注意：**唯一性约束仅针对模型名注册**。`api_key`（如包级变量 `MiniMax_Key`）是厂商凭证，
    可跨同厂商多个适配器**复用**，不属于唯一性约束，不要把它当作重复注册的冲突来源。
- 真实签名（忽略代码里过时的 `IsSupportAsyc` doc comment）：
  `RegisterLLMModelAdapter(modelName string, adapter model.LLMApiPictureInterface, ReqUrl string)`。
- 边界说明：`task_svc` 适配器实现 `LLMApiPictureInterface`；`client` 侧有**独立**的 manager
  （`client/internal/handler/audio`）注册的是 `LLMApiAudioInterface` —— 不要把图片适配器注册到
  client 侧。本技能只覆盖 task_svc。
- 仓库约定：`Code == 0` 为成功，`Code == 100` 为鉴权失败（适配器内部不相关，但 handler 会遵循）。

## 工作流总览与三道确认闸门

整体分四个阶段，**必须按顺序推进，不能直接跳到写代码**：

```
阶段一 接口分析（不写码） → 阶段二 技术文档落盘 + 确认 → 阶段三 代码实现 → 阶段四 结论报告
```

写码前有两道"写码前分叉"和一道"落盘确认"，**任一处必须停下等用户确认，不得擅自二选一或臆测通过**：

| 闸门 | 触发场景 | 位置 |
|---|---|---|
| 闸门① 多模型变体 | 文档列出多个可选 `model`（如 `image-01` / `image-01-live`） | 阶段一·5 |
| 闸门② 同步/异步双模 | 文档同时支持同步与异步两种模式 | 阶段一·3 |
| 闸门③ 技术文档确认 | 阶段二文档写完后，等用户确认再写码 | 阶段二 |

## 阶段一：接口分析与文档撰写（此阶段不写代码）

目标：把接口文档吃透，产出一份结构化的接口说明，供阶段二落盘。

1. **提取请求/响应体**
   从厂商文档中完整摘录请求体（含所有字段）与响应体（成功 / 失败 / 进行中）。

2. **撰写接口 schema 文档**，对每个字段标明三类约束之一：
   - **必传（required）**：每次调用都必须提供。
   - **条件必传（conditionally required）**：满足特定条件才必传，例如"图生图时 `image_urls`
     必传，纯文生图时可省略"——必须写清触发条件。
   - **可选（optional）**：有默认值或不传不影响主流程。
   响应侧同样标注关键字段（结果数组、业务错误码 `base_resp`/`code`、任务 ID 等）。
   **以真实 API 响应为准，而非文档示例**——字段名/类型若有出入，优先信真实响应。

3. **判定同步 / 异步**
   - 同步：POST 直接返回结果 → `GetTaskQueryUrl` 返回 `nil`，`ParseResponse` 设
     `TaskStatus: "SUCCEEDED"`。
   - 异步：返回 `task_id` 但**不含结果**，需轮询 → `GetTaskQueryUrl` 构造查询 URL，
     `ParseResponse` 须区分进行中与完成。
   - **判定关键：看"响应是否内联返回了结果"，而不是"有没有 id 字段"。** 部分接口（如 MiniMax）
     的 200 响应既带 `id`（文档称"用于查询任务状态"）又**直接内联返回** `image_urls` /
     `video_url`——这种情况仍是**同步**，`id` 只是记录标识，不要被它误导成异步。只有"响应里
     只有 `task_id`、没有结果"时才算异步。
   - 验证手段：`grep -n 'func.*GetTaskQueryUrl' <同形态适配器.go>` —— `return nil` 即同步；
     `Req.SetRequestURI(queryUrl)` 即异步。同时看文档是否出现 "task query / polling /
     callback"。
   - **闸门②：若文档同时支持同步与异步两种模式，立刻停下向用户确认采用哪一种，确认后再继续。**
     不要在未确认时自行二选一。

4. **设计临时结构体（TemReq）并完成参数拼装方案**
   - 临时结构体用于接收前端传来的参数，从 `item.Req` 反序列化得到。
   - **关键**：前端入参的类型**只有** `string` / `int` / `float` / `bool` / `array` 五种，
     不存在嵌套对象。撰写文档时必须逐个字段标明其前端类型（如 `image_urls` 是 `array` 的
     string 列表，而非对象）。
   - 拼装方案：说明如何把这套"仅基础类型"的 TemReq，转换成厂商要求的请求体。典型转换包括：
     - `array`（如图片 URL 列表）→ 厂商需要的嵌套结构体数组（参考 MiniMax 的
       `buildImageSubjectRefs`：把 `[]string` 转成 `[]*SubjectRef`）。
     - `string` 枚举 / 数值 → 直接填入或映射为厂商字段。
     - 条件必传字段 → 写清按什么条件拼入、不满足时整体省略。

5. **梳理可选模型变体（多模型时停下让用户选）**
   - **闸门①**：若文档列出多个可选 `model`（如 `image-01` / `image-01-live`），**不要自行挑一个
     就写**——停下，把可选模型清单（含各自差异：支持的枚举、专属字段、价格/权限等）列给用户，
     **等用户选定后再继续**分析后续阶段。
   - 理由：不同变体字段与约束不同，选错会导致适配器字段错位。例：MiniMax i2i 中 `21:9` 仅
     `image-01` 支持、`width`/`height` 仅 `image-01` 生效、`style` 仅 `image-01-live` 生效——
     若一个变体被误当另一个接入，白名单与条件必传逻辑全错。

## 阶段二：技术文档落盘并确认（写代码前必做）

- 将阶段一成果写成一份**明确的技术文档**，输出到仓库的 `wiki/plans/` 目录，命名建议
  `<模型名>_adapter_plan.md`（如 `wiki/plans/image-01-live_adapter_plan.md`）。
- 文档至少包含：能力类型、请求/响应 schema（必传/条件必传/可选）、同步或异步结论（含如
  有则记录"已与用户确认采用 X 模式"）、TemReq 字段与类型清单、参数拼装映射、拟注册的文件与
  模型 key、参考的同类适配器。
- **闸门③：把文档路径告诉用户并等待确认，确认通过后方可进入阶段三写代码。** 用户可能在此修正
  字段约束或拼装逻辑，应先改文档再改码。
- **无人确认时的行为**（自动化评测 / 用户暂离）：若无法获得真人确认，应把 plan 文档写到
  `wiki/plans/` 后明确告知"等待确认，已暂停写码"并停止，**不得臆测确认通过**而擅自进入阶段三。

## 阶段三：代码实现

1. **结构体骨架**：从最小骨架起手，再按文档补全字段。

   ```go
   type Xxx_Req struct {
       Prompt        string                   `json:"prompt"`
       Style         *Xxx_Style               `json:"style,omitempty"`   // 指针 ⇒ 为 nil 时省略
       EnableThing   *bool                    `json:"enable_thing,omitempty"`
       ResponseFormat string                  `json:"response_format"`   // 常硬编码为 "url"
   }
   type Xxx_Style struct { Type string `json:"type"` }
   type Xxx_TemReq struct {
       Prompt string `json:"prompt"`   // 仅取 client 发送且厂商支持的字段（类型只可能是基础类型）
   }
   type Xxx_Response struct {
       BaseResp struct { StatusCode int `json:"status_code"` } `json:"base_resp"`
       ImageUrls []string `json:"image_urls"`
   }
   ```

   规则：
   - 可选字段用指针（`*bool`、`*Struct`）+ `omitempty` 省略空值。
     - 坑点：**值类型结构体字段不会被 `omitempty` 省略** —— 会序列化成 `{}`，导致厂商报错如
       `invalid style: `。可选嵌套结构体务必用指针（`*Xxx_Style`）。
     - `*bool` 区分"未提供"与显式 `false`。
   - `Xxx_TemReq`：只提取 client 发送且厂商支持的字段，**不要盲目透传一切**；牢记其类型仅限
     基础类型/数组，拼装时再做转换。
     - 接收入参：`item.Req` 是 `map[string]interface{}`（前端 JSON 反序列化后的原始参数）。
       取数标准两步：`reqBytes, _ := json.Marshal(item.Req)` →
       `json.Unmarshal(reqBytes, &tem)`（其中 `tem` 为 `Xxx_TemReq`）。
   - `Xxx_Response`：必须覆盖厂商业务错误码，非零状态返回 `fmt.Errorf`，不要伪造成功。
   - `BuildRequest` 标准三步：`body, _ := json.Marshal(req)` → `Req.SetBody(body)` →
     `Req.SetRequestURI(url)`（+ 鉴权 header）。返回 `body`。
   - 运行时调用上下文：`BuildRequest` 由 task_svc 消费 NSQ 消息（收到生成请求）时调用，负责
     把 `item` 拼成发往厂商的 HTTP 请求；`ParseResponse` 在同步直接返回、或异步轮询查询时被
     调用，负责把厂商响应归一化为 `TaskResult`。写码时须保证两者字段映射一致。

2. **对齐已有适配器**（不要重复造轮子）：先读一个同形态适配器——同步图像 `miniMax-image.go`、
   `doubao-seedream-5-0-pro-260628.go`，异步视频 `miniMax-Hailuo3.go`；接口与
   `TaskResult`/`TaskResultUsage` 见 `common/model/req_pic_model.go`；Manager / `AllRegisterLLM`
   / `RegisterLLMModelAdapter` 见 `llm_api_interface.go`。

3. **注册并验证**：
   - **写码前先确认模型 key 是否已被注册**（避免重复实现或 first-wins 遮蔽）：
     `grep -rn 'RegisterLLMModelAdapter(<Key>' task_svc/internal/handler/llmange/`。若已存在，
     优先**扩展现有适配器**而非新建文件；确需新建时也要避开已被占用的 key。
   - 在 `llm_api_interface.go` 新增模型名常量（如 `var LLMMiniMax_Image = "image-01-live"`）。
   - 在 `AllRegisterLLM()` 注册（每个 key 只注册一次）。
   - **必须重新构建并重启进程** —— 旧二进制会显示老错误，让改动看起来无效。重启前先 vet：
     ```bash
     go build ./task_svc/... && go vet ./task_svc/internal/handler/llmange/...
     ```
   - 实跑验证：空字段是否被省略、枚举是否被厂商接受、响应能否解析。
   - 首次接入：把原始 `adapterResponse.Body()` 打一次日志（util.Error），便于与文档 diff，
     尽早修正字段名偏差。

## 阶段四：输出结论报告

代码完成后，向用户输出一份结论报告，并**落盘到仓库 `wiki/report/` 目录**（命名建议
`<模型名>_comparison_report.md` 或 `<模型名>_conclusion_report.md`）。报告明确两部分：
- **a. 已完成**：列出实际落地的能力、文件、注册的模型 key、验证结果。
- **b. 未完成及原因**：列出未实现/未验证的部分（如某条件必传分支未覆盖、厂商某枚举未实测、
  异步轮询间隔未调优等），并**指明具体原因**（缺文档、缺测试素材、厂商限制等），不隐瞒。

## 产物落盘位置汇总

- 阶段二技术文档：`wiki/plans/<模型名>_adapter_plan.md`
- 阶段四结论报告：`wiki/report/<模型名>_conclusion_report.md`

## 参考示例：MiniMax image-01-live

- 模型 key `image-01-live`，单一适配器 `miniMax-image.go` 同时覆盖 t2i（仅 prompt）
  与 i2i（由 client 的 `image_urls` 构造 `subject_reference`）。
- `buildImageSubjectRefs(v interface{})` 将 client 的 `image_urls`（string / `[]string` /
  `[]interface{}`）转换为 `[]*MiniMax_Image_SubjectRef{{Type:"character", ImageFile:url}}`；
  为空时返回 `nil`，从而省略 `subject_reference`（纯 t2i）。即"前端 array → 厂商嵌套数组"的典型拼装。
- `Style` 为 `*MiniMax_Image_Style`（指针），仅当提供 `style_type` 时才设置，否则省略。
- `aspect_ratio` 按白名单 map 校验，默认 `1:1`（注意 `21:9` 仅 `image-01` 支持，本变体排除）。
- `response_format` 硬编码为 `"url"`，使结果可直接用于 OSS 上传。
- `ParseResponse` 读取 `base_resp.status_code`（非零 → 错误），从 `image_urls`（兜底
  `image_base64`）填充 `TaskResult.Images`，并从 `metadata` 填充 `Usage.TotalTokens`。
