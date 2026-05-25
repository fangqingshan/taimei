---
name: yapi-sync
description: >-
  按 Controller#方法与 DTO 同步 YApi；写入前确认目标与摘要；可同步 @ApiOperation.notes。
disable-model-invocation: false
---

# YApi 同步（主 Skill）

## 硬约束

- 不新建 YApi 组/项目/分类，只用已有项。
- **写入前**：用户确认目标接口 + 更新摘要后再 `yapi_save_api`。

## MCP

`yapi_list_projects` · `yapi_get_categories` · `yapi_search_apis` · `yapi_get_api_desc` · `yapi_save_api`

## 模式

- 明确「新增」→ 选项目、选分类、组 payload、`yapi_save_api`、再问是否改 `@ApiOperation.notes`。
- 明确「更新/同步」→ `yapi_search_apis` 确认目标 → `yapi_get_api_desc` → 出摘要 → 确认后 `yapi_save_api`（带 `id`）→ 再问 notes。
- 不明确 → 先搜接口，有则更新，无则按新增。

## 字段与示例

- 字段说明顺序（命中即停）：`@ApiModelProperty` → JavaDoc/块注释 → 语义化中文（不明写「待补充」）。注解写什么文档写什么，不擅自改义（如「流程ID」勿改成「合同ID」）。
- 示例 JSON：无意义 `null`/空串；数组至少 1 项；有 DTO 则**完整展开**到叶子；`ActionResult<T>` 须含 `success`、`data`、`errorInfos` 且 `data` 按 `T` 展开。
- 标准 TM 头：缺则**追加**，已有**勿删**（常见：`TM-Header-TenantId`、`TM-Header-UserId`；合同模块常含 AppId、AccountId，与同接口已有文档对齐）。

## 合同模块版式（对齐参考：项目 937 / 接口 183475）

与 **`SignWebController#initiateSigningOneStep`** 同结构，避免描述/其他信息版式漂移。

| 项 | 要求 |
|----|------|
| **最后更新** | `desc` 与 `markdown` 中「最后更新：**yyyy-MM-dd**」须为**本次同步当日**（以执行环境日期为准；用户另行指定则以用户为准），两处同一天。 |
| **小节标题** | HTML 固定 `<h3>请求参数示例</h3>`、`<h3>响应参数示例</h3>`、`<h3>异常说明</h3>`；Markdown 固定 `##` 同名三节，大段之间用 `---`。勿用「Query 示例」「响应示例」等变体。 |
| **代码块** | `<pre><code class="language-json">` + 格式化 JSON。 |
| **异常表** | 三列：异常Code \| 异常信息（`ErrorCodeEnum` 英文）\| 描述（中文）。**只写本服务主动 `throw BusinessException(ErrorCodeEnum…)`**；不写第三方/下游透传、「动态」「见 errorInfos」等行。 |
| **响应** | YApi「返回数据」默认 **`res_body_is_json_schema: true` + draft-04 Schema**（与请求 Json 参数同级、表格化）。`desc/markdown` 的「响应参数示例」里、JSON 示例**前**加 **响应字段说明** 表（字段路径 \| 类型 \| 说明），与 Schema、DTO 一致。示例 JSON 仅作补充，**不能**代替 Schema。Schema 保存失败才降级为纯示例，但**必须**用该表补全字段。 |
| **GET 无 Body** | 勿加 `Content-Type: application/x-www-form-urlencoded` 等占位。 |
| **保存** | 同一次 `yapi_save_api` 必须同时传 `desc` 与 `markdown`，避免另一项被清空。 |

## 代码侧

1. Controller#方法 → 入参、返回、Header。
2. Service → 仅 **`ErrorCodeEnum` + 主动 throw** 写入异常表。
3. HTTP 文档形态与网关一致（如 `ActionResult<T>` 与 Controller 返回类型关系写清）。

## Anti-Pattern

未确认就写；无摘要就 `yapi_save_api`；删已有 Header；说明抄字段名或与注解不符；`ActionResult` 写成裸标量；示例残缺；版式偏离 183475；异常表写透传/动态；合同模块 `res_body` 只有大段 JSON、无 Schema（降级时也无响应字段表）。

## `@ApiOperation.notes`

- 默认：用户确认后再问是否同步 notes。
- **用户明确「notes 只要 YApi 链接」**：`notes` **仅**填该接口完整 URL（如 `http://mock.taimei.com/project/<pid>/interface/api/<apiId>`），不写项目号/接口号冗长说明。

## 写入前核对


- [ ] 目标接口、摘要已确认
- [ ] title / method / path / status；TM 头齐全（只追加）
- [ ] 合同版式：三节标题 + 响应字段表 + Schema（或降级表补全）；异常仅枚举主动分支
- [ ] `desc` + `markdown` 同批提交；**最后更新**日期正确
- [ ] notes 策略与用户要求一致（含「仅链接」）
