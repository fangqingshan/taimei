---
name: batch-import-curl
description: 根据用户提供的 serviceId、userId 和接口列表，构建本地调用 openApi 后端批量导入接口（ConfigController#batchImportRoute，POST /config/route/batchImport）的 curl 命令。接口信息格式不固定时，可直接粘贴原始信息（聊天记录/表格/文档/JSON/Postman导出等任意格式），由本技能自动抽取接口名、路径、请求方法；serviceId 和 userId 允许留空（""）由用户后续自行填写。只生成命令不执行。当用户提到批量导入接口、batchImport、构建本地调用 curl、或粘贴接口清单要求生成请求命令时使用。
---

# 批量导入接口 curl 构建助手

根据用户提供的必要信息，构建调用本地 openApi 后端批量导入接口的 curl 命令。**只生成命令，不执行调用。**

## 目标接口

- 后端方法：`com.taimeitech.open.backend.rest.ConfigController#batchImportRoute`
- 本地地址：`POST http://localhost:8080/config/route/batchImport`
- Content-Type：`application/json;charset=UTF-8`
- 响应结构：`{ "code": 0, "message": null, "data": [RoutePO...] }`，`data[]` 中每项包含 `id`（routeId）和 `interfaceId`

## 信息输入说明

| 信息项 | 说明 | 处理方式 |
|--------|------|----------|
| serviceId | 当前环境的接口组 ID（32 位十六进制主键） | 非必填，可为空字符串 `""`，用户后续自行替换 |
| userId | 操作人用户 ID（32 位十六进制主键） | 非必填，可为空字符串 `""`，用户后续自行替换 |
| 接口列表 | 接口中文名、接口路径、请求方法 | 必填，至少 1 个；格式不固定，直接粘贴原文由本技能抽取 |

用户可直接将别人提供的接口信息**原文粘贴**过来，无需整理成统一模板。

## 信息抽取规则

从任意格式的原文中抽取接口三要素（nameCN / path / method）。不要要求用户先整理格式。

### 抽取流程

1. **定位接口列表区域**：忽略寒暄、说明、前后缀文字，聚焦包含 HTTP 方法词（`GET`/`POST`/`PUT`/`DELETE`，大小写均可）或路径特征（`/` 开头、或含 `http://`/`https://` 的 URL）的内容
2. **逐条抽取三要素**：
   - **method**：匹配 `GET`/`POST`/`PUT`/`DELETE`，统一转大写
   - **path**：
     - 完整 URL → 仅取路径部分（去掉协议、域名、`?` 后的查询参数、`#` 后的锚点）
     - 含 `/api/{服务名}` 前缀 → 截掉前缀只保留短路径（如 `https://x.com/api/tms/external/getFolders` → `/external/getFolders`）
     - 裸路径直接使用；缺 `/` 前缀自动补
   - **nameCN**：优先取中文名（表格"接口名称/接口名/名称"列、行内中文说明等）；同一接口出现多个候选名时取最贴近业务含义的一个
3. **缺项处理**：
   - nameCN 缺失：用 `{method} {path}`（如 `POST /external/getFolders`）作占位，并在输出中明确提示用户补充中文名
   - method 缺失：无法推断，必须询问用户
   - path 缺失：无法推断，必须询问用户
   - 原文信息含糊、无法确定归属的条目：列出候选让用户确认后再构建
4. **去重**：提取后检查 `method:path` 组合，重复的保留一条并提示用户

### 常见输入格式示例

| 格式 | 示例 |
|------|------|
| Markdown / 文本表格 | `获取文件夹列表 \| /external/getFolders \| POST` |
| 编号列表 | `1. 获取文件夹列表 /external/getFolders POST` |
| 聊天叙述 | `接口：获取文件夹列表；路径：/external/getFolders；请求方式：POST` |
| JSON | `{"nameCN": "获取文件夹列表", "path": "/external/getFolders", "method": "POST"}` |
| Postman / YApi 导出 | 以接口名作标题、含 `method` 与 `url`（可能是完整 URL，需截取短路径） |
| 仅路径+方法 | `/external/getFolders POST`（nameCN 缺，按缺项处理） |

