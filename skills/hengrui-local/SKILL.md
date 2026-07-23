# 恒瑞本地化接口配置助手

帮助用户完成恒瑞本地化环境的接口配置工作，包括批量导入接口、生成 SQL 脚本、构建授权请求、生成测试调用 curl。

## 使用场景

当用户需要在恒瑞本地化环境配置接口时使用此 skill。完整流程：
1. 收集用户接口基础信息
2. 输出测试环境批量导入 curl（带格式化 JSON，兼容多终端）
3. 提供测试环境查询 SQL、生成本地化落地 Insert 脚本
4. 输出接口批量授权 curl，支持单 / 多接口授权
5. 生成 GET/POST 两类测试接口签名 curl，补齐签名规则说明

## 工作流程

### 第一步：收集用户信息

向用户确认以下必填信息：

| 信息项 | 说明 | 示例 |
|--------|------|------|
| 应用名称 | 应用英文标识（appCode） | etmf-webapi |
| 接口组名称 | 接口分组展示名称 | 对外开放接口 |
| 接口列表 | 接口中文名、后端路径、请求方法 | 见下方模板 |
| serviceId | 测试环境服务唯一 ID（32 位十六进制） | 8ac0801891933ef60192ae93bf18005e |
| userId | 操作人账号 ID | 2c948a9c79602d6c017a1cf9ba09005c |
| appId | 应用主键 ID | 8ac0801891933ef60192ae8a95680055 |

**接口信息收集模板**

| 接口中文名 | 接口路径 | 请求方法 |
|------------|----------|----------|
| 获取文件夹列表 | /external/getFolders | POST |
| 文件归档 | /external/fileArchive | POST |

**接口路径填写强制规则**
- 仅填写后端短路径，禁止携带 `/api/{服务名}` 前缀
- 完整线上地址示例：`https://trial-openapi.hengrui.com/api/etmf-webapi/external/getFolders`
- 录入仅保留：`/external/getFolders`
- 路径必须以 `/` 开头；不允许空格、`?`、`#`；连续斜杠会自动合并；末尾 `/` 会被网关自动剔除

### 第二步：生成测试环境批量导入请求

**关键约束**
`--data-raw` 内 JSON 必须带换行缩进格式化，否则工具识别为纯文本导致导入失败；同时提供两套 curl，按需输出：
- 换行格式化版（推荐 Linux/Mac/WSL 使用，可读性高）
- 单行压缩版（Windows PowerShell/IDEA Terminal 专用，规避换行符报错）

**1）换行格式化模板（标准输出）**

```bash
curl --location --request POST 'http://localhost:8080/config/route/batchImport' \
--header 'Content-Type: application/json' \
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

**2）单行兼容模板（Windows 终端专用）**

```bash
curl --location --request POST 'http://localhost:8080/config/route/batchImport' --header 'Content-Type: application/json' --data-raw '{"serviceId":"{serviceId}","userId":"{userId}","routes":[{"nameCN":"{接口中文名1}","path":"{接口路径1}","method":"{请求方法1}"},{"nameCN":"{接口中文名2}","path":"{接口路径2}","method":"{请求方法2}"}]}'
```

### 第三步：生成本地化环境 SQL 脚本

**前置查询 SQL（从测试环境拉取 ID）**

```sql
-- 查询应用基础信息
SELECT * FROM t_app WHERE id = '{appId}' AND is_deleted = 0;

-- 查询接口分组信息（用id查询，因为一个app下可能有多个分组）
SELECT * FROM t_service WHERE id = '{serviceId}' AND is_deleted = 0;

-- 查询当前服务下所有接口
SELECT * FROM t_interface WHERE service_id = '{serviceId}' AND is_deleted = 0;

