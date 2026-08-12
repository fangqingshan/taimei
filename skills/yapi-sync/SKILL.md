---
name: yapi-sync
description: >-
  按 Controller#方法与 DTO 同步 YApi；写入前确认目标与摘要；可同步 @ApiOperation.notes。
---

# YApi 同步（主 Skill）

## 硬约束

- 不新建 YApi 组/项目/分类，只用已有项。
- **写入前**：用户确认目标 + 更新摘要后再 `yapi_save_api`。
- `req_body_is_json_schema` / `res_body_is_json_schema` **必须为 `true`**。
- `yapi_save_api` 的 `catid` 必填。**当天新建的接口查不到分类**（`yapi_search_apis` 索引未更新、`yapi_get_categories` 列表不含），此时直接问用户要分类页 URL，从 `cat_{catid}` 提取 catid，不要猜（猜错会把接口挪到其他分类）。

## MCP

`yapi_list_projects` · `yapi_get_categories` · `yapi_search_apis` · `yapi_get_api_desc` · `yapi_save_api`

## 模式

- **明确「新增」**  选项目、选分类、组 payload、`yapi_save_api`  再问是否改 `@ApiOperation.notes`。
- **明确「更新/同步」**  `yapi_search_apis` 确认目标  `yapi_get_api_desc` 读现状  出摘要（含与现有文档的差异） 确认后 `yapi_save_api`（带 `id`） 再问 notes。
- **不明确**  先搜接口，有则更新，无则按新增。

## 一、Schema 格式 — YApi 兼容性关键

###  正确格式（YApi 可表格化渲染）

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

###  禁止项（YApi 渲染会坏掉）

| 禁止写法 | 原因 |
|---------|------|
| `"$schema": "http://json-schema.org/draft-04/schema#"` | YApi 不识别，表格渲染空白 |
| `"$id": "..."`, `"definitions": {...}` | YApi 不支持，被忽略 |
| `"additionalProperties": true/false` | 画蛇添足，可能干扰渲染 |

###  `$$ref` 规则（YApi 表格渲染依赖它）

参考标准接口：项目 789 的 `290013`（新增项目-开放接口）与 `274458`（emailLogin）。**只在「类型引用点」加 `$$ref`，不是每层都加**：

| 位置 | `$$ref` 写法 |
|------|-------------|
| 请求体根对象 | `#/definitions/{DTO类名}`，如 `#/definitions/OpenProjectCreateDTO` |
| 响应根对象 | `#/definitions/ActionResult«{T}»`，如 `#/definitions/ActionResult«String»` |
| Map 类型字段（extMap 等） | `#/definitions/Map`，`properties` 写空 `{}` |
| errors 数组 | items 简写 `{"$$ref":"#/definitions/ErrorInfo"}`，**不展开内部字段** |

####  请求体 Schema（data 为 DTO 对象）

```json
{
  "type": "object",
  "$$ref": "#/definitions/OpenProjectCreateDTO",
  "required": ["projectName", "programCode", "syncApps"],
  "properties": {
    "projectName": {"type": "string", "description": "项目名称"},
    "countryIds": {
      "type": "array",
      "description": "国家/地区的Id列表，为空则默认中国",
      "items": {"type": "string"}
    },
    "extMap": {
      "type": "object",
      "$$ref": "#/definitions/Map",
      "description": "更多项目信息，通过MQ透传",
      "properties": {}
    }
  }
}
```

####  响应 Schema — data 是基本类型（string/boolean/number）

`data` 直接写基本类型，**不要包一层 object**：

```json
{
  "type": "object",
  "$$ref": "#/definitions/ActionResult«String»",
  "properties": {
    "success": {"type": "boolean", "description": "true:成功；false:失败"},
    "data": {"type": "string", "description": "项目ID"},
    "errors": {
      "type": "array",
      "items": {"$$ref": "#/definitions/ErrorInfo"},
      "description": "错误消息"
    }
  }
}
```

