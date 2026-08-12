---
name: hengrui-local
description: 恒瑞本地化接口配置自动化工具。自动完成批量导入接口、生成 SQL 脚本、构建授权请求、生成测试 curl、同步 Confluence 文档和更新应用清单。使用场景：用户需要在恒瑞本地化环境配置接口时调用此 Skill。
---

# 恒瑞本地化接口配置助手

完全自动化的接口配置工具，一键完成从接口定义到文档更新的全流程。

## 快速开始

### 必填信息收集

向用户确认以下信息：

| 信息项 | 说明 | 示例 |
|--------|------|------|
| 应用名称 | 应用英文标识（appCode） | word-template-plus |
| appId | 应用主键 ID | 2c94ab0a7abdfdd8017ac27095670023 |
| serviceId | 测试环境服务唯一 ID（32 位十六进制） | 2c94bb8f82c930c20182e46bef280008 |
| 接口列表 | 接口中文名、后端路径、请求方法 | 见下方模板 |

### 接口信息收集模板

| 接口中文名 | 接口路径 | 请求方法 |
|------------|----------|----------|
| 获取文件夹列表 | /external/getFolders | POST |
| 文件归档 | /external/fileArchive | POST |

**接口路径填写强制规则**
- 仅填写后端短路径，禁止携带 `/api/{服务名}` 前缀
- 完整线上地址示例：`https://trial-openapi.hengrui.com/api/word-template-plus/external/getFolders`
- 录入仅保留：`/external/getFolders`
- 路径必须以 `/` 开头；不允许空格、`?`、`#`；连续斜杠会自动合并；末尾 `/` 会被网关自动剔除

---

## 工作流程

### 步骤 1：批量导入接口到测试环境

**自动执行：**
1. 构造请求体，调用 API：`POST http://trialos.test.com/api/open/config/route/batchImport`
2. 从响应提取 `interfaceId` 和 `routeId`
3. 保存真实 ID 供后续使用
4. 失败时显示错误并停止

**实现说明（重要！Windows PowerShell 兼容）：**
- 使用 PowerShell 原生 `Invoke-WebRequest` 命令（不使用 curl）
- curl 的 `-H` `-d` 参数在 Windows PowerShell 中不被识别
- 通过 `@{}` 哈希表构造请求头和请求体
- 使用 `ConvertTo-Json` 确保中文字符正确编码

**请求体结构：**
```json
{
  "serviceId": "{serviceId}",
  "userId": "2c948a9c79602d6c017a1cf9ba09005c",
  "routes": [
    {
      "nameCN": "{接口中文名}",
      "path": "{接口路径}",
      "method": "{请求方法}"
    }
  ]
}
```

**PowerShell 实现示例：**
```powershell
$body = @{
    serviceId = "2c94bb8f82c930c20182e46bef280008"
    userId = "2c948a9c79602d6c017a1cf9ba09005c"
    routes = @(
        @{
            nameCN = "接口中文名"
            path = "/接口路径"
            method = "POST"
        }
    )
} | ConvertTo-Json -Depth 10

$response = Invoke-WebRequest -Uri "http://trialos.test.com/api/open/config/route/batchImport" `
  -Method Post `
  -Headers @{"Content-Type"="application/json;charset=UTF-8"} `
  -Body $body

$data = $response.Content | ConvertFrom-Json
# 从 $data.data[].id 提取 routeId
# 从 $data.data[].interfaceId 提取 interfaceId
```

### 步骤 2：查询服务名称

**自动执行：**
```sql
SELECT id, service_name FROM t_service 
WHERE id = '{serviceId}' AND is_deleted = 0 LIMIT 1;
```

使用 `mysql-open` MCP 服务查询数据库，获取服务名供后续文档生成使用。

### 步骤 3：生成本地化 SQL 脚本

自动生成包含以下 6 条语句的 SQL 脚本：

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
| API 批量导入失败 | 显示具体错误，停止并提示修正 |
| 数据库查询失败 | 显示错误，使用默认服务名或让用户输入 |
| Confluence 操作失败 | 显示错误，提示检查权限或格式 |
| 清单更新失败 | 显示错误，提示手动检查页面格式 |

---

## 参考文档

- 接口批量导入数据填写指南：https://cf.taimei.com/pages/viewpage.action?pageId=174485732
- 批量导入接口 API：https://cf.taimei.com/pages/viewpage.action?pageId=174485741
- 恒瑞 openapi 初始化：https://cf.taimei.com/pages/viewpage.action?pageId=147558326
- openApi 本地化部署梳理：https://cf.taimei.com/pages/viewpage.action?pageId=147556575
- 接口清单参考格式：https://cf.taimei.com/pages/viewpage.action?pageId=174486561
- 全量应用接口清单：https://cf.taimei.com/pages/viewpage.action?pageId=174486450
