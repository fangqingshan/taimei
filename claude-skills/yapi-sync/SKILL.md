---
name: yapi-sync
description: >-
  按 Controller#方法与 DTO 同步 YApi；写入前确认目标与摘要；可同步 @ApiOperation.notes。
disable-model-invocation: false
---

# YApi 同步（主 Skill）

## 硬约束

- 不新建 YApi 组/项目/分类，只用已有项。
- **写入前**：用户确认目标 + 更新摘要后再 `yapi_save_api`。
- `req_body_is_json_schema` / `res_body_is_json_schema` **必须为 `true`**。

## MCP

`yapi_list_projects` · `yapi_get_categories` · `yapi_search_apis` · `yapi_get_api_desc` · `yapi_save_api`

## 模式

- **明确「新增」** → 选项目、选分类、组 payload、`yapi_save_api` → 再问是否改 `@ApiOperation.notes`。
- **明确「更新/同步」** → `yapi_search_apis` 确认目标 → `yapi_get_api_desc` 读现状 → 出摘要（含与现有文档的差异）→ 确认后 `yapi_save_api`（带 `id`）→ 再问 notes。
- **不明确** → 先搜接口，有则更新，无则按新增。

## 一、Schema 格式 — YApi 兼容性关键

### ✅ 正确格式（YApi 可表格化渲染）

```json
{
  "type": "object",
  "required": ["field1"],
  "properties": {
    "accountId": {
      "type": "string",
      "description": "账户id"
    },
    "autoLogin": {
      "type": "boolean",
      "description": "是否需要自动登录"
    },
    "tags": {
      "type": "array",
      "description": "标签列表",
      "items": {
        "type": "object",
        "properties": {
          "id": {"type": "string", "description": "标签ID"},
          "name": {"type": "string", "description": "标签名"}
        }
      }
    }
  }
}
```

### ❌ 禁止项（YApi 渲染会坏掉）

| 禁止写法 | 原因 |
|---------|------|
| `"$schema": "http://json-schema.org/draft-04/schema#"` | YApi 不识别，表格渲染空白 |
| `"$id": "..."`, `"definitions": {...}` | YApi 不支持，被忽略 |
| `"additionalProperties": true/false` | 画蛇添足，可能干扰渲染 |

### ✅ `$$ref` 必填（YApi 表格渲染依赖它）

`$$ref` **不是禁止项，而是必须项**。参考同项目已正常渲染的分类接口（如 emailLogin `274458`），每个嵌套对象都必须加 `$$ref`：

```json
{
  "type": "object",
  "$$ref": "#/definitions/ActionResult«LoginResponseDto»",
  "properties": {
    "data": {
      "type": "object",
      "$$ref": "#/definitions/LoginResponseDto",
      "description": "返回数据",
      "required": [],
      "properties": { ... }
    },
    "errors": {
      "type": "array",
      "$$ref": "#/definitions/ErrorInfo",
      "items": {
        "type": "object",
        "properties": {
          "code": {"type": "string", "description": "错误code"},
          "message": {"type": "string", "description": "错误消息"},
          "arguments": {"type": "array", "items": {"type": "object", "properties": {}}, "description": "占位符参数"},
          "internationalized": {"type": "boolean", "description": "是否国际化"}
        }
      },
      "description": "错误消息"
    },
    "success": {"type": "boolean", "description": "true:成功；false:失败"}
  }
}
```

> 即使项目没有真实的 `definitions` 定义，YApi UI 也**依赖 `$$ref` 来识别类型并触发字段表格渲染**。每个嵌套层级（controller → data → items）都要加。

### 关键规则

- 嵌套对象**全部展开到叶子字段**，不要留 `"type": "object"` 空壳。
- 响应 `ActionResult<T>` 统一结构：`success` (boolean) + `data` (按 T 展开) + `errors` (array) — 字段名必须用 **`errors`**（非 `errorInfos`），和同项目已渲染接口保持一致。
- 必填字段加 `required: ["field1", "field2"]`，基于 `@Valid` / `@NotBlank` 等约束。
- `data` 对象加 `"required": []`（空数组），和 emailLogin 一致。

