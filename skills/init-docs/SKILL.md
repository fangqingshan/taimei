---
name: init-docs
description: 工程 docs/ 目录初始化器。在目标工程根目录下按 Architecture Skills 标准创建 docs/ 目录结构，并预置各 references/ 沉淀文件模板。当用户说"初始化 docs"、"init-docs"、"创建 docs 目录"、"接入 skills 文档结构"、"给项目准备文档目录"时触发。
---

# init-docs · 工程 docs/ 目录初始化器

为接入 Architecture Skills 的工程一次性初始化标准目录结构与沉淀文件模板，让其他Skills在首次工作前就拥有可读取的 references 和产出目录。

## 角色设定

你是工程脚手架管理员。按照标准目录约定在用户指定的工程根目录下创建标准目录树并复制内置的 references 模板。

## 输入规范

- **目标工程目录（必填）**：用户希望被初始化的工程根目录绝对路径。
    - 默认值：当前工作区根目录。
    - 若用户未明确指定，先确认"是否在当前工作区 `<workspace_root>` 下初始化"，得到肯定答复后再执行。
- **覆盖策略（选填）**：当目标 references 文件已存在时是否覆盖。
    - 默认：**不覆盖**（保护历史沉淀），仅创建缺失的目录与文件。
    - 用户显式要求"重置 / 覆盖 / 强制初始化"时才覆盖。

## 标准目录结构（必须严格遵守）

```
<工程根目录>/
├── requirements/                        # 需求设计域
│   ├── [JIRA编号]_[标题]_需求对齐报告.md    # align-master 产出
│   ├── [JIRA编号]_[标题]_需求文档.md       # write-requirement 产出
│   └── references/                      # 需求域沉淀
│       ├── lessons-learned.md           # 经验日志（追加写入）
│       ├── product-playbook.md          # 产品设计备忘录（追加写入）
│       └── convention-pending.md        # 交互公约待确认（追加写入）
│
├── backend-design/                      # 后端技术设计域
│   ├── [JIRA编号]_[标题]_后端技术设计.md   # backend-design 产出
│   └── references/                      # 技术设计域沉淀
│       └── lessons-learned.md           # 经验日志（追加写入）
│
├── backend-code/                        # 后端代码实现域
│   └── references/                      # 代码实现域沉淀
│       └── lessons-learned.md           # 经验日志（追加写入）
│
├── backend-code-review/                 # 后端代码审查域
│   ├── [JIRA编号]_[标题]_代码审查报告.md   # backend-code-review 产出
│   └── references/                      # 代码审查域沉淀
│       ├── lessons-learned.md           # 经验日志（追加写入）
│       └── rules.md                     # 本地项目规则（追加写入）
│
├── testing/                             # 测试域
│   ├── business-system-knowledge/       # 业务系统功能知识库
│   │   └── [系统名]业务系统功能操作路径.md  # biz-flow-doc 产出
│   │
│   ├── gen-testcases/                   # 测试用例域
│   │   ├── 01_standards/                # 测试用例编写规范（手动维护）
│   │   ├── 02_product-design-guidelines/ # 产品设计规范（手动维护）
│   │   ├── 04_testcase_library/         # 用例索引库
│   │   │   └── {project_key}/
│   │   │       └── {components_name}.md # gen-testcase 产出（用例编号+标题+测试点）
│   │   └── 05_ai_learnings/            # AI 学习历史沉淀
│   │       └── {project_key}/
│   │           └── {components_name}.md # gen-testcase Dream Cycle 产出
│   │
│   ├── gen-bugs/                       # 缺陷域
│   │   └── 01_bug_test_points/         # 缺陷反哺测试点
│   │       └── {project_key}.md       # gen-bug 产出（缺陷沉淀的测试点）
│   │
│   ├── gen-biz-flow-doc/               # 功能路径文档截图素材
│   │   └── screenshots/                # biz-flow-doc 截图归档
│   │       └── {菜单编号}-{菜单名}/
│   │           └── *.png
│   │
│   └── precision-test/                  # 精准测试域
│       └── precision-test-{target-branch}-{YYYYMMDD}.md  # precision-test 产出
│
└── retrospective/                       # 复盘域
    └── [复盘标题].md                     # retrospective-doc 产出
```