####  响应 Schema — data 是对象/List（展开到叶子）

`ActionResult«LoginResponseDto»` 时，`data` 展开 DTO 全部字段并加 `"required": []`：

```json
{
  "type": "object",
  "$$ref": "#/definitions/ActionResult«LoginResponseDto»",
  "properties": {
    "success": {"type": "boolean", "description": "true:成功；false:失败"},
    "data": {
      "type": "object",
      "$$ref": "#/definitions/LoginResponseDto",
      "description": "返回数据",
      "required": [],
      "properties": { "...展开到叶子字段..." }
    },
    "errors": {
      "type": "array",
      "items": {"$$ref": "#/definitions/ErrorInfo"},
      "description": "错误消息"
    }
  }
}
```

#### data 是 List 时

```json
"data": {
  "type": "array",
  "$$ref": "#/definitions/List«UserInfoVo»",
  "description": "返回数据",
  "items": {
    "type": "object",
    "$$ref": "#/definitions/UserInfoVo",
    "properties": { "...展开到叶子字段..." }
  }
}
```

> `errors` 数组**永远简写**（items 只留 `$$ref`），错误字段细节由「响应字段说明」表承载；业务 DTO/VO 对象才展开到叶子。

### 关键规则

- 业务 DTO/VO 嵌套对象**全部展开到叶子字段**，不要留 `"type": "object"` 空壳（Map 类型除外，Map 就是空 `properties`）。
- 响应 `ActionResult<T>` 统一结构：`success` (boolean) + `data` (按 T 展开) + `errors` (array) — 字段名必须用 **`errors`**（非 `errorInfos`），和同项目已渲染接口保持一致。
- 必填字段加 `required: ["field1", "field2"]`（根对象层级），基于 `@Valid` / `@NotBlank` 等约束。
- `data` 为对象时加 `"required": []`（空数组），和标准接口一致。
- **请求体 `Json参数` 根对象必加 `$$ref: "#/definitions/{请求DTO类名}"`**，缺失会影响表格渲染。
- **`data` 是基本类型时禁止附加 `$$ref` / `required: []`**（如 `#/definitions/String` 是错的，只写 `{"type":"string","description":"..."}`）。

### Java 类型 → Schema 类型映射

| Java 类型 | Schema 写法 |
|-----------|------------|
| `String` / 枚举 | `{"type": "string"}` |
| `Integer` / `Long` / `int` | `{"type": "number"}`（或 `"integer"`） |
| `Boolean` | `{"type": "boolean"}` |
| `Date` / `LocalDateTime` | `{"type": "string"}`，description 注明格式如 `yyyy-MM-dd HH:mm:ss` |
| `BigDecimal` | `{"type": "number"}` |
| `List<基本类型>` | `{"type": "array", "items": {"type": "string"}}` |
| `List<DTO>` | `{"type": "array", "items": {对象展开 + $$ref}}` |
| `Map<String,?>` | `{"type": "object", "$$ref": "#/definitions/Map", "properties": {}}` |
| DTO / VO 对象 | `{"type": "object", "$$ref": "#/definitions/类名", "properties": {展开}}` |

## 二、desc / markdown — 格式隔离 + 防转义

### ⛔ 硬边界：desc = 纯 HTML，markdown = 纯 Markdown，严禁混写（290875 翻车根源）

| 字段 | 允许的内容 | 严禁出现 |
|------|-----------|----------|
| `desc`（接口描述） | 只用 `<p> <br> <h2> <hr> <table> <thead> <tbody> <tr> <th> <td> <strong> <pre><code>` 等 HTML 标签 | `## 标题`、`\| 表格 \|`、` ```json ` 等任何 Markdown 语法 |
| `markdown`（接口文档） | 只用 `##` 标题、Markdown 表格、` ```json ` 围栏代码块 | 任何 HTML 标签 |

