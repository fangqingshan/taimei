---
name: hengrui-local
description: 恒瑞本地化接口配置自动化工具。支持用户直接粘贴任意格式接口信息（文本/表格/JSON/Postman导出或提供文件），自动抽取接口列表并构建批量导入接口 curl（serviceId/userId 可为空 ""，由用户补充）；用户回传最终完整 curl 后，按顺序执行批量导入、查询服务名、生成 SQL 脚本、构建授权请求、生成测试 curl、同步 Confluence 文档、更新应用清单。使用场景：用户需要在恒瑞本地化环境配置接口、提供接口清单要求生成批量导入命令、或要求完成接口配置全流程时调用此 Skill。
---

# 恒瑞本地化接口配置助手

**两阶段工作流：**

- **阶段一（交互）**：接收接口信息（任意格式原文或文件）→ 抽取接口三要素 → 构建批量导入 curl（serviceId / userId 置 `""`）→ 用户补充后回传最终完整 curl
- **阶段二（执行）**：执行批量导入 → 提取真实 ID → 按顺序完成查询服务名、SQL 脚本、授权请求、测试 curl、Confluence 文档、应用清单

---

## 阶段一：构建批量导入 curl

### 1.1 信息输入说明

| 信息项 | 说明 | 处理方式 |
|--------|------|----------|
| appCode / 应用名称 | 应用英文标识（如 word-template-plus） | 原文含则提取；缺失则询问 |
| appId | 应用主键 ID | 原文含则提取；缺失则询问（SQL 脚本需要） |
| serviceId | 测试环境服务唯一 ID（32 位十六进制） | 可为空字符串 `""`，用户后续自行补充 |
| userId | 操作人用户 ID（32 位十六进制） | 可为空字符串 `""`；也可使用默认值 `2c948a9c79602d6c017a1cf9ba09005c` |
| 接口列表 | 接口中文名、后端路径、请求方法 | 必填，至少 1 个；格式不固定，任意格式直接粘贴或提供文件 |

接口信息格式不固定，用户可直接**原文粘贴**（聊天记录/表格/文档/JSON/Postman 导出等），或提供文件路径（`.md`/`.txt`/`.csv`/`.json` 等文本格式直接用 Read 读取；`.xlsx` 等二进制格式请用户粘贴内容或另存为 CSV 后粘贴）。**不要要求用户整理成统一模板。**

### 1.2 信息抽取规则

从任意格式原文中抽取接口三要素（nameCN / path / method）：

1. **定位接口列表区域**：忽略寒暄、说明、前后缀文字，聚焦含 HTTP 方法词（`GET`/`POST`/`PUT`/`DELETE`，大小写均可）或路径特征（`/` 开头、含 `http://`/`https://` 的 URL）的内容
2. **逐条抽取三要素**：
   - **method**：匹配 `GET`/`POST`/`PUT`/`DELETE`，统一转大写
   - **path**：
     - 完整 URL → 仅取路径部分（去掉协议、域名、`?` 后的查询参数、`#` 后的锚点）
     - 含 `/api/{appCode}` 前缀 → 截掉前缀只保留短路径（如 `https://trial-openapi.hengrui.com/api/word-template-plus/external/getFolders` → `/external/getFolders`）
     - 裸路径直接使用；缺 `/` 前缀自动补
   - **nameCN**：优先取中文名（表格"接口名称/接口名/名称"列、行内中文说明等）；同一接口多个候选名时取最贴近业务含义的一个
3. **缺项处理**：
   - nameCN 缺失 → 用 `{method} {path}`（如 `POST /external/getFolders`）作占位，并在输出中明确提示用户补充中文名
   - method 缺失 → 无法推断，必须询问用户
   - path 缺失 → 无法推断，必须询问用户
   - 原文信息含糊、无法确定归属的条目 → 列出候选让用户确认后再构建
4. **去重**：提取后检查 `method:path` 组合，重复的保留一条并提示用户

**常见输入格式示例：**

| 格式 | 示例 |
|------|------|
| Markdown / 文本表格 | `获取文件夹列表 \| /external/getFolders \| POST` |
| 编号列表 | `1. 获取文件夹列表 /external/getFolders POST` |
| 聊天叙述 | `接口：获取文件夹列表；路径：/external/getFolders；请求方式：POST` |
| JSON | `{"nameCN": "获取文件夹列表", "path": "/external/getFolders", "method": "POST"}` |
| Postman / YApi 导出 | 以接口名作标题、含 `method` 与 `url`（可能是完整 URL，需截取短路径） |
| 仅路径+方法 | `/external/getFolders POST`（nameCN 缺，按缺项处理） |