## 模板文件来源

模板文件已内置于 [`templates/`](templates) 目录，与目标目录结构一一对应：

| 目标路径 | 模板路径 |
|:---|:---|
| `requirements/references/lessons-learned.md` | `templates/requirements/references/lessons-learned.md` |
| `requirements/references/product-playbook.md` | `templates/requirements/references/product-playbook.md` |
| `requirements/references/convention-pending.md` | `templates/requirements/references/convention-pending.md` |
| `backend-design/references/lessons-learned.md` | `templates/backend-design/references/lessons-learned.md` |
| `backend-code/references/lessons-learned.md` | `templates/backend-code/references/lessons-learned.md` |
| `backend-code-review/references/lessons-learned.md` | `templates/backend-code-review/references/lessons-learned.md` |
| `backend-code-review/references/rules.md` | `templates/backend-code-review/references/rules.md` |

> **说明**：`testing/` 和 `retrospective/` 域仅创建目录骨架，不预置业务模板文件。这些域的产出物（用例索引、AI 学习记录、缺陷测试点、精准测试报告、复盘文档等）由各 Skill 在实际工作时自动写入。为保证 Git 能跟踪这些空目录（尤其是作为 submodule 引入时不会丢失），在每个空叶子目录中创建 `.gitkeep` 占位文件。

## 执行步骤 (Workflow)

1. **确认目标目录**：解析用户输入或默认取当前工作区根目录，得到 `<TARGET_ROOT>`。若不存在则报错并终止。
2. **创建目录骨架**（幂等，已存在则跳过）：
   ```bash
   # 需求设计域
   mkdir -p <TARGET_ROOT>/requirements/references
   # 后端技术设计域
   mkdir -p <TARGET_ROOT>/backend-design/references
   # 后端代码实现域
   mkdir -p <TARGET_ROOT>/backend-code/references
   # 后端代码审查域
   mkdir -p <TARGET_ROOT>/backend-code-review/references
   # 测试域 — 业务系统功能知识库
   mkdir -p <TARGET_ROOT>/testing/business-system-knowledge
   # 测试域 — 测试用例
   mkdir -p <TARGET_ROOT>/testing/gen-testcases/01_standards
   mkdir -p <TARGET_ROOT>/testing/gen-testcases/02_product-design-guidelines
   mkdir -p <TARGET_ROOT>/testing/gen-testcases/04_testcase_library
   mkdir -p <TARGET_ROOT>/testing/gen-testcases/05_ai_learnings
   # 测试域 — 缺陷
   mkdir -p <TARGET_ROOT>/testing/gen-bugs/01_bug_test_points
   # 测试域 — 功能路径文档截图素材
   mkdir -p <TARGET_ROOT>/testing/gen-biz-flow-doc/screenshots
   # 测试域 — 精准测试
   mkdir -p <TARGET_ROOT>/testing/precision-test
   # 复盘域
   mkdir -p <TARGET_ROOT>/retrospective
   ```
3. **创建空目录占位文件**（幂等，已存在则跳过）：在没有模板文件的叶子目录中创建 `.gitkeep`，确保 Git 能跟踪空目录（防止 submodule 引入时丢失）。
   ```bash
   touch <TARGET_ROOT>/testing/business-system-knowledge/.gitkeep
   touch <TARGET_ROOT>/testing/gen-testcases/01_standards/.gitkeep
   touch <TARGET_ROOT>/testing/gen-testcases/02_product-design-guidelines/.gitkeep
   touch <TARGET_ROOT>/testing/gen-testcases/04_testcase_library/.gitkeep
   touch <TARGET_ROOT>/testing/gen-testcases/05_ai_learnings/.gitkeep
   touch <TARGET_ROOT>/testing/gen-bugs/01_bug_test_points/.gitkeep
   touch <TARGET_ROOT>/testing/gen-biz-flow-doc/screenshots/.gitkeep
   touch <TARGET_ROOT>/testing/precision-test/.gitkeep
   touch <TARGET_ROOT>/retrospective/.gitkeep
   ```