-- 查询接口对应路由配置
SELECT * FROM t_route 
WHERE interface_id IN (
    SELECT id FROM t_interface 
    WHERE service_id = '{serviceId}' 
      AND is_deleted = 0
) AND is_deleted = 0;
```

**本地化入库 Insert 模板（替换查询到的真实 ID）**

```sql
-- 新增应用记录
INSERT INTO `open`.`t_app` (`id`, `owner`, `name`, `app_code`, `user_id`, `app_id`, `type`, `available`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `summary`, `sequence`, `service_register_type`) 
VALUES ('{app_id}', '{owner}', '{name}', '{app_code}', '{user_id}', '{app_uuid}', 'InnerApp', 1, 0, NULL, NOW(), NULL, NOW(), 0, '{summary}', 1, 2);

-- 新增接口分组
INSERT INTO `open`.`t_service` (`id`, `owner`, `service_name`, `front_stamp`, `type`, `app_id`, `description`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `available`, `user_id`, `service_version`, `service_addr`, `sequence`) 
VALUES ('{service_id}', '{owner}', '{service_name}', '{app_code}', NULL, '{app_id}', NULL, 0, NULL, NOW(), NULL, NULL, 0, 1, '{user_id}', NULL, NULL, NULL);

-- 单条接口记录（多接口循环复制）
INSERT INTO `open`.`t_interface` (`id`, `service_id`, `name`, `name_cn`, `detail`, `summary`, `usage_scenario`, `announcements`, `owner`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `sequence`, `scope`) 
VALUES ('{interface_id}', '{service_id}', '{接口英文名}', '{接口中文名}', NULL, NULL, NULL, NULL, '{owner}', 0, NULL, NOW(), NULL, NULL, 0, NULL, NULL);

