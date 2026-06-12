---
name: wk-crm-prototype-to-spec
description: 根据蓝湖原型或文字描述，填写 CRM 新模块开发需求模板（Markdown + Excel 双格式）。提取页面结构、字段、按钮、业务规则，自动生成可直接作为 /wk-crm-new-module 输入的完整需求文档。Use when filling CRM module requirement templates from prototypes or descriptions.
license: MIT
compatibility: 需要 openpyxl Python 库（Excel 生成）。
metadata:
  author: wukong-team
  version: "1.0"
---

根据原型或描述填写 CRM 新模块开发需求模板，同步生成 Markdown 和 Excel 双格式文件。

**Input**

用户应提供以下任一输入：
1. **蓝湖原型URL**：`https://lanhuapp.com/...` 分享链接（使用 browser-use MCP 工具访问）
2. **原型截图路径**：本地截图文件路径（使用 image-analyzer 技能分析）
3. **文字描述**：模块名称、字段列表、业务规则等口头描述
4. **混合输入**：原型 + 补充说明组合

如果输入不清晰，使用 AskUserQuestion 工具逐项收集必填参数。

**Output**

完成后输出两个文件：
- `doc/{模块目录}/CRM新模块开发需求模板.md` — Markdown 格式需求文档
- `doc/{模块目录}/CRM新模块开发需求模板.xlsx` — Excel 格式需求文档（6 个 Sheet）

这两个文件可直接作为 `/wk-crm-new-module` Skill 的输入。

**Steps**

## Phase 0：原型分析与数据提取

### 蓝湖原型解析流程

如果用户提供蓝湖 URL：

1. 使用 `browser-use` MCP 工具导航到蓝湖页面
2. 等待 iframe 加载完成（`wait_for` + timeout 5000ms）
3. 执行 `take_snapshot` 获取页面 a11y tree
4. 从 a11y tree 中提取结构化信息：
   - **列表页**：Tab 名称、列标题、搜索字段、操作按钮、状态驱动规则
   - **新建页**：表单字段、字段类型、必填标记、默认值、计算规则
   - **详情页**：展示区域、操作按钮
   - **字段说明表**：字段名、输入方式、是否字典、关联逻辑
5. 如果有多个子页面，逐一 `click` 导航并重复 `take_snapshot`

### 截图解析流程

如果用户提供截图路径：
1. 使用 `image-analyzer` 技能分析每张截图
2. 提取可见的字段名、按钮、Tab、表格结构等

### 必须提取的参数

从原型中提取以下信息（缺失项标记为"待确认"）：

| 类别 | 提取项 |
|------|--------|
| 基础信息 | 模块中文名、英文名、是否有审批、表单渲染模式 |
| 列表页 | Tab名称、列字段、搜索字段、操作按钮及状态驱动规则 |
| 表单字段 | 字段英文名、中文显示名、类型(formType)、必填、选项值、列表是否显示 |
| 子表 | 子表名、字段列表、分类方式 |
| 业务规则 | 状态流转、自动填充、计算规则、ERP集成、作废/撤回规则 |
| 权限 | 额外按钮、角色授权、权限维度 |

**门禁**：提取完成，列出待确认项 ✓

## Phase 1：参数确认与冲突检测

1. **确认 type 和菜单基址**

   读取 `wk-crm-skills/wk-crm-new-module/references/module-registry.md`：
   - 检查 `已分配 CrmEnum.type` 表，确认 type 数字未被占用
   - 检查 `已分配菜单基址` 表，确认基址未被占用
   - type 范围：中集定制模块使用 200-299，当前已用到 242，建议从 **250** 起
   - 基址范围：当前已用到 5040，建议从 **5100** 起，每次递增 20

2. **推断复杂度级别**

   | 条件 | 级别 |
   |------|------|
   | 无审批 + 无子表 | L1 |
   | 有审批 + 无子表 | L2 |
   | 有审批 + 有子表/ERP | L3 |
   | 多子表 + 产品配置 | L4 |

3. **推断表单渲染模式**

   根据 `references/form-rendering-patterns.md` 决策指南：
   - 所有字段标准渲染 → **模式 A**
   - 有独立数据源子表/弹窗选择区块 → **模式 B**
   - 某些字段需要自定义渲染组件 → **模式 C**
   - B + C 组合也常见

4. **产出参数卡片**

   汇总为表格让用户确认：

   | 参数 | 推断值 | 确认 |
   |------|--------|------|
   | 模块中文名 | {提取值} | ? |
   | 模块英文名 | {camelCase} | ? |
   | CrmEnum.type | {未占用数字} | ? |
   | 菜单基址 | {未占用数字} | ? |
   | 复杂度 | L{1-4} | ? |
   | 表单渲染模式 | {A/B/C} | ? |

**门禁**：用户确认参数卡片 ✓

## Phase 2：填写 Markdown 模板

基于 `wk-crm-skills/wk-crm-new-module/references/CRM新模块开发需求模板.md` 的结构，生成填写后的文档。

### 七个章节填写规则

**一、模块基础信息**：
- 标题改为 `{模块中文名} - 开发需求文档`
- 第一行说明改为 `> 本文档基于{来源}分析生成，可作为 /wk-crm-new-module Skill 的输入。`
- 带 * 的 8 个字段必须填写
- 参考模块：从已有模块中选择最接近的（详见 `references/reference-modules.md`）