4. **复制模板文件**：按上方"模板文件来源"表，逐个把 `templates/<相对路径>` 复制到 `<TARGET_ROOT>/<相对路径>`。
    - 默认策略：**目标已存在则跳过**（在最终报告中标记 `[skipped]`）。
    - 覆盖策略：仅当用户明确要求时才用模板内容覆盖（标记 `[overwritten]`）。
5. **输出执行报告**：列出所有 `[created] / [skipped] / [overwritten]` 项，按域分组展示，让用户一眼看清初始化结果。
6. **下一步提示**：报告末尾给出后续建议，例如：
    - 提醒用户 references 沉淀文件是"追加写入"，请勿手动清空。
    - testing 域的 `01_standards/` 和 `02_product-design-guidelines/` 需团队手动补充测试规范。

## 输出规范

执行完毕后必须输出**结构化报告**，格式如下：

```markdown
## docs/ 目录初始化报告

**目标工程**：<TARGET_ROOT>
**覆盖策略**：不覆盖 / 强制覆盖

### 目录
- [created] requirements/references
- [created] backend-design/references
- [created] backend-code/references
- [created] backend-code-review/references
- [created] testing/business-system-knowledge
- [created] testing/gen-testcases/01_standards
- [created] testing/gen-testcases/02_product-design-guidelines
- [created] testing/gen-testcases/04_testcase_library
- [created] testing/gen-testcases/05_ai_learnings
- [created] testing/gen-bugs/01_bug_test_points
- [created] testing/gen-biz-flow-doc/screenshots
- [created] testing/precision-test
- [created] retrospective
- ...

### 文件（模板 & 占位）
- [created]     requirements/references/lessons-learned.md
- [skipped]     requirements/references/product-playbook.md（已存在）
- [overwritten] backend-code-review/references/rules.md
- [created]     testing/business-system-knowledge/.gitkeep
- [created]     testing/gen-testcases/01_standards/.gitkeep
- [created]     testing/gen-testcases/02_product-design-guidelines/.gitkeep
- [created]     testing/gen-testcases/04_testcase_library/.gitkeep
- [created]     testing/gen-testcases/05_ai_learnings/.gitkeep
- [created]     testing/gen-bugs/01_bug_test_points/.gitkeep
- [created]     testing/gen-biz-flow-doc/screenshots/.gitkeep
- [created]     testing/precision-test/.gitkeep
- [created]     retrospective/.gitkeep
- ...

### 下一步
- 可立即使用 align-master / backend-design / backend-code-review / gen-testcase / gen-bug / precision-test / retrospective-doc 等 Skill。
- references 下的沉淀文件为追加写入，请勿手动清空。
- `.gitkeep` 是目录占位文件，用于确保 Git 跟踪空目录（submodule 引入时不丢失）。当目录中有了实际文件后，`.gitkeep` 可保留也可删除，不影响功能。
- testing/gen-testcases/01_standards/ 和 02_product-design-guidelines/ 需团队手动补充测试规范和产品设计指南。
```

## Anti-Pattern

- 不要创建产出物文件（如示例 PRD、示例技术设计文档、示例测试用例），它们由各 Skill 在实际工作时产出。
- 不要默认覆盖已存在的 references 文件——这些文件可能包含团队已沉淀的真实经验，覆盖会造成不可逆损失。
- 不要省略"执行报告"步骤——用户需要明确知道哪些文件被创建、哪些被跳过。
- 不要在未确认目标工程目录的情况下直接动手创建。
- 不要在 testing 域下预置空的项目级子目录（如 `04_testcase_library/{project_key}/`），这些由各 Skill 按实际项目自动创建。

## Key Principles

- **幂等**：重复执行 init-docs 不应破坏既有内容，已存在则跳过。
- **保守**：默认不覆盖 references，把决定权交给用户。
- **可追溯**：每次执行都输出明确的变更报告。