### 1.3 字段校验规则（生成前必须检查）

1. **method**：仅允许 `GET` / `POST` / `PUT` / `DELETE`（大写，后端正则 `^(GET|POST|PUT|DELETE)$`）；原始小写写法自动转大写
2. **path**：
   - 不能为空，不能含空白字符、`?`、`#`
   - 仅填后端短路径，禁止携带 `/api/{appCode}` 前缀（抽取时已截掉）
   - 缺少 `/` 前缀自动补上；连续 `//` 会合并；末尾 `/` 会被网关自动剔除
3. **nameCN**：不能为空，前后空格会被后端自动去除；缺失时按缺项处理补占位
4. **serviceId / userId**：允许为空字符串 `""`；非空值需为 32 位十六进制主键，否则提示用户
5. **重复检查**：同一批请求中 `method:path` 组合不能重复

### 1.4 批量导入 curl 模板

```bash
curl --location --request POST 'http://trialos.test.com/api/open/config/route/batchImport' \
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
- `routes` 数组按抽取结果逐项填充，顺序保持一致；只有一个接口时也要用数组格式
- serviceId / userId 未提供时填入 `""`，输出后提示用户自行替换
- 请求体 JSON 字符串用双引号；nameCN 内含双引号需转义为 `\"`
- 输出完整 curl 时，同时附一份简短"抽取结果摘要"（接口名 / 路径 / 方法逐条列出，serviceId、userId 标注是否为空）便于用户核对
- 后端会自动生成接口编码（规则：`{method小写}_{path_slug}`，如 `POST /external/getFolders` → `post_external_getfolders`），SQL 脚本中需要用到

### 1.5 等待用户回传（关键检查点）

输出完整 curl 后**停止，等待用户**，不要自行继续：

- 用户补充 serviceId / userId 后回传**最终完整 curl** → 进入阶段二执行
- 或用户自行执行后回传**响应 JSON** → 进入阶段二（跳过执行步骤）

---

## 阶段二：按顺序执行后续流程

### 步骤 1：执行批量导入

收到用户回传的完整 curl 后执行。Windows PowerShell 中 `curl` 是 `Invoke-WebRequest` 别名、且 `\` 续行不生效，优先转换为等效 PowerShell 脚本执行（或使用 `curl.exe`）：

```powershell
$body = @{
    serviceId = "{serviceId}"
    userId = "{userId}"
    routes = @(
        @{
            nameCN = "{接口中文名}"
            path = "{接口路径}"
            method = "{请求方法}"
        }
    )
} | ConvertTo-Json -Depth 10

$response = Invoke-WebRequest -Uri "http://trialos.test.com/api/open/config/route/batchImport" `
  -Method Post `
  -Headers @{"Content-Type"="application/json;charset=UTF-8"} `
  -Body $body

$data = $response.Content | ConvertFrom-Json
```

- 校验 `$data.code -eq 0`；失败时显示 `message` 并停止
- **提取 ID**：`data[]` 顺序与 `routes[]` 顺序一致，逐项取 `id`（routeId）和 `interfaceId`，与接口一一对应保存，供 SQL 脚本使用
- 若响应顺序异常，按 data 项中的接口路径 / method 对应回接口

### 步骤 2：查询服务名称

```sql
SELECT id, service_name FROM t_service 
WHERE id = '{serviceId}' AND is_deleted = 0 LIMIT 1;
```

使用 `mysql-open` MCP 服务查询数据库，获取服务名供文档生成使用。查询失败或 serviceId 为空时，使用默认服务名或询问用户。

### 步骤 3：生成本地化 SQL 脚本

自动生成包含以下 6 条语句的 SQL 脚本（1 条添加应用 + 1 条添加分组 + 每个接口 1 条 t_interface + 每个接口 1 条 t_route）：

**1. 添加应用 (t_app)**
```sql
INSERT INTO `open`.`t_app` 
(`id`, `owner`, `name`, `app_code`, `user_id`, `app_id`, `type`, `available`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `summary`, `sequence`, `service_register_type`) 
VALUES 
('{appId}', NULL, '{appCode}', '{appCode}', '2c948a9c79602d6c017a1cf9ba09005c', '{appId}', 'InnerApp', 1, 0, NULL, NOW(), NULL, NOW(), 0, NULL, 1, 2);
```