> **YApi 渲染 desc 时不解析 Markdown**：管道表格会被当纯文本连续输出，页面上表格挤成一行（`| 字段路径 | 类型 | 说明 | |---|---|---| | success | ...`）。两个字段内容相同、格式不同，提交前逐字段自检。

**️ 大坑：`desc` / `markdown` 中 JSON 代码块必须使用实际换行，不能出现 `\\n` 字面量。**

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

与同项目同分类已有接口对齐（标准参考：`290013`）。默认三节，`<hr>`（markdown 中 `---`）分隔：

| 节 | 说明 |
|----|------|
| 开头 | `com.xxx.Controller#method` 全路径 + 一句话功能描述，另起一行 `最后更新：yyyy-MM-dd` |
| `## 请求参数示例` | 格式化 JSON 代码块 |
| `## 响应参数示例` | **响应字段说明**表（字段路径 \| 类型 \| 说明）+ 空行 + JSON 示例 |
| `## 异常说明` | 三列异常表（异常Code \| 异常信息 \| 描述） |

> 字段表在前、JSON 示例在后，字段表保证即使 Schema 渲染失败也能看清所有字段。

### desc HTML 标准模板（照抄结构，只换内容）

```html
<p>com.taimeitech.econfig.open.controller.SyncApiController#openCreateProject<br>
第三方同步服务 - 新增项目（开放接口）<br>
最后更新：2026-07-08</p>
<hr>
<h2>请求参数示例</h2>
<pre><code data-language="json" class="lang-json">{
  "projectName": "Demo Clinical Trial",
  "syncApps": ["eCooperate", "eArchives"]
}
</code></pre>
<hr>
<h2>响应参数示例</h2>
<p><strong>响应字段说明</strong></p>
<table>
<thead>
<tr>
<th>字段路径</th>
<th>类型</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr>
<td>success</td>
<td>boolean</td>
<td>true:成功；false:失败</td>
</tr>
<tr>
<td>data</td>
<td>string</td>
<td>项目ID</td>
</tr>
<tr>
<td>errors</td>
<td>array[ErrorInfo]</td>
<td>错误消息</td>
</tr>
<tr>
<td>errors[].code</td>
<td>string</td>
<td>错误code</td>
</tr>
<tr>
<td>errors[].message</td>
<td>string</td>
<td>错误消息</td>
</tr>
<tr>
<td>errors[].arguments</td>
<td>array</td>
<td>占位符参数</td>
</tr>
<tr>
<td>errors[].internationalized</td>
<td>boolean</td>
<td>是否国际化</td>
</tr>
</tbody>
</table>
<pre><code data-language="json" class="lang-json">{
  "success": true,
  "data": "PROJECT-001",
  "errors": null
}
</code></pre>
<hr>
<h2>异常说明</h2>
<table>
<thead>
<tr>
<th>异常Code</th>
<th>异常信息</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>eConfig0052</td>
<td>Distribution system is not supported</td>
<td>syncApps 传入的系统名称不匹配 OMP 中已注册的系统</td>
</tr>
</tbody>
</table>
```

**模板要点**：
- 代码块用 `<pre><code data-language="json" class="lang-json">`，内部是**真实换行**的格式化 JSON
- 响应字段说明表的 `errors[].xxx` 行**固定七行**（success / data / errors / errors[].code / message / arguments / internationalized），data 行按实际类型写
- 异常信息列写**错误码原文**（通常英文含 `{0}` 占位符），描述列写中文触发场景

### 异常表规则

- 只枚举 Controller 方法中**主动 throw** 的 `BusinessException(ErrorCodeEnum…)` / `BusinessException(CodeEnum…)`
- 不写第三方透传、RuntimeException、「见 errorInfos」等模糊描述
- 三列：Code / Message / 中文描述

## 三、Headers — 按接口场景配置

**不要无脑复制，但标准场景直接照抄下表（以 290013 实际配置为准）：**