-- 单条路由记录（与接口一一对应）
INSERT INTO `open`.`t_route` (`id`, `interface_id`, `open_path`, `service_path`, `traffic_limit`, `type`, `summary`, `open_request_id`, `service_request_id`, `response_id`, `common_header_ids`, `route_version`, `version`, `create_by`, `create_time`, `update_by`, `update_time`, `is_deleted`, `need_user_auth`) 
VALUES ('{route_id}', '{interface_id}', '{open_path}', '{service_path}', NULL, '{method}', NULL, NULL, '', '', NULL, NULL, 0, NULL, NOW(), NULL, NOW(), 0, NULL);
```

**ID 规范**
- 直接复用测试环境查询到的原始主键 ID
- 格式统一为 32 位十六进制字符串，示例：`8ac0817196334a350196574f11d40007`

### 第四步：生成授权请求报文

**环境密钥对照表**

| 环境 | 租户 ID | appKey | secretId |
|------|---------|--------|----------|
| UAT | hengruiUAT | 8ac0801295d66d74019633367b0a00a2 | 8ac0801295d66d74019633367b0a00a4 |
| PROD | hengrui | 8ac0817196334a35019637518a010001 | 8ac0817196334a35019637518a1d0003 |

**批量授权 curl 模板（支持多接口批量）**

```bash
curl --location --request POST 'http://localhost:8080/operate/interface/auth' \
--header 'Content-Type: application/json;charset=UTF-8' \
--data-raw '{
    "authRequests": [
        {
            "interfaceId": "{interface_id1}",
            "secretId": "{secretId}",
            "expire": null,
            "alter": true
        },
        {
            "interfaceId": "{interface_id2}",
            "secretId": "{secretId}",
            "expire": null,
            "alter": true
        }
    ],
    "envNames": ["test"]
}'
```

**参数说明：**
- `alter: true`：覆盖该接口已有授权配置
- `expire: null`：永久有效；如需限时授权填入时间戳

### 第五步：生成测试接口 curl

> **重要说明**：测试时使用恒瑞本地化的openApi域名访问。
> - UAT: https://trial-openapi-uat.hengrui.com
> - PROD: https://trial-openapi.hengrui.com

**测试参数固定值**
- nonce: 123456
- timestamp: 1685442994731
- sign_method: md5

**完整签名规则**
1. 提取 URL 全部请求参数，按 key 字母升序排序
2. 拼接格式：`key1=value1&key2=value2`
3. 末尾拼接对应环境 appSecret，整体 MD5 加密得到 sign 签名

**恒瑞本地化 AppKey 信息**

| 环境 | 租户ID | appKey | appSecret | sign |
|------|--------|--------|-----------|------|
| UAT | hengruiUAT | 8ac0801295d66d74019633367b0a00a2 | bad4210e2e023ab0d7b39f0423d4cd2fdb79ac81aba0131c68fe96d873390838 | 40d695f18827db4e2fac5e868c3778c5 |
| PROD | hengrui | 8ac0817196334a35019637518a010001 | 55e677fa2a07bf2e9d6a1ac922306c24a97b622f603b1e1d045ec802e1f9d221 | d1c5760ee3f10f1c8c7042e3ff6cf319 |

**UAT 环境测试 curl 模板**

```bash
curl --location --request POST 'https://trial-openapi-uat.hengrui.com/api/{appCode}{接口路径}?sign_method=md5&nonce=123456&timestamp=1685442994731&sign=40d695f18827db4e2fac5e868c3778c5' --header 'appKey: 8ac0801295d66d74019633367b0a00a2' --header 'tm-header-tenantid: hengruiUAT' --header 'Content-Type: application/json' --data '{}'
```

**PROD 环境测试 curl 模板**

```bash
curl --location --request POST 'https://trial-openapi.hengrui.com/api/{appCode}{接口路径}?sign_method=md5&nonce=123456&timestamp=1685442994731&sign=d1c5760ee3f10f1c8c7042e3ff6cf319' --header 'appKey: 8ac0817196334a35019637518a010001' --header 'tm-header-tenantid: hengrui' --header 'Content-Type: application/json' --data '{}'
```

**etmf-webapi 测试示例**

获取文件夹列表（UAT）：
```bash
curl --location --request POST 'https://trial-openapi-uat.hengrui.com/api/etmf-webapi/external/getFolders?sign_method=md5&nonce=123456&timestamp=1685442994731&sign=40d695f18827db4e2fac5e868c3778c5' --header 'appKey: 8ac0801295d66d74019633367b0a00a2' --header 'tm-header-tenantid: hengruiUAT' --header 'Content-Type: application/json' --data '{}'
```

文件归档（UAT）：
```bash
curl --location --request POST 'https://trial-openapi-uat.hengrui.com/api/etmf-webapi/external/fileArchive?sign_method=md5&nonce=123456&timestamp=1685442994731&sign=40d695f18827db4e2fac5e868c3778c5' --header 'appKey: 8ac0801295d66d74019633367b0a00a2' --header 'tm-header-tenantid: hengruiUAT' --header 'Content-Type: application/json' --data '{}'
```

**路径区分提醒**
批量导入仅填写短路径 `/external/xxx`；测试调用必须拼接前缀 `/api/{appCode}`，完整路径为 `/api/etmf-webapi/external/xxx`。

### 第六步：生成接口信息维护文档

> **功能**：自动生成符合参考格式的接口配置信息文档

**参考格式**：[恒瑞openApi本地化添加接口-2026/7/22](https://cf.taimei.com/pages/viewpage.action?pageId=174486561)

**输出内容**

系统自动生成一份完整的接口配置文档，包括：

1. **应用信息**
   - 应用 ID：`{appId}`
   - 应用编码：`{appCode}`
   - 负责人：`{owner}`

2. **分组信息**
   - 分组 ID：`{serviceId}`
   - 分组名称：`{接口组名称}`
   - 应用编码：`{appCode}`

3. **接口信息表**（来自步骤1的接口列表）
   - 接口 ID | 接口编码 | 接口名称 | 路由 ID | 接口路径 | 请求方法

4. **SQL脚本**（来自步骤3）
   - INSERT t_app（应用记录）
   - INSERT t_service（分组记录）
   - INSERT t_interface（接口记录）
   - INSERT t_route（路由记录）

5. **授权请求**（来自步骤4）
   - UAT 环境 curl
   - PROD 环境 curl

6. **测试接口**（来自步骤5）
   - UAT 环境测试 curl
   - PROD 环境测试 curl

**使用方式**

- 可以手动复制此文档内容到 Confluence
- 或者在 Confluence 中参考此格式创建新文档
- 文档命名：`恒瑞openApi本地化添加接口-{YYYY/M/D}`（例：2026/7/22）
- 创建位置：在 [openApi本地化部署梳理](https://cf.taimei.com/pages/viewpage.action?pageId=147556575) 页面下

### 第七步：增量更新全量接口清单

> **目标**：维护 [全量应用接口清单](https://cf.taimei.com/pages/viewpage.action?pageId=174486450)

**操作步骤**

1. **打开目标页面** (pageId=174486450)

2. **按照现有格式追加新接口行**
   - 找到对应的环境（UAT 或 PROD）
   - 找到对应的应用编码和服务分组
   - 在该分组下追加新接口行

3. **追加内容格式**（与现有行保持一致）

   ```
   应用编码 | 服务名称 | 接口编码 | 接口名称 | 接口路径 | 请求方法
   etmf-webapi | etmf_webapi接口组一 | post_external_getfolders | 获取文件夹列表 | /external/getFolders | POST
   ```

4. **同时更新 UAT 和 PROD 两个环境**

5. **如果是新应用/新服务**
   - 在相应环境下新增应用/服务分组
   - 然后追加接口行

**注意事项**

- 保持现有数据完整性（无覆盖、无删除）
- 保持层级结构一致（环境→应用→服务→接口）
- 接口编码、接口路径、请求方法等字段必须准确

### 第八步：更新全量清单的汇总统计

> **目标**：更新 [全量应用接口清单](https://cf.taimei.com/pages/viewpage.action?pageId=174486450) 中的汇总部分

**操作步骤**

1. **找到汇总表格**（位于页面底部）

2. **按应用更新统计数据**

   | 应用编码 | 服务数 | 接口数 |
   |---------|--------|--------|
   | etmf-webapi | 1 | 2 |

3. **更新规则**

   - 如果是新应用：新增应用行
   - 更新对应应用的服务数和接口数
   - 重新计算"合计"行的总数

4. **同步 UAT 和 PROD**

   > 两个环境的统计应保持对称（相同应用、相同服务数、相同接口数）

**完整工作流时间预估**

| 步骤 | 操作 | 耗时 | 自动化 |
|------|------|------|--------|
| 1 | 收集信息 | 2 min | 手工 |
| 2 | 生成导入 curl | 1 min | ✅ 自动 |
| 3 | 生成 SQL | 1 min | ✅ 自动 |
| 4 | 生成授权 curl | 1 min | ✅ 自动 |
| 5 | 生成测试 curl | 1 min | ✅ 自动 |
| 6 | 创建 Confluence 文档 | 1 min | ✅ 自动 |
| 7 | 维护应用接口清单 | 0 min | ✅ 自动 |
| 8 | 更新统计汇总 | 0 min | ✅ 自动 |
| **合计** | - | **8 min** | **87.5% 自动化** |

## 参考文档

- 接口批量导入数据填写指南：https://cf.taimei.com/pages/viewpage.action?pageId=174485732
- 批量导入接口 API：https://cf.taimei.com/pages/viewpage.action?pageId=174485741
- 恒瑞 openapi 初始化：https://cf.taimei.com/pages/viewpage.action?pageId=147558326
- openApi 本地化部署梳理：https://cf.taimei.com/pages/viewpage.action?pageId=147556575
- 接口清单参考格式：https://cf.taimei.com/pages/viewpage.action?pageId=174486561
- 全量应用接口清单：https://cf.taimei.com/pages/viewpage.action?pageId=174486450