## 二、desc / markdown — 防止转义问题

**⚠️ 大坑：`desc` / `markdown` 中 JSON 代码块必须使用实际换行，不能出现 `\\n` 字面量。**

### 正确做法

`desc`（HTML 格式）中的 JSON 代码块：
```html
<pre><code class="language-json">{
  "mobile": "13800138000",
  "valid": "123456"
}</code></pre>
```

`markdown`（Markdown 格式）中的 JSON 代码块：
```markdown
```json
{
  "mobile": "13800138000",
  "valid": "123456"
}
```
```

> 注意：在 MCP 工具调用的 JSON 参数中，换行写为实际换行（multi-line string），或用 `\n` 转义序列（而非 `\\n`）。

### 文档结构

与同项目同分类已有接口对齐。默认三节，`---` 分隔：

| 节 | 说明 |
|----|------|
| 开头 | `com.xxx.Controller#method` 全路径 + 一句话功能描述，另起一行 `最后更新：yyyy-MM-dd` |
| `## 请求参数示例` | 格式化 JSON 代码块 |
| `## 响应参数示例` | **响应字段说明**表（字段路径 \| 类型 \| 说明）+ 空行 + JSON 示例 |
| `## 异常说明` | 三列异常表（异常Code \| 异常信息 \| 描述） |

> 字段表在前、JSON 示例在后，字段表保证即使 Schema 渲染失败也能看清所有字段。

### 异常表规则

- 只枚举 Controller 方法中**主动 throw** 的 `BusinessException(ErrorCodeEnum…)` / `BusinessException(CodeEnum…)`
- 不写第三方透传、RuntimeException、「见 errorInfos」等模糊描述
- 三列：Code / Message / 中文描述

## 三、Headers — 按接口场景精简

**不要无脑复制同分类接口的所有 TM 头，按实际需要配：**

| 场景 | 建议 Headers |
|------|-------------|
| **登录/注册/预登录**（用户未认证） | `Content-Type`（必填）+ `TM-Header-TenantId`（非必填） |
| **已登录接口**（需用户身份） | `Content-Type` + `TM-Header-TenantId` + `TM-Header-UserId` + `TM-Header-UserIp` + 按需 |

> 原则：**登录类接口 Headers 从简，已登录接口从同分类已有接口复制。**

## 四、字段说明

命中即停：`@ApiModelProperty` → JavaDoc/块注释 → 语义化中文（不写「待补充」）。注解写什么文档写什么，不擅自改义。

## 五、@ApiOperation.notes

- **新增**：保存后问用户「是否同步 notes 到代码？」
- **更新**：在摘要中展示现有 notes vs 代码中的 notes 差异
- **用户明确「notes 只要 YApi 链接」**：仅填完整 URL

## 六、保存后验证（不可跳过）

`yapi_save_api` 后立即调用 `yapi_get_api_desc` 检查：

- [ ] `Json参数` 已按表格字段渲染（不是纯 JSON 字符串）
- [ ] `返回数据` 字段已展开（success / data.* / errors），含 `$$ref`
- [ ] `desc` / `markdown` JSON 代码块无 `\n` 字面量（正确显示换行）
- [ ] Headers 按场景精简，无多余头
- [ ] 异常表正确展示

发现问题 → 修复后重新保存。

## 写入前核对

- [ ] 目标接口、摘要已确认
- [ ] title / method / path / status 正确
- [ ] Headers 按场景精简
- [ ] Schema 无 `$schema`，但**必须有 `$$ref`**（每个嵌套层级）
- [ ] 嵌套对象全部展开到叶子
- [ ] 响应字段名用 `errors`（非 `errorInfos`）
- [ ] `desc` / `markdown` JSON 代码块使用实际换行，无 `\\n`
- [ ] 三节完整：请求示例 + 响应字段表&示例 + 异常说明
- [ ] `desc` + `markdown` 同批提交；最后更新日期为当日
- [ ] 异常只枚举主动 throw 分支
- [ ] notes 策略已确认
- [ ] **保存后已拉取验证**