| 场景 | Headers 配置 |
|------|-------------|
| **已登录接口 / 开放接口（open/**）** | `Content-Type`（必填）+ `TM-Header-TenantId`（必填，desc：通用参数 租户ID 必填）+ `TM-Header-UserId`（必填，desc：通用参数 用户ID 必填）+ `TM-Header-UserIp`（必填，desc：通用参数）+ `TM-Header-AccountId`（非必填，desc：通用参数）+ `TM-Header-Locale`（非必填，desc：通用参数） |
| **登录/注册/预登录**（用户未认证） | `Content-Type`（必填）+ `TM-Header-TenantId`（非必填） |

> 原则：**已登录/开放接口直接复制同分类标准接口的全套头（含 required 标志和 desc）；只有登录类接口才从简。**不确定时先 `yapi_get_api_desc` 拉同分类接口看实际配置。

## 四、字段说明

命中即停：`@ApiModelProperty`  JavaDoc/块注释  语义化中文（不写「待补充」）。注解写什么文档写什么，不擅自改义。

## 五、@ApiOperation.notes

- **新增**：保存后问用户「是否同步 notes 到代码？」
- **更新**：在摘要中展示现有 notes vs 代码中的 notes 差异
- **用户明确「notes 只要 YApi 链接」**：仅填完整 URL

## 六、保存后验证（不可跳过）

`yapi_save_api` 后立即调用 `yapi_get_api_desc` 检查：

- [ ] `Json参数` 已按表格字段渲染（不是纯 JSON 字符串）
- [ ] `返回数据` 字段已展开（success / data / errors），根对象与类型引用点含 `$$ref`
- [ ] `desc` / `markdown` JSON 代码块无 `\n` 字面量（正确显示换行）
- [ ] desc 三节结构完整（开头三行 + 请求示例 + 响应字段表&示例 + 异常说明）
- [ ] Headers 按场景配置，required 标志和 desc 正确
- [ ] 异常表正确展示（Code / Message / 描述三列）

发现问题  修复后重新保存。

## 七、历史翻车案例（生成前必读，避免重蹈覆辙）

### 案例：290875 添加项目管理员（2026-08-05）

| 缺陷 | 错误写法 | 正确写法 |
|------|---------|---------|
| desc 写成 Markdown，页面表格挤成一行 | desc 里用 `## 请求参数示例` + `\| 字段 \|` | desc 纯 HTML（见 HTML 标准模板） |
| errors 未简写 | errors.items 展开 code/message/arguments/internationalized | `{"items": {"$$ref": "#/definitions/ErrorInfo"}}` |
| data 为 string 却加多余属性 | `"$$ref": "#/definitions/String"` + `"required": []` | `{"type": "string", "description": "..."}` |
| 请求体根对象缺 $$ref | 无 `$$ref` | `"$$ref": "#/definitions/{DTO类名}"` |
| 响应字段表只有 5 行 | 缺 errors[].arguments / internationalized | 固定 7 行 |

## 写入前核对

- [ ] 目标接口、摘要已确认
- [ ] title / method / path / status 正确
- [ ] Headers 按场景配置（已登录/开放接口全套，登录类从简）
- [ ] Schema 无 `$schema`；`$$ref` 只加在类型引用点（根对象 / Map / errors items）
- [ ] 业务 DTO/VO 展开到叶子；data 是基本类型时不包 object；errors 永远简写
- [ ] 响应字段名用 `errors`（非 `errorInfos`）
- [ ] `desc` 是纯 HTML（无 `##` / `\| \|` / ` ``` ` 等 Markdown 语法）；`markdown` 是纯 Markdown（无 HTML 标签）
- [ ] `desc` / `markdown` JSON 代码块使用实际换行，无 `\\n`
- [ ] desc 按 HTML 标准模板：开头三行 + 三节 `<hr>` 分隔；响应字段表固定七行
- [ ] `desc` + `markdown` 同批提交；最后更新日期为当日
- [ ] 异常只枚举主动 throw 分支，信息列写错误码原文，描述列写中文场景
- [ ] notes 策略已确认
- [ ] **保存后已拉取验证**
