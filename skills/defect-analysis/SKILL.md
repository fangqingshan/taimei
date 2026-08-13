---
name: defect-analysis
description: 根据 Jira 缺陷单（缺陷号或链接）和代码改动（GitLab MR 链接或本地 git 提交），生成标准四段式缺陷分析（问题产生原因 / 影响范围 / 纠正措施CA / 预防措施PA）。当用户提供 Jira bug 链接、缺陷号、GitLab MR 链接，并要求写缺陷分析、CA/PA、缺陷复盘、根因分析时使用。
---

# 缺陷分析（Defect Analysis）

修复 bug 后，基于 Jira 缺陷单 + 代码改动（GitLab MR 或本地 git），生成标准四段式缺陷分析，在对话中输出供用户审阅。

## 工作流程

```css
进度清单：
- [ ] 步骤 1：读取 Jira 缺陷单（用户提供了 Jira 链接/缺陷号时）
- [ ] 步骤 2：获取代码改动（GitLab MR / 本地 commit）
- [ ] 步骤 3：结合代码上下文定位根因
- [ ] 步骤 4：按模板输出四段式分析供审阅
```

## 输入形式识别

| 用户输入 | 解析结果 | 说明 |
|----------|----------|------|
| Jira 链接 `https://jira.taimei.com/browse/TROS-123` | issue_key = `TROS-123` | 取 `/browse/` 后一段 |
| 仅缺陷号 `TROS-123` | issue_key = `TROS-123` | 直接使用 |
| GitLab MR 链接 `https://gitlab.taimei.com/group/project/-/merge_requests/456` | project_id = `group/project`，merge_request_iid = `456` | 取 `/-/merge_requests/` 前的路径为 project_id，其后数字为 iid |
| 本地 commit / 分支 / 未提交改动 | 走本地 git 流程 | 见步骤 2 |

**说明：**
- Jira 与 MR 可能只提供其一，能取到什么就用什么；不要因为缺一项而卡住
- 只有 MR 没有缺陷号时：可先基于 MR 分析，输出时缺陷号标注为待用户补充
- 只有缺陷号没有 MR/提交时：先尝试从 Jira 关联开发信息（修复分支/提交）中找代码，找不到则询问用户

## 步骤 1：读取 Jira 缺陷单

调用 MCP 工具（mcp-atlassian）：

- `jira_get_issue`：`issue_key`、`fields="summary,description,priority,status,issuetype,comment"`、`comment_limit=20`
- 描述若只有截图（形如 `![](image-xxx.png)`），调用 `jira_get_issue_images` 获取截图内容辅助理解；仍不清楚时向用户询问缺陷现象
- 留意缺陷单中的复现步骤、涉及租户、版本信息，这些是写"影响范围"的依据
- 如缺陷单含修复分支/提交信息（如 GitLab 集成），记录备用

## 步骤 2：获取代码改动

### 方式 A：用户提供 GitLab MR 链接（优先）

按"输入形式识别"解析出 `project_id` 和 `merge_request_iid` 后，调用 MCP 工具（mcp-gitlab）：

1. `get_merge_request`：获取 MR 标题、描述、源/目标分支、关联 issue（MR 描述中常含缺陷号）、合并状态
2. `list_merge_request_changed_files`：改动文件清单（先看整体规模）
3. `get_merge_request_diffs`：获取 diff 内容；diff 过大时改用 `get_merge_request_file_diff` 按文件逐个取
4. `get_merge_request_notes`：查看评审讨论与提交评论，可能包含根因线索或已指出的问题点
5. 需要完整上下文时，用 `get_file_contents`（`ref` 填 MR 源分支）读取改动文件的完整内容

### 方式 B：本地 git

```bash
git log -3 --oneline          # 先列最近提交让用户可确认
git show <commit> --stat      # 改动文件清单
git show <commit>             # 完整 diff
```

- 用户明确指定了 commit / 分支时以用户指定为准
- 用户说"还没提交"时改用 `git diff HEAD` + `git status`
- 注意在正确的 git 仓库根目录执行；多模块仓库先在对应子目录确认

## 步骤 3：定位根因

不要只看 diff 表面，必须：

1. Read 改动文件的完整方法上下文，理解**旧代码为什么错**（缺判空？条件写反？事务边界？并发？数据兼容？）
2. 追一层调用链：确认触发入口（controller 端点/定时任务/MQ），用于写影响范围
3. 区分"根因"与"表象"：报错堆栈/提示语是表象，代码缺陷逻辑才是根因
4. 用 MR 的提交信息与评审讨论印证根因判断；将缺陷单中的复现步骤与代码执行路径对应起来
5. 信息不足时（如缺复现步骤、缺环境信息、MR 与缺陷单无对应关系），先向用户提问再输出，不要臆造

## 步骤 4：按模板输出

在对话中输出以下格式（中文，供用户审阅后自行复制到 Jira；不要主动回填 Jira）：

```markdown
## TROS-XXXXX 缺陷分析

**1 问题产生原因：**
[技术根因：哪个类#方法、什么缺陷逻辑、什么触发条件下暴露。一段话讲清因果链，不贴大段代码]

**2 影响范围：**
[受影响的功能/接口/入口清单；影响的租户面（全租户/特定配置租户）；影响的版本区间；触发条件（必现/特定数据下出现）]

**3 纠正措施（CA）：**
[本次修复改了什么：文件/方法 + 修复逻辑一句话；如何验证的（编译/自测/测试环境验证）]

**4 预防措施（PA）：**
[防再发动作：补充单测/回归用例、代码评审关注点、增加校验或守卫、完善规范文档等，1-3 条，可落地不空话]
```

## 写作质量要求

- **原因**写到代码级根因，禁止只写"代码考虑不周"之类空话
- **影响范围**基于调用链事实，不夸大不缩小；无法确认的面明确标注"待确认"
- **CA** 只写本次实际做的修复，不掺入未做的事
- **PA** 与根因对应（如根因是缺判空 → PA 是评审关注点/静态检查，而不是无关的流程口号）
- 全文控制在 300 字以内为宜，Jira 填写场景不需要长文

## 示例

**输入 1**：`https://jira.taimei.com/browse/TROS-28651 修复完了，帮我写缺陷分析`（最近 commit 为修复提交）

**输出**：

```markdown
## TROS-28651 缺陷分析

**1 问题产生原因：**
EnterpriseServiceImpl#updateEnterpriseInfo 更新企业配置时未对 xxx 字段判空，
ZM_TEST001 租户该字段历史数据为 null，触发 NPE 导致接口报系统错误。

**2 影响范围：**
仅影响"企业配置信息编辑"功能（/web/enterprise/update）；
所有 xxx 字段为空的租户必现，字段有值的租户不受影响；影响版本 N4.x 起。

**3 纠正措施（CA）：**
在 updateEnterpriseInfo 中对 xxx 增加判空兜底（commit abc1234），
编译通过并在测试环境用 ZM_TEST001 租户复现路径验证修复。

**4 预防措施（PA）：**
1. 为该接口补充字段为空的边界用例；
2. 代码评审时对历史存量数据字段的判空作为固定检查项。
```

**输入 2**：`这个 bug 修好了 https://gitlab.taimei.com/tros/backend/-/merge_requests/1823 帮我写缺陷分析`

**处理要点**：先从 MR 链接解析 `project_id=tros/backend`、`merge_request_iid=1823`，再按步骤 2 方式 A 拉取 MR 信息与 diff；若 MR 描述未关联缺陷号，先询问用户缺陷号，避免凭空编造 `TROS-XXXXX`。
