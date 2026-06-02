---
name: yapi-sync
description: >-
  按 Controller#方法与 DTO 同步 YApi；写入前确认目标与摘要；可同步 @ApiOperation.notes。
disable-model-invocation: false
---

# YApi 同步

## 硬约束

- 不新建 YApi 组/项目/分类，只用已有项。
- 写入前先确认**目标接口**与**更新摘要**，再调 `yapi_save_api`。

## 工作流程

- **新增**：选项目 → 选分类 → 组 payload → `yapi_save_api` → 询问是否同步 `@ApiOperation.notes`。
- **更新**：`yapi_search_apis` 定位 → `yapi_get_api_desc` 取现状 → 出摘要 → 确认后 `yapi_save_api`（带 `id`）→ 询问 notes。
- **不明确**：先搜，命中即更新，否则按新增。

MCP：`yapi_list_projects` · `yapi_get_categories` · `yapi_search_apis` · `yapi_get_api_desc` · `yapi_save_api`

## 内容规则

### 1. 描述 / 字段 desc：写业务行为，不抄内部代码

API 文档给**接入方**看，写他用得上的业务行为。同样适用于 Header / Query / Body 字段的 `desc`。

| ✅ 应写 | ❌ 不应写 |
|--|--|
| 业务场景与触发条件 | 内部服务/类/方法名（`xxxMicroService.xxxMethod`） |
| 入参/出参的业务含义 | 内部分支编排（「按 X × Y × Z 组合记录日志」） |
| 外部可观察影响（Cookie / 通知 / 链接失效 / 超限拦截） | 内部存储/缓存 key 生成（「用 MD5(X) 从 Redis 取 key」） |
| 与上下游接口的衔接顺序、前置/后置条件 | 日志/埋点、内部字段名 |

- **反例**：「调用 `cspMicroService.checkCertificationAuthCode` 解码后按「认证类型 × 成功/失败 × 认证方式 × 是否全部要素」组合记录活动日志」
- **正例**：「接收认证平台回调下发的 authCode，校验通过则身份认证成功并写入 Cookie 与临时认证 Token；失败累计超过租户阈值时链接失效并通知发起人」
- 顶部首行可保留 `Controller#方法` 全限定名作为定位锚点；正文除此之外**不应**出现内部代码符号。

### 2. 字段说明与示例

- 字段说明取值顺序（命中即停）：`@ApiModelProperty` → JavaDoc/块注释 → 语义化中文（不明写「待补充」）。注解写什么文档写什么，不擅自改义（如「流程ID」勿改成「合同ID」）。
- 示例 JSON：无意义 `null`/空串；数组至少 1 项；有 DTO 则**完整展开**到叶子。
- 请求头：仅给一个示例 `TM-Header-TenantId`（租户ID，必填字段标注是/否）；其他 Header 各服务业务不同，**不自动生成/修改**，留给用户手动维护。

### 3. 返回数据形态：默认 `ActionResult<T>` 包装

框架对 Controller 返回值统一包装为 `ActionResult<T>` 输出 HTTP body，**所有接口默认按此处理**——即便 Controller 方法签名上写的是 `Boolean`/`String`/`Long`/`DTO`，HTTP body 也是 `ActionResult<T>` 结构。

- Schema 顶层为 object，含 `success`、`data`、`errorInfos`；`data` 按 `T` 完整展开到叶子。`T` 即 Controller 方法签名上的返回类型。
- 示例 JSON 形如 `{"success": true, "data": <T 的样例值>, "errorInfos": []}`。
- desc 一句话标注框架包装关系：「Controller 返回 `T`，对外 HTTP 为 `ActionResult<T>`，`data` 为<业务含义>」。
- 失败由 `BusinessException` 抛出而非进返回体时，描述里**一句话**说明（如「成功 `data` 恒为 true，失败统一以 `BusinessException` 抛出」），异常表枚举主动 throw 的码。
- **例外**（罕见，需在源码确认）：Controller 显式绕过框架包装（`void`、原始 `ResponseEntity<T>`、文件流下载、第三方回调强制裸返回等）时，按实际响应体出 Schema，并在 desc 明确「未走 ActionResult 包装」。**不能**仅凭 yapi 旧文档里写「非 ActionResult 包装」就当作裸响应——很可能是历史误写。

