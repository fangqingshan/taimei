---
name: batch-import-curl
description: 根据用户提供的 serviceId、userId 和接口列表，构建本地调用 openApi 后端批量导入接口（ConfigController#batchImportRoute，POST /config/route/batchImport）的 curl 命令。只生成命令不执行。当用户提到批量导入接口、batchImport、构建本地调用 curl、或提供接口组ID和接口清单要求生成请求命令时使用。
---

# 批量导入接口 curl 构建助手

根据用户提供的必要信息，构建调用本地 openApi 后端批量导入接口的 curl 命令。**只生成命令，不执行调用。**

## 目标接口

- 后端方法：`com.taimeitech.open.backend.rest.ConfigController#batchImportRoute`
- 本地地址：`POST http://localhost:8080/config/route/batchImport`
- Content-Type：`application/json;charset=UTF-8`
- 响应结构：`{ "code": 0, "message": null, "data": [RoutePO...] }`，`data[]` 中每项包含 `id`（routeId）和 `interfaceId`

## 必要信息收集

向用户确认以下信息，缺失时必须询问，不要擅自假设：

| 信息项 | 说明 | 是否必填 |
|--------|------|----------|
| serviceId | 当前环境的接口组 ID（32 位十六进制主键） | 必填 |
| userId | 操作人用户 ID（32 位十六进制主键） | 必填 |
| 接口列表 | 接口中文名、接口路径、请求方法 | 必填，至少 1 个 |

**接口列表收集模板：**

| 接口中文名 | 接口路径 | 请求方法 |
|------------|----------|----------|
| 获取文件夹列表 | /external/getFolders | POST |

## 字段校验规则（生成前必须检查）

生成 curl 之前逐条校验，不合法时提示用户修正，不要带着非法值生成命令：

1. **method**：仅允许 `GET` / `POST` / `PUT` / `DELETE`（大写，后端正则 `^(GET|POST|PUT|DELETE)$`）
2. **path**：
    - 不能为空，不能含空白字符、`?`、`#`
    - 仅填后端短路径，禁止携带 `/api/{服务名}` 前缀
    - 缺少 `/` 前缀会自动补上；连续 `//` 会合并；末尾 `/` 会被去除
3. **nameCN**：不能为空，前后空格会被后端自动去除
4. **重复检查**：同一批请求中 `method:path` 组合不能重复

## curl 命令模板

```bash
curl --location --request POST 'http://localhost:8080/config/route/batchImport' \
--header 'Content-Type: application/json;charset=UTF-8' \
--data-raw '{
    "serviceId": "{serviceId}",
    "userId": "{userId}",
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
- `routes` 数组按用户提供的接口列表逐项填充，顺序保持一致
- 只有一个接口时也要用数组格式
- 请求体 JSON 中的字符串使用双引号；若用户的 nameCN 内含双引号需转义为 `\"`

## 工作流程

1. **收集信息**：按上表收集 serviceId、userId、接口列表，缺失项询问用户
2. **校验**：按字段校验规则逐条检查，发现问题列出让用户修正
3. **构建**：填充 curl 模板，将完整命令输出给用户
4. **补充提示**：告知用户后端会自动生成接口编码（规则：`{method小写}_{path_slug}`，如 `POST /external/getFolders` → `post_external_getfolders`），导入成功后从响应 `data[]` 提取 `id`（routeId）和 `interfaceId` 供后续使用

## 常见后端报错提示

构建命令时可提醒用户以下服务端校验错误，便于排查：

| 报错信息 | 原因 |
|----------|------|
| 接口组ID不存在 | serviceId 在当前环境数据库中不存在或已删除 |
| 用户ID不存在 | userId 在用户表中不存在 |
| 请求中存在重复的请求方法和接口路径 | 同批次内 method:path 重复 |
| 接口路径不能为空，且不能包含空白字符、查询参数或锚点 | path 含空格 / `?` / `#` |

## 注意事项

- **只生成 curl 命令，不执行任何调用**，除非用户明确要求执行
- Windows PowerShell 中 `curl` 是 `Invoke-WebRequest` 的别名，上述 Linux 风格 curl 无法直接运行；若用户需要在 PowerShell 中执行，可补充提示改用 Git Bash / WSL，或提供等效的 `Invoke-WebRequest` 写法