**2. 添加分组 (t_service)**
```sql
INSERT INTO `open`.`t_service` 
(`id`, `owner`, `service_name`, `front_stamp`, `type`, `app_id`, `description`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `available`, `user_id`, `service_version`, `service_addr`, `sequence`) 
VALUES 
('{serviceId}', NULL, '{service_name}', '{appCode}', NULL, '{appId}', NULL, 0, NULL, NOW(), NULL, NULL, 0, 1, '2c948a9c79602d6c017a1cf9ba09005c', NULL, NULL, NULL);
```

**3. 添加接口 (t_interface) - 每个接口一条**
```sql
INSERT INTO `open`.`t_interface` 
(`id`, `service_id`, `name`, `name_cn`, `detail`, `summary`, `usage_scenario`, `announcements`, `owner`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `sequence`, `scope`) 
VALUES 
('{interfaceId}', '{serviceId}', '{接口编码}', '{接口中文名}', NULL, '{接口中文名}', NULL, NULL, NULL, 0, NULL, NOW(), NULL, NULL, 0, NULL, NULL);
```

**4. 添加路由 (t_route) - 每个接口一条**
```sql
INSERT INTO `open`.`t_route` 
(`id`, `interface_id`, `open_path`, `service_path`, `traffic_limit`, `type`, `summary`, `open_request_id`, `service_request_id`, `response_id`, `common_header_ids`, `route_version`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `need_user_auth`) 
VALUES 
('{routeId}', '{interfaceId}', '{接口路径}', '{接口路径}', NULL, '{请求方法}', '{接口中文名}', NULL, '', '', NULL, NULL, 0, NULL, NOW(), NULL, NOW(), 0, NULL);
```

**说明：** `{接口编码}` = `{method小写}_{path_slug}`（如 `post_external_getfolders`）；`{interfaceId}` / `{routeId}` 取自步骤 1 响应。

### 步骤 4：生成授权请求

**UAT 环境授权 curl：**
```bash
curl --location --request POST 'http://trialos.test.com/api/open/operate/interface/auth' \
--header 'Content-Type: application/json;charset=UTF-8' \
--data-raw '{
    "authRequests": [
        {
            "interfaceId": "{interfaceId1}",
            "secretId": "8ac0801295d66d74019633367b0a00a4",
            "expire": null,
            "alter": true
        }
    ],
    "envNames": ["test"]
}'
```

**PROD 环境授权 curl：**
```bash
curl --location --request POST 'http://trialos.test.com/api/open/operate/interface/auth' \
--header 'Content-Type: application/json;charset=UTF-8' \
--data-raw '{
    "authRequests": [
        {
            "interfaceId": "{interfaceId1}",
            "secretId": "8ac0817196334a35019637518a1d0003",
            "expire": null,
            "alter": true
        }
    ],
    "envNames": ["prod"]
}'
```

**说明：** 每个接口生成一条 authRequest，按顺序填充各自的 interfaceId。

### 步骤 5：生成测试 curl

**环境配置表：**
| 环境 | 租户ID | appKey | 域名 |
|------|--------|--------|------|
| UAT | hengruiUAT | 8ac0801295d66d74019633367b0a00a2 | https://trial-openapi-uat.hengrui.com |
| PROD | hengrui | 8ac0817196334a35019637518a010001 | https://trial-openapi.hengrui.com |

**签名参数固定值**
- nonce: 123456
- timestamp: 1685442994731
- sign_method: md5
- UAT sign: 40d695f18827db4e2fac5e868c3778c5
- PROD sign: d1c5760ee3f10f1c8c7042e3ff6cf319

**GET 请求示例（UAT）：**
```bash
curl --location --request GET 'https://trial-openapi-uat.hengrui.com/api/{appCode}{接口路径}?sign_method=md5&nonce=123456&timestamp=1685442994731&sign=40d695f18827db4e2fac5e868c3778c5' \
--header 'appKey: 8ac0801295d66d74019633367b0a00a2' \
--header 'tm-header-tenantid: hengruiUAT' \
--header 'Content-Type: application/json'
```

**POST 请求示例（PROD）：**
```bash
curl --location --request POST 'https://trial-openapi.hengrui.com/api/{appCode}{接口路径}?sign_method=md5&nonce=123456&timestamp=1685442994731&sign=d1c5760ee3f10f1c8c7042e3ff6cf319' \
--header 'appKey: 8ac0817196334a35019637518a010001' \
--header 'tm-header-tenantid: hengrui' \
--header 'Content-Type: application/json' \
--data '{}'
```