### 4. 异常表

- 三列：异常Code \| 异常信息（`ErrorCodeEnum` 英文）\| 描述（中文）。
- **只写本服务主动 `throw BusinessException(ErrorCodeEnum…)`**；不写第三方/下游透传、「动态」「见 errorInfos」等行。

## 合同模块版式（对齐参考：项目 937 / 接口 183475）

与 **`SignWebController#initiateSigningOneStep`** 同结构，避免描述/其他信息版式漂移。

| 项 | 要求 |
|----|------|
| **小节标题** | HTML 固定 `<h3>请求参数示例</h3>`、`<h3>响应参数示例</h3>`、`<h3>异常说明</h3>`；Markdown 固定 `##` 同名三节，大段间用 `---`。勿用「Query 示例」「响应示例」等变体。 |
| **代码块** | `<pre><code class="language-json">` + 格式化 JSON。 |
| **最后更新** | `desc` 与 `markdown` 中「最后更新：**yyyy-MM-dd**」须为**本次同步当日**（以执行环境日期为准；用户另行指定则以用户为准），两处同一天。 |
| **响应** | `res_body_is_json_schema: true` + draft-04 Schema，顶层固定 `ActionResult<T>` object（`success/data/errorInfos`），`data` 按 `T`（Controller 方法签名返回类型）完整展开。`desc/markdown` 的「响应参数示例」里、JSON 示例**前**加 **响应字段说明** 表（字段路径 \| 类型 \| 说明），与 Schema/DTO 一致。示例 JSON 仅作补充，**不能**代替 Schema。Schema 保存失败才降级为纯示例，但**必须**用该表补全字段。 |
| **GET 无 Body** | 勿加 `Content-Type: application/x-www-form-urlencoded` 等占位。 |
| **保存** | 同一次 `yapi_save_api` 必须同时传 `desc` 与 `markdown`，避免另一项被清空。 |

## `@ApiOperation.notes`

- 默认：用户确认后再问是否同步。
- 用户明确「**只要 YApi 链接**」：`notes` 仅填该接口完整 URL（如 `http://mock.taimei.com/project/<pid>/interface/api/<apiId>`），不写项目号/接口号冗长说明。

## Anti-Pattern

- 未确认就写；无摘要就 `yapi_save_api`；擅自新增/修改/删除 Header（除示例 `TM-Header-TenantId` 外）
- 字段说明抄字段名或与注解不符
- 描述/字段 desc 塞内部服务/类/方法/Redis key/日志埋点细节（接入方看不懂）
- Controller 方法签名声明的 `Boolean/String/DTO` 被当作裸响应直接出顶层 Schema，漏掉框架默认的 `ActionResult<T>` 包装；`data` 没按 `T` 完整展开
- 示例残缺；版式偏离 183475
- 异常表写透传/动态；合同模块 `res_body` 只有大段 JSON、无 Schema（降级时也无响应字段表）

## 写入前核对

- [ ] 目标接口、更新摘要已确认
- [ ] title / method / path / status；仅给示例 `TM-Header-TenantId`，其他 Header 不自动改
- [ ] 描述/字段 desc 只写业务行为，无内部服务/类/方法/Redis/日志埋点细节
- [ ] 返回 Schema 顶层为 `ActionResult<T>` 结构（`success/data/errorInfos`），`data` 按 `T` 完整展开；如确属「显式绕过包装」例外，在 desc 标注
- [ ] 合同版式：三节标题 + 响应字段表 + Schema（或降级表补全）；异常仅枚举主动分支
- [ ] `desc` + `markdown` 同批提交；「最后更新」日期正确
- [ ] notes 策略与用户要求一致（含「仅链接」）