## 字段校验规则（生成前必须检查）

生成 curl 之前逐条校验，不合法时提示用户修正，不要带着非法值生成命令：

1. **method**：仅允许 `GET` / `POST` / `PUT` / `DELETE`（大写，后端正则 `^(GET|POST|PUT|DELETE)$`）；识别到的原始写法（如小写 `post`）自动转大写
2. **path**：
    - 不能为空，不能含空白字符、`?`、`#`
    - 仅填后端短路径，禁止携带 `/api/{服务名}` 前缀（抽取时已截掉）
    - 缺少 `/` 前缀会自动补上；连续 `//` 会合并；末尾 `/` 会被去除
3. **nameCN**：不能为空，前后空格会被后端自动去除；缺失时按缺项处理补占位
4. **serviceId / userId**：允许为空字符串 `""`；若为非空值，需为 32 位十六进制主键，否则提示用户
5. **重复检查**：同一批请求中 `method:path` 组合不能重复

## curl 命令模板

```bash
curl --location --request POST 'http://localhost:8080/config/route/batchImport' \
--header 'Content-Type: application/json;charset=UTF-8' \
--data-raw '{
    "serviceId": "{serviceId 或 \"\"}",
    "userId": "{userId 或 \"\"}",
    "routes": [
        {
            "nameCN": "{接口中文名1}",
            "path": "{接口路径1}",
            "method": "{请求方法1}"
        },
        {
            "nameCN": "{接口中文名2}",
            "path": "{接口路径2}",
            "method": "{请求方法2}"
        }
    ]
}'
```

**说明：**
- `routes` 数组按抽取结果逐项填充，顺序保持一致
- 只有一个接口时也要用数组格式
- serviceId / userId 未提供时填入 `""`，并在输出后提示用户自行替换
- 请求体 JSON 中的字符串使用双引号；若 nameCN 内含双引号需转义为 `\"`

## 工作流程

1. **接收信息**：用户直接粘贴原始接口信息（无需整理成模板）；serviceId / userId 若原文含则提取，否则置 `""`
2. **抽取与校验**：按"信息抽取规则"抽取三要素，按"字段校验规则"逐条检查；有歧义或缺失时先列出问题与候选，让用户确认或修正
3. **构建**：填充 curl 模板，将完整命令输出给用户；同时附一份简短"抽取结果摘要"（接口名 / 路径 / 方法逐条列出，serviceId、userId 标注是否为空）便于核对
4. **补充提示**：告知用户后端会自动生成接口编码（规则：`{method小写}_{path_slug}`，如 `POST /external/getFolders` → `post_external_getfolders`）；serviceId / userId 为空时提醒替换后再执行；导入成功后从响应 `data[]` 提取 `id`（routeId）和 `interfaceId` 供后续使用

## 常见后端报错提示

构建命令时可提醒用户以下服务端校验错误，便于排查：

| 报错信息 | 原因 |
|----------|------|
| 接口组ID不存在 | serviceId 为空、在当前环境数据库中不存在或已删除 |
| 用户ID不存在 | userId 为空、在用户表中不存在 |
| 请求中存在重复的请求方法和接口路径 | 同批次内 method:path 重复 |
| 接口路径不能为空，且不能包含空白字符、查询参数或锚点 | path 含空格 / `?` / `#` |

## 注意事项

- **只生成 curl 命令，不执行任何调用**，除非用户明确要求执行
- serviceId / userId 留空时，直接执行会得到"接口组ID不存在 / 用户ID不存在"报错，属预期行为，提醒用户替换后重试
- Windows PowerShell 中 `curl` 是 `Invoke-WebRequest` 的别名，上述 Linux 风格 curl 无法直接运行；若用户需要在 PowerShell 中执行，可补充提示改用 Git Bash / WSL，或提供等效的 `Invoke-WebRequest` 写法