**说明：** GET 请求不带 body；POST 请求带 `--data '{}'`。UAT / PROD 环境各生成一份，按接口逐个填充 `{appCode}{接口路径}`。

### 步骤 6：自动创建 Confluence 文档

**文档标题：** `恒瑞openApi本地化添加接口-YYYY/MM/DD`

**文档格式：** 参考 https://cf.taimei.com/pages/viewpage.action?pageId=174486561

**文档结构：**
1. **应用信息** - 表格展示 appId、应用名称、appCode、用户ID
2. **分组信息** - 表格展示 serviceId、分组名称、前缀
3. **接口信息** - 表格展示所有接口的 ID、名称、中文名、路由 ID、路径、方法
4. **SQL 脚本** - 按类型分组（添加应用、分组、接口、路由），每种类型下展示对应的 SQL 语句
5. **授权请求** - UAT 和 PROD 环境分别展示 curl 命令
6. **测试接口** - 包含参数说明、固定值说明，UAT 和 PROD 环境各展示每个接口的测试 curl

**自动化处理：**
1. 调用 `confluence_get_page` 检查是否存在同日期文档
2. 如果存在则更新，不存在则在 `pageId=147556575` 下创建子页面
3. 文档采用参考格式，包含完整的应用信息、SQL 脚本、授权请求、测试 curl
4. 使用 markdown 格式，代码块采用 ```sql 和 ```bash 语法高亮

**文档位置：** https://cf.taimei.com/pages/viewpage.action?pageId=147556575

### 步骤 7：自动更新应用接口清单

**清单页面：** https://cf.taimei.com/pages/viewpage.action?pageId=174486450

**自动化处理：**
1. 检查 UAT 和 PROD 表中是否存在应用
2. 存在则在该应用下添加接口记录，不存在则新增应用分组
3. 更新汇总统计表（存在则更新接口数，不存在则新增应用统计行）
4. 调用 `confluence_update_page` 提交变更

---

## 配置信息参考

### API 地址
- 批量导入：`http://trialos.test.com/api/open/config/route/batchImport`
- 授权API：`http://trialos.test.com/api/open/operate/interface/auth`

### 数据库配置
- 库名：`open`
- 用户表：`t_app`
- 服务表：`t_service`
- 接口表：`t_interface`
- 路由表：`t_route`

### 固定用户 ID
- `userId: 2c948a9c79602d6c017a1cf9ba09005c`

### MCP 工具使用
- **数据库查询**：`mysql-open` 服务的 `query` 工具
- **Confluence 操作**：`mcp-atlassian` 服务的 confluence 相关工具

---

## 错误处理

| 错误类型 | 处理方案 |
|--------|--------|
| 批量导入时 serviceId 为空/不存在 | 接口组ID不存在，提示用户补充或修正后重试 |
| 批量导入时 userId 为空/不存在 | 用户ID不存在，提示用户补充或修正后重试 |
| API 批量导入失败（code != 0） | 显示 message，停止并提示修正 |
| 批量导入响应顺序与请求不一致 | 按响应中接口路径/method 对应回接口 |
| 数据库查询失败 | 显示错误，使用默认服务名或让用户输入 |
| Confluence 操作失败 | 显示错误，提示检查权限或格式 |
| 清单更新失败 | 显示错误，提示手动检查页面格式 |

---

## 注意事项

- **阶段一只生成 curl 命令，不执行任何调用**；收到用户回传的完整 curl 后才进入阶段二
- Windows PowerShell 中 `curl` 是 `Invoke-WebRequest` 的别名，Linux 风格 curl（含 `\` 续行）无法直接运行；执行时转换为 `Invoke-WebRequest` 或使用 `curl.exe`（可用 Git Bash / WSL）
- 若用户在阶段二中途只回传部分信息（如缺 appId），先询问补齐再继续，不要用占位符生成 SQL

---

## 参考文档

- 接口批量导入数据填写指南：https://cf.taimei.com/pages/viewpage.action?pageId=174485732
- 批量导入接口 API：https://cf.taimei.com/pages/viewpage.action?pageId=174485741
- 恒瑞 openapi 初始化：https://cf.taimei.com/pages/viewpage.action?pageId=147558326
- openApi 本地化部署梳理：https://cf.taimei.com/pages/viewpage.action?pageId=147556575
- 接口清单参考格式：https://cf.taimei.com/pages/viewpage.action?pageId=174486561
- 全量应用接口清单：https://cf.taimei.com/pages/viewpage.action?pageId=174486450