**二、数据库表设计**：
- 主表名格式 `wk_crm_{tableName}`（tableName 来自模块英文名的 snake_case）
- 系统字段 9 个固定不变（主键名 = snake_case 的主键字段名）
- 业务字段：从表单字段中提取需要持久化的字段
- 子表：L3+ 必须填写，字段名用 snake_case

**三、表单字段定义**：
- 从原型中提取所有页面可见字段
- formType 必须使用 `references/module-registry.md` §FieldEnum type 中的标准值
- 中集定制类型（kh/jtContract/jtOrder/goodsRepository/normalInvoice）需要特别注意
- 列表显示：原型列表页中出现的列标记为"是"

**四、业务逻辑**：
- 状态流转：根据原型操作按钮及状态驱动规则表填写
- ERP 集成：L3+ 填写，描述同步触发条件和方向
- 特殊规则：原型中的业务说明文字、注意事项

**五、页面需求**：
- 列表 Tab：通常为标准四个（全部/我负责的/下属负责的/我关注的）
- 列表默认字段：从原型列表页列标题提取
- 操作按钮：标准 CRUD + 原型中的额外按钮

**六、菜单与权限**：
- 标准 6 按钮（save/update/index/read/delete/transfer）
- 额外按钮从基址+7 起（如作废 cancel、撤回 withdraw）
- menu_type：index=2（页面路由），其他=3（按钮）

**七、参考信息**：
- 填写参考模块、文档路径、原型 URL 等元信息

**门禁**：Markdown 文件已生成且结构完整 ✓

## Phase 3：生成 Excel 文件

使用 `scripts/fill_xlsx.py` 脚本或内联 openpyxl 代码生成 Excel。

### Excel 6 个 Sheet 结构

详细列映射见 `references/excel-sheet-mapping.md`。概要：

| Sheet | 列 | 数据来源 |
|-------|------|---------|
| ① 模块基础信息 | 参数名/参数值/填写说明 | MD 第一章 |
| ② 主表字段 | 字段名/类型/必填/默认值/说明 | MD 第二章 2.1 |
| ③ 表单字段 | 字段名/显示名/类型/必填/选项/列表显示/系统自动/说明 | MD 第三章 |
| ④ 子表设计 | 子表名/中文名/字段名/显示名/类型/必填/说明 | MD 第二章 2.2 |
| ⑤ 业务规则 | 规则类型/描述/触发时机/说明 | MD 第四章 |
| ⑥ 页面与权限 | 配置项/值/说明 | MD 第五、六章 |

### Excel 生成方式

**方式 A：使用脚本**（推荐）

复制 `scripts/fill_xlsx.py` 模板脚本到工作目录，修改数据部分后执行：
```bash
python fill_xlsx.py
```

**方式 B：内联生成**

如果数据量不大，可直接在对话中编写 openpyxl 代码生成。

### openpyxl 注意事项

1. **合并单元格处理**：模板中有合并单元格，清空数据前必须先 `unmerge_cells`
2. **MergedCell 只读**：`MergedCell.value` 是只读属性，不能直接赋值
3. **安全清空函数**：
   ```python
   def clear_rows(ws, start_row, max_row=None):
       for row in ws.iter_rows(min_row=start_row, max_row=max_row or ws.max_row):
           for cell in row:
               if not isinstance(cell, openpyxl.cell.cell.MergedCell):
                   cell.value = None
   ```
4. **样式保留**：填写数据行时添加绿色背景标识已填写

**门禁**：Excel 文件已生成且 6 个 Sheet 数据完整 ✓

## Phase 4：交叉验证

对比 Markdown 和 Excel 内容，确认：

- [ ] 模块基础信息 8 项一致
- [ ] 主表业务字段数量和名称一致
- [ ] 表单字段数量和类型一致
- [ ] 子表字段一致（L3+）
- [ ] 业务规则条目一致
- [ ] 页面配置（Tab/字段/按钮）一致

**门禁**：双格式交叉验证通过 ✓

**Guardrails**

- **type 不可冲突**：必须查 module-registry.md 确认未占用
- **formType 必须标准**：只使用 FieldEnum type 对照表中的值
- **子表 snake_case**：数据库字段名一律 snake_case
- **表单字段 camelCase**：页面字段名一律 camelCase
- **系统字段不修改**：9 个系统字段保持不变
- **parent_id=1**：菜单目录挂在 CRM 根菜单下
- **双格式同步**：MD 和 XLSX 必须内容一致
- **不遗漏中集定制类型**：jtContract/kh/jtOrder 等定制类型需正确识别

**Reference Map**

| 文档 | 路径 | 用途 |
|------|------|------|
| 模块注册表 | `wk-crm-skills/wk-crm-new-module/references/module-registry.md` | type/基址/FieldEnum 对照表 |
| 表单渲染模式 | `wk-crm-skills/wk-crm-new-module/references/form-rendering-patterns.md` | 三种模式决策 |
| MD 空白模板 | `wk-crm-skills/wk-crm-new-module/references/CRM新模块开发需求模板.md` | 模板结构参考 |
| XLSX 空白模板 | `wk-crm-skills/wk-crm-new-module/references/CRM新模块开发需求模板.xlsx` | Excel 模板文件 |
| Excel Sheet 映射 | `references/excel-sheet-mapping.md` | 6 个 Sheet 详细列映射 |
| 参考模块选择 | `references/reference-modules.md` | 已有模块对照表 |
| 填充脚本 | `scripts/fill_xlsx.py` | Excel 生成 Python 脚本 |
